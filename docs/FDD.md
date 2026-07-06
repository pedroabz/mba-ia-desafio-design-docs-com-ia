# FDD -- Sistema de Webhooks de Notificação de Pedidos

| Campo           | Valor                                                        |
|-----------------|--------------------------------------------------------------|
| **Autor**       | Larissa (Tech Lead)                                          |
| **Status**      | Em Elaboração                                                |
| **Data**        | 2026-07-05                                                   |
| **Revisores**   | Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)          |
| **RFC base**    | [RFC -- Sistema de Webhooks](RFC.md)                         |

---

## 1. Contexto e motivação técnica

Este documento detalha a implementação do sistema de webhooks outbound para notificação de mudanças de status de pedidos, conforme proposto no [RFC](RFC.md). As decisões arquiteturais (Transactional Outbox, worker separado, HMAC-SHA256, at-least-once, retry com backoff e DLQ) estão registradas nos ADRs referenciados pelo RFC e não serão repetidas aqui.

O FDD especifica contratos de API, fluxos internos detalhados, modelagem de dados, códigos de erro, estratégias de resiliência e pontos de integração com o código existente -- informações suficientes para que um desenvolvedor inicie a implementação sem ambiguidade.

---

## 2. Objetivos técnicos

1. Inserir eventos de webhook na outbox de forma atômica dentro da transação existente de `changeStatus`, sem alterar a semântica das operações atuais (update de status, histórico, estoque).
2. Implementar um worker Node.js independente (`src/worker.ts`) que processe eventos pendentes via polling a cada 2 segundos, realize dispatch HTTP com assinatura HMAC-SHA256 e gerencie retries com backoff.
3. Expor endpoints REST para gestão de configurações de webhook (CRUD), rotação de secret, consulta de histórico de entregas e replay administrativo de eventos na DLQ.
4. Reutilizar integralmente os padrões existentes do projeto: estrutura modular, `AppError`, Pino, middleware de erro centralizado, middleware de autenticação e Zod.
5. Manter latência máxima de entrega abaixo de 10 segundos (pior caso: intervalo de polling de 2 segundos + timeout HTTP de 10 segundos).

---

## 3. Escopo e exclusões

### Dentro do escopo

- Tabelas `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries` e `webhook_dead_letter` no MySQL via Prisma.
- Módulo `src/modules/webhooks/` com controller, service, repository, routes, schemas e processor.
- Entry point `src/worker.ts` com script `npm run worker`.
- Endpoints de CRUD de configuração, rotação de secret, histórico de entregas e replay de DLQ.
- Assinatura HMAC-SHA256, rotação com grace period de 24 horas, validação de HTTPS obrigatório.
- Filtragem de eventos por status na inserção da outbox.
- Snapshot do payload no momento da inserção.

### Fora do escopo

- Notificação por email em caso de falhas recorrentes do webhook (decisão explícita -- Larissa [09:37]).
- Dashboard visual para o cliente (projeto separado de frontend -- Larissa [09:40]).
- Rate limiting de saída por cliente (adiado para observação em produção -- Diego [09:39]).
- Archival/expurgo de eventos entregues na outbox (registrado como ponto futuro -- Diego [09:08]).
- Escalabilidade com múltiplos workers (limitação conhecida, single-worker nesta fase -- Diego [09:13]).

---

## 4. Fluxos detalhados

### 4.1. Criação do evento na outbox (dentro da transação do changeStatus)

O método `changeStatus` em `src/modules/orders/order.service.ts` (linhas 126-179) atualmente executa, dentro de `this.prisma.$transaction()`:

1. `findUnique` do pedido com items (linha 132).
2. Validação da transição de status via `canTransition` (linha 147).
3. Débito ou reposição de estoque, se aplicável (linhas 151-155).
4. `update` do status do pedido (linha 158).
5. `create` no `orderStatusHistory` (linha 159).
6. `findUnique` para retornar o pedido atualizado (linha 169).

**Alteração:** entre os passos 5 e 6, inserir a chamada a uma função `publishWebhookEvent`:

```typescript
async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: { id: string; orderNumber: string; customerId: string; totalCents: number },
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>
```

Comportamento interno de `publishWebhookEvent`:

1. Consultar `webhook_endpoints` para obter todos os endpoints ativos (`active = true`) cujo `customerId` corresponda ao `order.customerId`.
2. Para cada endpoint, verificar se `toStatus` está na lista `eventFilters` do endpoint. Se `eventFilters` for nulo ou vazio, considerar como "todos os status" (sem filtro).
3. Para cada endpoint que passou no filtro, montar o payload JSON (snapshot) e inserir uma linha em `webhook_outbox` com `status = 'PENDING'`.
4. Se nenhum endpoint ativo corresponder ao cliente ou nenhum passar no filtro, não inserir nada na outbox (comportamento silencioso, sem erro).

A função recebe o `tx` (transaction client) para participar da mesma transação ACID. Se a transação fizer rollback, os registros de outbox desaparecem junto.

**Payload serializado na outbox (snapshot):**

```json
{
  "event_id": "uuid-gerado-como-pk-da-outbox",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-05T14:30:00.000Z",
  "data": {
    "order_id": "uuid-do-pedido",
    "order_number": "ORD-000042",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "customer_id": "uuid-do-cliente",
    "total_cents": 15000
  }
}
```

O campo `timestamp` reflete o instante da inserção (snapshot), não o instante do envio. O payload é enxuto por decisão explícita (Diego [09:43]): não inclui items do pedido para evitar inflação do payload. Se o cliente precisar de detalhes, consulta `GET /api/v1/orders/:id`.

### 4.2. Processamento pelo worker (polling, dispatch, marcação)

O worker (`src/worker.ts`) executa o seguinte loop:

```
loop infinito:
  1. SELECT dos N registros mais antigos da webhook_outbox
     WHERE status = 'PENDING'
       AND (next_retry_at IS NULL OR next_retry_at <= NOW())
     ORDER BY created_at ASC
     LIMIT batch_size (sugestão: 10)

  2. Para cada evento no batch:
     a. Marcar status como 'PROCESSING' (evitar reprocessamento por restart).
     b. Buscar o webhook_endpoint correspondente (endpoint_id da outbox).
        - Se o endpoint não existe mais (foi deletado), mover direto para DLQ
          com failure_reason = "Endpoint removed". Sem retry.
     c. Montar headers:
        - Content-Type: application/json
        - X-Event-Id: <event_id (PK da outbox)>
        - X-Webhook-Id: <endpoint_id>
        - X-Timestamp: <timestamp ISO 8601 do instante de envio>
        - X-Signature: HMAC-SHA256(secret, request_body)
     d. Gerar assinatura com a secret ativa do endpoint.
        Se o endpoint possui previous_secret dentro do grace period de 24h,
        a assinatura é feita apenas com a secret atual. O cliente é responsável
        por tentar verificar com ambas durante o período de transição.
     e. Realizar HTTP POST para a url do endpoint com timeout de 10 segundos.
     f. Se resposta 2xx:
        - Marcar outbox como 'DELIVERED'.
        - Inserir registro em webhook_deliveries com success = true,
          http_status, response_time_ms.
     g. Se resposta não-2xx ou timeout/erro de rede:
        - Incrementar attempt_count no outbox.
        - Se attempt_count < 5:
          - Calcular next_retry_at com base no backoff.
          - Marcar status como 'PENDING' (volta para a fila).
          - Inserir registro em webhook_deliveries com success = false,
            http_status (ou null se timeout), response_time_ms, error_message.
        - Se attempt_count >= 5:
          - Mover para DLQ (ver 4.4).

  3. Aguardar 2 segundos (setTimeout).
```

**Recuperação de eventos travados:** Na inicialização, o worker deve resetar eventos com `status = 'PROCESSING'` há mais de 60 segundos de volta para `PENDING`. Isso cobre o cenário em que o worker caiu durante o processamento de um batch.

**Tamanho do batch:** Valor inicial sugerido de 10. Pode ser configurável via variável de ambiente (`WEBHOOK_BATCH_SIZE`).

### 4.3. Retry com backoff

Tabela de intervalos de retry (fixos, conforme decidido na reunião -- Diego [09:17]):

| attempt_count | Intervalo desde a falha | next_retry_at              |
|---------------|-------------------------|----------------------------|
| 1             | 1 minuto                | momento_da_falha + 1 min   |
| 2             | 5 minutos               | momento_da_falha + 5 min   |
| 3             | 30 minutos              | momento_da_falha + 30 min  |
| 4             | 2 horas                 | momento_da_falha + 2h      |
| 5 (última)    | 12 horas                | momento_da_falha + 12h     |

Após a 5a tentativa falhada, o evento é movido para a DLQ. Tempo total entre primeira falha e última tentativa: aproximadamente 15 horas.

Implementação sugerida:

```typescript
const RETRY_INTERVALS_MS = [
  1 * 60 * 1000,       // 1 minuto
  5 * 60 * 1000,       // 5 minutos
  30 * 60 * 1000,      // 30 minutos
  2 * 60 * 60 * 1000,  // 2 horas
  12 * 60 * 60 * 1000, // 12 horas
];

function getNextRetryAt(attemptCount: number): Date | null {
  if (attemptCount >= RETRY_INTERVALS_MS.length) return null; // vai para DLQ
  return new Date(Date.now() + RETRY_INTERVALS_MS[attemptCount]);
}
```

O worker só seleciona registros cujo `next_retry_at` seja menor ou igual a `NOW()`, garantindo que retries agendados não sejam processados antes do tempo.

### 4.4. DLQ (Dead Letter Queue)

Quando `attempt_count` atinge 5 e a última tentativa falha:

1. Inserir registro em `webhook_dead_letter` com: payload original, `failure_reason` (último erro HTTP ou mensagem de timeout/rede), `last_http_status`, `attempts_count = 5`, referência ao `endpoint_id`.
2. Remover o registro da `webhook_outbox` (a outbox principal permanece limpa, conforme Diego [09:18]).
3. Registrar log de nível `warn` via Pino com `event_id`, `endpoint_id`, `customer_id` e `failure_reason`.

**Replay manual:**

O endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` (role ADMIN) executa:

1. Buscar o registro na `webhook_dead_letter` pelo `id`.
2. Reinserir o payload como novo registro na `webhook_outbox` com `status = 'PENDING'`, `attempt_count = 0`, `next_retry_at = NULL`.
3. Remover o registro da `webhook_dead_letter`.
4. Registrar log de nível `info` com `event_id`, `replayed_by` (usuário admin do JWT, incluindo `user.id` e `user.email`) e `endpoint_id`, conforme requisito de auditoria (Sofia [09:36]).

---

## 5. Contratos públicos

Todos os endpoints estão sob o prefixo `/api/v1` (mesmo padrão do router existente em `src/routes/index.ts`). Todos exigem autenticação via JWT (`authenticate` middleware), exceto onde indicado diferentemente.

### 5.1. POST /api/v1/webhooks -- Criar configuração de webhook

**Descrição:** Cria um novo endpoint de webhook para um cliente. A secret é gerada pelo servidor e devolvida na resposta (única vez em que é visível em texto plano, além da rotação).

**Headers do request:**
```
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request body:**
```json
{
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://atlas-comercial.com/webhooks/orders",
  "eventFilters": ["SHIPPED", "DELIVERED"]
}
```

- `customerId` (string, uuid, obrigatório): ID do cliente que receberá as notificações.
- `url` (string, url, obrigatório): URL HTTPS do endpoint do cliente. URLs com esquema HTTP são rejeitadas com erro `WEBHOOK_INVALID_URL`.
- `eventFilters` (array de OrderStatus, opcional): lista de status que disparam o webhook. Se omitido ou vazio, todos os status são notificados.

**Response 201 Created:**
```json
{
  "id": "f8e7d6c5-b4a3-2190-fedc-ba0987654321",
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://atlas-comercial.com/webhooks/orders",
  "eventFilters": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_k7G9mP2xQ4rT8vB1nC6jH3wF5yA0dE",
  "active": true,
  "createdAt": "2026-07-05T10:00:00.000Z",
  "updatedAt": "2026-07-05T10:00:00.000Z"
}
```

**Erros possíveis:**

| Status | Código               | Condição                                         |
|--------|----------------------|--------------------------------------------------|
| 400    | VALIDATION_ERROR     | Campos obrigatórios ausentes ou formato inválido  |
| 400    | WEBHOOK_INVALID_URL  | URL não utiliza esquema HTTPS                    |
| 401    | UNAUTHORIZED         | JWT ausente ou inválido                           |
| 404    | NOT_FOUND            | Customer não encontrado                           |
| 409    | WEBHOOK_DUPLICATE_URL| Já existe webhook ativo para o mesmo customer+URL |

### 5.2. GET /api/v1/webhooks?customerId=... -- Listar webhooks

**Descrição:** Lista todos os endpoints de webhook cadastrados para um cliente.

**Headers do request:**
```
Authorization: Bearer <jwt>
```

**Query parameters:**
- `customerId` (string, uuid, obrigatório): filtra por cliente.
- `page` (number, opcional, default: 1)
- `pageSize` (number, opcional, default: 20, max: 100)

**Response 200 OK:**
```json
{
  "data": [
    {
      "id": "f8e7d6c5-b4a3-2190-fedc-ba0987654321",
      "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "url": "https://atlas-comercial.com/webhooks/orders",
      "eventFilters": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-05T10:00:00.000Z",
      "updatedAt": "2026-07-05T10:00:00.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 1,
    "totalPages": 1
  }
}
```

**Nota:** O campo `secret` não é incluído na listagem.

**Erros possíveis:**

| Status | Código           | Condição                    |
|--------|------------------|-----------------------------|
| 400    | VALIDATION_ERROR | Query parameters inválidos   |
| 401    | UNAUTHORIZED     | JWT ausente ou inválido      |

### 5.3. PATCH /api/v1/webhooks/:id -- Editar webhook

**Descrição:** Atualiza campos de configuração de um webhook existente. Apenas os campos enviados são alterados. A `secret` não pode ser alterada via PATCH -- para rotação, usar o endpoint específico (5.5).

**Headers do request:**
```
Authorization: Bearer <jwt>
Content-Type: application/json
```

**Request body (todos os campos opcionais):**
```json
{
  "url": "https://atlas-comercial.com/webhooks/v2/orders",
  "eventFilters": ["PROCESSING", "SHIPPED", "DELIVERED"],
  "active": false
}
```

**Response 200 OK:**
```json
{
  "id": "f8e7d6c5-b4a3-2190-fedc-ba0987654321",
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://atlas-comercial.com/webhooks/v2/orders",
  "eventFilters": ["PROCESSING", "SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-07-05T10:00:00.000Z",
  "updatedAt": "2026-07-05T15:30:00.000Z"
}
```

**Erros possíveis:**

| Status | Código               | Condição                                  |
|--------|----------------------|-------------------------------------------|
| 400    | VALIDATION_ERROR     | Campos com formato inválido               |
| 400    | WEBHOOK_INVALID_URL  | URL não utiliza esquema HTTPS             |
| 401    | UNAUTHORIZED         | JWT ausente ou inválido                    |
| 404    | WEBHOOK_NOT_FOUND    | Webhook com o ID informado não existe     |

### 5.4. DELETE /api/v1/webhooks/:id -- Remover webhook

**Descrição:** Remove permanentemente uma configuração de webhook. Eventos pendentes na outbox para este endpoint permanecem e serão descartados pelo worker ao detectar que o endpoint não existe mais (movidos para DLQ com `failure_reason = "Endpoint removed"`).

**Headers do request:**
```
Authorization: Bearer <jwt>
```

**Response 204 No Content:** (sem body)

**Erros possíveis:**

| Status | Código            | Condição                              |
|--------|-------------------|---------------------------------------|
| 401    | UNAUTHORIZED      | JWT ausente ou inválido                |
| 404    | WEBHOOK_NOT_FOUND | Webhook com o ID informado não existe |

### 5.5. POST /api/v1/webhooks/:id/rotate-secret -- Rotação de secret

**Descrição:** Gera uma nova secret para o webhook. A secret anterior permanece válida por 24 horas (grace period) para permitir migração sem perda de entregas (Sofia [09:21]).

**Headers do request:**
```
Authorization: Bearer <jwt>
```

**Request body:** Nenhum.

**Response 200 OK:**
```json
{
  "id": "f8e7d6c5-b4a3-2190-fedc-ba0987654321",
  "secret": "whsec_n3M8pR5tQ7vW1xB4kC9jH2yF6zA0dE",
  "previousSecretExpiresAt": "2026-07-06T15:30:00.000Z"
}
```

**Comportamento interno:**
1. Gera nova secret com `crypto.randomBytes(32)`, codificada com prefixo `whsec_`.
2. Move a secret atual para `previousSecret` com `previousSecretExpiresAt = NOW() + 24h`.
3. Retorna a nova secret na resposta.

**Erros possíveis:**

| Status | Código            | Condição                              |
|--------|-------------------|---------------------------------------|
| 401    | UNAUTHORIZED      | JWT ausente ou inválido                |
| 404    | WEBHOOK_NOT_FOUND | Webhook com o ID informado não existe |

### 5.6. GET /api/v1/webhooks/:id/deliveries -- Histórico de entregas

**Descrição:** Retorna o histórico de tentativas de entrega para um webhook específico, ordenado por data decrescente. Inclui tanto sucessos quanto falhas (Marcos [09:34]).

**Headers do request:**
```
Authorization: Bearer <jwt>
```

**Query parameters:**
- `page` (number, opcional, default: 1)
- `pageSize` (number, opcional, default: 20, max: 100)

**Response 200 OK:**
```json
{
  "data": [
    {
      "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
      "eventId": "a9b8c7d6-e5f4-3210-fedc-ba0987654321",
      "eventType": "order.status_changed",
      "httpStatus": 200,
      "success": true,
      "responseTimeMs": 342,
      "errorMessage": null,
      "attemptNumber": 1,
      "createdAt": "2026-07-05T14:30:02.000Z"
    },
    {
      "id": "c2d3e4f5-a6b7-8901-bcde-f12345678901",
      "eventId": "b0c9d8e7-f6a5-4321-edcb-a10987654321",
      "eventType": "order.status_changed",
      "httpStatus": 503,
      "success": false,
      "responseTimeMs": 10000,
      "errorMessage": "Service Unavailable",
      "attemptNumber": 1,
      "createdAt": "2026-07-05T14:28:04.000Z"
    }
  ],
  "pagination": {
    "page": 1,
    "pageSize": 20,
    "total": 2,
    "totalPages": 1
  }
}
```

**Erros possíveis:**

| Status | Código            | Condição                              |
|--------|-------------------|---------------------------------------|
| 401    | UNAUTHORIZED      | JWT ausente ou inválido                |
| 404    | WEBHOOK_NOT_FOUND | Webhook com o ID informado não existe |

### 5.7. POST /api/v1/admin/webhooks/dead-letter/:id/replay -- Replay de DLQ (ADMIN)

**Descrição:** Reinsere um evento da dead letter queue na outbox para nova tentativa de entrega. Requer role ADMIN (Sofia [09:36]).

**Headers do request:**
```
Authorization: Bearer <jwt>
```

**Request body:** Nenhum.

**Response 200 OK:**
```json
{
  "message": "Evento reinserido na fila de processamento",
  "outboxId": "e4f5a6b7-c8d9-0123-abcd-ef4567890123",
  "replayedBy": "admin-user-id",
  "replayedAt": "2026-07-05T16:00:00.000Z"
}
```

**Erros possíveis:**

| Status | Código            | Condição                                 |
|--------|-------------------|------------------------------------------|
| 401    | UNAUTHORIZED      | JWT ausente ou inválido                   |
| 403    | FORBIDDEN         | Usuário autenticado não possui role ADMIN |
| 404    | WEBHOOK_NOT_FOUND | Registro na DLQ não encontrado            |

### 5.8. Headers do webhook HTTP POST (enviados pelo worker ao endpoint do cliente)

| Header           | Valor                                      | Descrição                                                        |
|------------------|--------------------------------------------|------------------------------------------------------------------|
| `Content-Type`   | `application/json`                         | Tipo do corpo da requisição.                                     |
| `X-Event-Id`     | UUID do evento (PK da outbox)              | Identificador único e estável para deduplicação pelo cliente.    |
| `X-Webhook-Id`   | UUID da configuração do endpoint           | Identifica qual cadastro de webhook gerou o envio.               |
| `X-Signature`    | HMAC-SHA256 hex-encoded                    | Assinatura do corpo do request para validação de autenticidade.  |
| `X-Timestamp`    | ISO 8601 do instante de envio              | Timestamp do envio para detecção de replay attack pelo cliente.  |

**Cálculo da assinatura:**

```typescript
import { createHmac } from 'node:crypto';

function signPayload(secret: string, body: string): string {
  return createHmac('sha256', secret).update(body).digest('hex');
}
```

O corpo é assinado como string JSON exatamente como será enviado (sem reformatação). O cliente recalcula o HMAC com a mesma secret e compara.

---

## 6. Modelagem de dados (Prisma)

Novas tabelas a serem adicionadas em `prisma/schema.prisma`. Todas seguem o padrão existente: UUID como PK com `@default(uuid()) @db.Char(36)`, campos de timestamp com `@default(now())` e `@updatedAt` onde aplicável.

### 6.1. webhook_endpoints

```prisma
model WebhookEndpoint {
  id                       String    @id @default(uuid()) @db.Char(36)
  customerId               String    @db.Char(36)
  url                      String    @db.VarChar(2048)
  secret                   String    @db.VarChar(255)
  previousSecret           String?   @db.VarChar(255)
  previousSecretExpiresAt  DateTime?
  eventFilters             Json?
  active                   Boolean   @default(true)
  createdAt                DateTime  @default(now())
  updatedAt                DateTime  @updatedAt

  customer    Customer              @relation(fields: [customerId], references: [id])
  outboxItems WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@unique([customerId, url])
  @@index([customerId, active])
  @@map("webhook_endpoints")
}
```

**Notas:**
- O campo `eventFilters` é armazenado como JSON (array de strings de OrderStatus). Se nulo ou vazio, todos os status são aceitos.
- A constraint `@@unique([customerId, url])` previne duplicação de URL para o mesmo cliente.
- É necessário adicionar a relação reversa no model `Customer` existente: `webhookEndpoints WebhookEndpoint[]`. Essa é uma alteração puramente declarativa que não gera coluna nova na tabela `customers`.

### 6.2. webhook_outbox

```prisma
enum WebhookOutboxStatus {
  PENDING
  PROCESSING
  DELIVERED
  FAILED
}

model WebhookOutbox {
  id            String              @id @db.Char(36)
  endpointId    String              @db.Char(36)
  payload       Json
  status        WebhookOutboxStatus @default(PENDING)
  attemptCount  Int                 @default(0)
  nextRetryAt   DateTime?
  createdAt     DateTime            @default(now())
  updatedAt     DateTime            @updatedAt

  endpoint   WebhookEndpoint   @relation(fields: [endpointId], references: [id], onDelete: Cascade)
  deliveries WebhookDelivery[]

  @@index([status, createdAt])
  @@index([endpointId])
  @@map("webhook_outbox")
}
```

**Notas:**
- O `id` **não** usa `@default(uuid())` -- é gerado no application layer e corresponde ao `event_id` enviado no header `X-Event-Id`.
- O índice `[status, createdAt]` otimiza a query principal do worker: `WHERE status = 'PENDING' ORDER BY created_at ASC`.

### 6.3. webhook_deliveries

```prisma
model WebhookDelivery {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  endpointId     String   @db.Char(36)
  httpStatus     Int?
  success        Boolean
  responseTimeMs Int?
  errorMessage   String?  @db.VarChar(1000)
  attemptNumber  Int
  createdAt      DateTime @default(now())

  outbox   WebhookOutbox   @relation(fields: [outboxId], references: [id], onDelete: Cascade)
  endpoint WebhookEndpoint @relation(fields: [endpointId], references: [id])

  @@index([outboxId])
  @@index([endpointId, createdAt])
  @@map("webhook_deliveries")
}
```

### 6.4. webhook_dead_letter

```prisma
model WebhookDeadLetter {
  id             String   @id @default(uuid()) @db.Char(36)
  outboxId       String   @db.Char(36)
  endpointId     String   @db.Char(36)
  payload        Json
  failureReason  String   @db.VarChar(1000)
  lastHttpStatus Int?
  attemptsCount  Int
  createdAt      DateTime @default(now())

  endpoint WebhookEndpoint @relation(fields: [endpointId], references: [id])

  @@index([endpointId])
  @@index([createdAt])
  @@map("webhook_dead_letter")
}
```

---

## 7. Matriz de erros previstos

Todos os erros de domínio do módulo de webhooks estendem `AppError` (de `src/shared/errors/app-error.ts`) e utilizam o prefixo `WEBHOOK_` (Bruno [09:28]). O middleware centralizado em `src/middlewares/error.middleware.ts` captura automaticamente qualquer instância de `AppError` sem necessidade de alteração (linha 15: `if (err instanceof AppError)`).

| Código de erro               | HTTP Status | Mensagem                                                     | Contexto de uso                                              |
|------------------------------|-------------|--------------------------------------------------------------|--------------------------------------------------------------|
| `WEBHOOK_NOT_FOUND`          | 404         | Webhook not found                                            | GET, PATCH, DELETE, rotate-secret, deliveries quando o ID não existe |
| `WEBHOOK_INVALID_URL`        | 400         | Webhook URL must use HTTPS scheme                            | POST e PATCH quando a URL não começa com `https://`           |
| `WEBHOOK_SECRET_REQUIRED`    | 400         | Webhook secret is required for signature verification        | Erro interno se, por inconsistência, o endpoint não possuir secret |
| `WEBHOOK_DELIVERY_FAILED`    | N/A         | Webhook delivery failed: {reason}                            | Log do worker ao registrar falha de entrega (não exposto via API) |
| `WEBHOOK_PAYLOAD_TOO_LARGE`  | 422         | Webhook payload exceeds maximum size of 64KB                 | Inserção na outbox quando o payload serializado excede 64KB   |
| `WEBHOOK_ENDPOINT_INACTIVE`  | 422         | Webhook endpoint is inactive                                 | Tentativa de operação em endpoint desativado, se aplicável    |
| `WEBHOOK_DLQ_NOT_FOUND`      | 404         | Dead letter entry not found                                  | Replay quando o ID da DLQ não existe                         |
| `WEBHOOK_DUPLICATE_URL`      | 409         | Webhook URL already registered for this customer             | POST quando o cliente já possui um endpoint com a mesma URL   |

**Implementação sugerida** (em `src/modules/webhooks/webhook.errors.ts`):

```typescript
import { AppError } from '../../shared/errors/app-error.js';

export class WebhookNotFoundError extends AppError {
  constructor() {
    super('Webhook not found', 404, 'WEBHOOK_NOT_FOUND');
  }
}

export class WebhookInvalidUrlError extends AppError {
  constructor() {
    super('Webhook URL must use HTTPS scheme', 400, 'WEBHOOK_INVALID_URL');
  }
}

export class WebhookPayloadTooLargeError extends AppError {
  constructor(sizeBytes: number) {
    super('Webhook payload exceeds maximum size of 64KB', 422, 'WEBHOOK_PAYLOAD_TOO_LARGE', {
      sizeBytes,
      limitBytes: 65_536,
    });
  }
}

export class WebhookDuplicateUrlError extends AppError {
  constructor() {
    super('Webhook URL already registered for this customer', 409, 'WEBHOOK_DUPLICATE_URL');
  }
}

export class WebhookDlqNotFoundError extends AppError {
  constructor() {
    super('Dead letter entry not found', 404, 'WEBHOOK_DLQ_NOT_FOUND');
  }
}
```

**Nota sobre a hierarquia de erros:** A abordagem acima estende `AppError` diretamente (em vez de estender `NotFoundError`, `BadRequestError`, etc.) para garantir que o `errorCode` tenha o prefixo `WEBHOOK_`. A classe `NotFoundError` existente utiliza `errorCode` fixo `'NOT_FOUND'` sem aceitar parâmetro para customização. Estender `AppError` diretamente evita essa restrição sem exigir alteração nas classes base.

---

## 8. Estratégias de resiliência

### 8.1. Timeout de dispatch HTTP

Cada chamada HTTP do worker para o endpoint do cliente utiliza timeout de **10 segundos** (Diego [09:42]). Caso o tempo expire sem resposta, o evento é tratado como falha e entra na lógica de retry.

Utilizar o `fetch` nativo do Node.js 20+ com `AbortController`:

```typescript
const controller = new AbortController();
const timeout = setTimeout(() => controller.abort(), 10_000);

try {
  const response = await fetch(url, {
    method: 'POST',
    headers,
    body: payloadString,
    signal: controller.signal,
  });
  // processar resposta
} finally {
  clearTimeout(timeout);
}
```

### 8.2. Retry com backoff

Conforme detalhado na seção 4.3. Intervalos fixos: 1m, 5m, 30m, 2h, 12h. Total de 5 tentativas cobrindo janela de aproximadamente 15 horas.

### 8.3. Isolamento de falhas por endpoint

Falha de um endpoint de um cliente não bloqueia entregas para outros clientes. O worker processa eventos em batch e cada evento é tratado individualmente. Se o endpoint A falha, o evento de A vai para retry enquanto o evento de B (próximo no batch) é processado normalmente.

### 8.4. Proteção contra endpoint removido

Quando o worker tenta processar um evento cujo `endpoint_id` não existe mais em `webhook_endpoints` (por ter sido deletado), o evento deve ser movido diretamente para a DLQ com `failure_reason = "Endpoint removed"`, sem tentativas de retry.

### 8.5. Limite de payload

Payload máximo de 64KB (Diego [09:24]). Validação no momento da inserção na outbox. Na prática, o payload do evento `order.status_changed` é muito menor (< 1KB), mas o limite protege contra cenários anômalos.

### 8.6. Recuperação de eventos travados

Na inicialização do worker, eventos com `status = 'PROCESSING'` há mais de 60 segundos são resetados para `PENDING`. Isso cobre o cenário em que o worker caiu durante o processamento.

### 8.7. Critérios de falha

Uma tentativa de entrega é considerada falha quando:
- O endpoint retorna status HTTP fora da faixa 2xx (200-299).
- A requisição excede o timeout de 10 segundos (`AbortError`).
- Ocorre erro de rede (DNS, conexão recusada, TLS inválido).

---

## 9. Observabilidade

### 9.1. Logger

Utilizar o Pino existente em `src/shared/logger/index.ts`. O worker instancia um child logger para contexto:

```typescript
import { logger } from './shared/logger/index.js';
const workerLogger = logger.child({ component: 'webhook-worker' });
```

**Alteração necessária em `src/shared/logger/index.ts`:** Adicionar `'*.secret'` ao array `redactPaths` (atualmente na linha 4) para garantir que secrets de webhook não vazem nos logs. Os paths existentes (`*.password`, `*.passwordHash`, `*.token`, `*.accessToken`) não cobrem o campo `secret`.

### 9.2. Logs estruturados

Eventos de log obrigatórios:

| Nível   | Evento                       | Campos                                                                    |
|---------|------------------------------|---------------------------------------------------------------------------|
| `info`  | `webhook_worker_started`     | `pollingIntervalMs`, `batchSize`                                         |
| `info`  | `webhook_worker_shutdown`    | `signal`                                                                 |
| `info`  | `webhook_dispatched`         | `eventId`, `endpointId`, `customerId`, `httpStatus`, `responseTimeMs`    |
| `warn`  | `webhook_delivery_failed`    | `eventId`, `endpointId`, `customerId`, `attemptCount`, `error`, `nextRetryAt` |
| `warn`  | `webhook_moved_to_dlq`       | `eventId`, `endpointId`, `customerId`, `failureReason`, `totalAttempts`  |
| `info`  | `webhook_dlq_replayed`       | `eventId`, `endpointId`, `replayedBy`                                    |
| `info`  | `webhook_secret_rotated`     | `endpointId`, `previousExpiresAt`                                        |
| `error` | `webhook_worker_error`       | `err` (erros inesperados do loop do worker)                              |
| `debug` | `webhook_batch_processed`    | `batchSize`, `processedCount`, `failedCount`, `durationMs`               |

### 9.3. Métricas sugeridas

A transcrição não definiu requisitos de métricas ou sistema de monitoramento. O projeto atual não possui infraestrutura de métricas (Prometheus, Datadog, etc.). A implementação inicial se apoia exclusivamente em logs estruturados. Métricas podem ser extraídas dos logs pelo sistema de observabilidade que a equipe utilizar.

Pontos de instrumentação recomendados para futura extração:

- Total de eventos despachados com sucesso vs. falha (extraível dos logs `webhook_dispatched` e `webhook_delivery_failed`).
- Latência de entrega (diferença entre `createdAt` da outbox e timestamp do envio bem-sucedido).
- Quantidade de eventos pendentes na outbox (extraível via query periódica).
- Quantidade de eventos na DLQ (extraível dos logs `webhook_moved_to_dlq`).

### 9.4. Tracing

O projeto atual não possui instrumentação de distributed tracing. Não foi discutido na reunião. O `eventId` (UUID único por evento, PK da outbox) serve como correlação entre o log de inserção na outbox (API), os logs de tentativa de envio (worker) e os registros na tabela de deliveries.

---

## 10. Dependências e compatibilidade

### 10.1. Dependências existentes reutilizadas

| Dependência       | Versão atual | Uso no módulo de webhooks                                      |
|-------------------|-------------|----------------------------------------------------------------|
| `@prisma/client`  | 5.22.0      | Acesso ao banco, transações, queries da outbox e configurações |
| `express`         | 4.21.1      | Router, middlewares, endpoints REST                            |
| `zod`             | 3.23.8      | Validação de schemas de entrada (body, query, params)          |
| `pino`            | 9.5.0       | Logging estruturado no worker e nos endpoints                  |
| `jsonwebtoken`    | 9.0.2       | Autenticação JWT (reutilizado via middleware existente)         |
| `uuid`            | 11.0.3      | Geração de UUIDs (se necessário fora do Prisma)                |

### 10.2. Dependências novas

Nenhuma dependência nova é necessária. Os dois recursos adicionais utilizam módulos nativos do Node.js:

- **HMAC-SHA256:** Módulo nativo `node:crypto` (`createHmac`).
- **Geração de secret:** Módulo nativo `node:crypto` (`randomBytes`).
- **HTTP dispatch:** `fetch` nativo do Node.js 20+ (o projeto já exige `"node": ">=20"` em `package.json`).

### 10.3. Compatibilidade

- **Node.js >= 20:** Obrigatório (já especificado em `package.json`). Necessário para `fetch` nativo.
- **MySQL:** Nenhuma feature específica de versão. Tabelas novas utilizam tipos e índices padrão.
- **Prisma 5.22.0:** Suporte completo a `Json` type, enums e relações utilizadas na modelagem.
- **Endpoints existentes:** Nenhum endpoint existente é alterado. Todos continuam funcionando sem mudança.
- **Única alteração em código existente que afeta fluxo crítico:** Inserção da chamada a `publishWebhookEvent` dentro do método `changeStatus`. Se nenhum webhook estiver configurado, a função executa apenas um `SELECT` com índice (impacto < 5ms).

---

## 11. Critérios de aceite técnicos

### Consistência transacional

- [ ] A inserção na `webhook_outbox` ocorre dentro da mesma `prisma.$transaction` que atualiza o status da order e insere em `order_status_history`.
- [ ] Se a transação sofre rollback (ex.: estoque insuficiente), nenhum registro existe na outbox.
- [ ] O payload armazenado na outbox é um snapshot do estado da order no momento da transição (não é renderizado sob demanda no envio).

### Worker

- [ ] O worker roda como processo Node.js separado, iniciado via `npm run worker`.
- [ ] O worker utiliza instância própria de `PrismaClient` (não compartilha com a API).
- [ ] O polling ocorre a cada 2 segundos.
- [ ] Graceful shutdown via `SIGINT`/`SIGTERM` aguarda batch atual antes de encerrar.
- [ ] Na inicialização, eventos em `PROCESSING` há mais de 60 segundos são recuperados para `PENDING`.

### Entrega HTTP

- [ ] O worker envia HTTP POST ao endpoint cadastrado com headers `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp` e `Content-Type: application/json`.
- [ ] A assinatura é calculada com `HMAC-SHA256(secret, raw_body)` usando a secret ativa do endpoint.
- [ ] Timeout de 10 segundos por chamada.
- [ ] Respostas 2xx são tratadas como sucesso; demais como falha.
- [ ] O `X-Event-Id` enviado em todas as tentativas de um mesmo evento é idêntico (UUID da outbox).

### Retry e DLQ

- [ ] Após falha, o evento é agendado para retry com backoff: 1m, 5m, 30m, 2h, 12h.
- [ ] Após 5 tentativas falhadas, o evento é movido para `webhook_dead_letter` e removido da outbox.
- [ ] Endpoint de replay reinsere na outbox com `attempt_count = 0` e remove da DLQ.
- [ ] O endpoint de replay exige role `ADMIN` e registra em log quem executou a ação (auditoria).

### Segurança

- [ ] URLs HTTP são rejeitadas no cadastro com erro `WEBHOOK_INVALID_URL`.
- [ ] Secrets são geradas pelo sistema com `crypto.randomBytes(32)`.
- [ ] Rotação de secret preserva a secret anterior por 24 horas em `previousSecret`.
- [ ] A secret só é retornada na criação (POST) e na rotação (rotate-secret). Listagens e edições não retornam secret.

### Filtragem de eventos

- [ ] Se um webhook está configurado com `eventFilters: ["SHIPPED", "DELIVERED"]`, mudanças para `PROCESSING` não geram registro na outbox.
- [ ] Se `eventFilters` for nulo ou vazio, todos os status geram evento.

### Configuração e consulta

- [ ] CRUD completo funciona: criar (POST 201), listar (GET 200), atualizar (PATCH 200), remover (DELETE 204).
- [ ] Histórico de entregas retorna tentativas com `httpStatus`, `responseTimeMs`, `errorMessage`, `attemptNumber` e `success`.
- [ ] Paginação segue o padrão existente (função `paginated()` de `src/shared/http/response.ts`).

### Aderência a padrões

- [ ] Módulo organizado em `src/modules/webhooks/` com controller, service, repository, routes, schemas e errors.
- [ ] Schemas de validação escritos com Zod.
- [ ] Logs escritos com Pino.
- [ ] Rotas registradas no router central (`src/routes/index.ts`).
- [ ] Todos os erros utilizam `AppError` com códigos prefixados `WEBHOOK_`.
- [ ] O `errorMiddleware` existente trata os erros sem alteração.

---

## 12. Riscos e mitigação

| Risco | Probabilidade | Severidade | Mitigação |
|-------|---------------|------------|-----------|
| Transação do `changeStatus` fica mais lenta com queries adicionais de lookup de endpoints e inserção na outbox | Média | Baixa | As queries adicionais são um `SELECT` com índice em `[customerId, active]` e `INSERT` simples. Impacto estimado < 5ms. Monitorar via logs de duração. |
| Worker cai e eventos acumulam sem processamento | Média | Alta | Supervisão de processo obrigatória (pm2, systemd ou container com restart policy). Eventos não são perdidos -- permanecem no banco e são processados na reinicialização. |
| Eventos ficam travados em `PROCESSING` após crash do worker | Média | Média | Mecanismo de recuperação na inicialização: resetar eventos `PROCESSING` há mais de 60 segundos para `PENDING`. |
| Endpoint de cliente consistentemente lento (próximo de 10s) degrada throughput do single-worker | Baixa | Média | Timeout duro de 10s limita o impacto por evento. Cada evento lento ocupa no máximo 10s. Escalar para múltiplos workers é ponto futuro. |
| Vazamento de secret de webhook | Baixa | Alta | Secret única por endpoint (isolamento de impacto). Rotação com grace period de 24h. HTTPS obrigatório. Secret não retornada em listagens. |
| Outbox cresce indefinidamente por falta de expurgo | Baixa | Média | Registros `DELIVERED` permanecem na tabela, mas o índice `[status, createdAt]` garante que a query do worker (que lê apenas `PENDING`) não degrada. Archival é ponto futuro. |
| Grace period de rotação de secret não é respeitado por falha no cálculo | Baixa | Média | Testes específicos para validar que `previousSecretExpiresAt` é calculado como `NOW() + 24h` e que o worker verifica esse campo antes de usar a secret anterior. |

**Ponto em aberto:** Criptografia at-rest das secrets de webhook no banco de dados. A transcrição não menciona essa questão. Recomenda-se discutir com Sofia (Engenheira de Segurança) na revisão de segurança pré-deploy, que está planejada para o fim da terceira sprint.

---

## 13. Integração com o sistema existente

### 13.1. `src/modules/orders/order.service.ts` -- Inserção na outbox

Ponto de integração mais crítico. O método `changeStatus` (linhas 126-179) será estendido para chamar `publishWebhookEvent(tx, order, from, to)` dentro do bloco `this.prisma.$transaction()`, entre a inserção no `orderStatusHistory` (linha 159) e o `findUnique` final (linha 169).

A função `publishWebhookEvent` será definida em `src/modules/webhooks/webhook.outbox.ts` e importada pelo `OrderService`. Ela recebe o `tx: Prisma.TransactionClient` para participar da mesma transação. O `OrderService` não precisa conhecer detalhes de webhook (endpoints, secrets, filtros) -- a função encapsula toda a lógica de lookup e inserção.

O objeto `order` já disponível na transação (carregado na linha 132 com `include: { items: true }`) fornece `id`, `orderNumber`, `customerId` e `totalCents`, evitando query adicional para montar o snapshot.

### 13.2. `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` -- Reuso da hierarquia de erros

Novos erros de domínio de webhook estendem `AppError` diretamente, utilizando a assinatura do construtor `(message, statusCode, errorCode, details?)` sem necessidade de alterar a classe base. Os erros serão definidos em `src/modules/webhooks/webhook.errors.ts`.

A decisão de estender `AppError` diretamente (em vez de `NotFoundError`, `BadRequestError`) é necessária porque as classes especializadas existentes utilizam `errorCode` fixo (ex.: `NotFoundError` sempre gera `'NOT_FOUND'`), enquanto os erros de webhook precisam de códigos com prefixo `WEBHOOK_`.

### 13.3. `src/middlewares/error.middleware.ts` -- Captura automática

O middleware de erro já trata qualquer instância de `AppError` (linha 15: `if (err instanceof AppError)`), extraindo `statusCode`, `errorCode` e `details` para montar a resposta JSON padronizada. Novos erros com prefixo `WEBHOOK_` serão capturados automaticamente. **Nenhuma alteração é necessária neste arquivo.**

### 13.4. `src/middlewares/auth.middleware.ts` -- Autenticação e autorização

O endpoint de replay de DLQ utiliza `authenticate` seguido de `requireRole('ADMIN')` (função definida na linha 49, que aceita roles variádicas). Os demais endpoints de CRUD utilizam apenas `authenticate`. **Nenhuma alteração é necessária neste arquivo.**

Exemplo de uso no router de DLQ:
```typescript
router.post(
  '/dead-letter/:id/replay',
  authenticate,
  requireRole('ADMIN'),
  controller.replay,
);
```

### 13.5. `src/shared/logger/index.ts` -- Pino existente

O Pino configurado é reutilizado integralmente. O worker cria um child logger com `{ component: 'webhook-worker' }` para contexto.

**Alteração necessária:** Adicionar `'*.secret'` ao array `redactPaths` (linha 4) para garantir que secrets de webhook não vazem nos logs. Os paths atuais (`*.password`, `*.passwordHash`, `*.token`, `*.accessToken`) não cobrem `secret`.

### 13.6. `src/server.ts` -- Referência para `src/worker.ts`

O novo entry point `src/worker.ts` segue a mesma estrutura de bootstrap de `src/server.ts`:

- Importa `createPrismaClient` de `src/config/database.ts` (instância própria do PrismaClient, por ser outro processo -- Bruno [09:30]).
- Importa `logger` de `src/shared/logger/index.ts`.
- Importa `env` de `src/config/env.ts`.
- Implementa graceful shutdown com `SIGINT`/`SIGTERM`, desconectando o PrismaClient.
- **Não** levanta servidor HTTP (não utiliza Express).

Scripts a serem adicionados em `package.json`:
```json
{
  "worker": "node --env-file=.env dist/worker.js",
  "worker:dev": "tsx watch --env-file=.env src/worker.ts"
}
```

### 13.7. `prisma/schema.prisma` -- Novas tabelas

Quatro novas tabelas conforme detalhado na seção 6: `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`. Um novo enum `WebhookOutboxStatus`.

Alteração no model `Customer` existente (linha 40): adicionar relação reversa `webhookEndpoints WebhookEndpoint[]`. Alteração puramente declarativa que não gera coluna nova na tabela `customers`.

Nenhuma alteração em tabelas ou enums existentes além dessa relação.

Nova migration necessária: `npx prisma migrate dev --name add_webhook_tables`.

### 13.8. `src/routes/index.ts` e `src/app.ts` -- Registro do módulo

O tipo `Controllers` (linha 13 de `src/routes/index.ts`) deve ser estendido para incluir `webhooks: WebhookController`. O `buildApiRouter` deve registrar as rotas:

```typescript
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks));
```

A função `buildControllers` em `src/app.ts` (linhas 26-52) deve instanciar `WebhookRepository`, `WebhookService` e `WebhookController`, seguindo o padrão idêntico dos demais módulos.

---

## 14. Estrutura de arquivos do módulo

```
src/
  modules/
    webhooks/
      webhook.controller.ts      -- handlers dos endpoints REST (CRUD, rotate-secret, deliveries)
      webhook.service.ts         -- lógica de negócio (CRUD, rotação, histórico, replay)
      webhook.repository.ts      -- queries Prisma (endpoints, deliveries, DLQ)
      webhook.routes.ts          -- rotas Express (CRUD + rotate-secret + deliveries)
      webhook.admin.routes.ts    -- rotas Express admin (DLQ replay)
      webhook.schemas.ts         -- schemas Zod para validação de entrada
      webhook.errors.ts          -- classes de erro com prefixo WEBHOOK_
      webhook.outbox.ts          -- função publishWebhookEvent (usada pelo OrderService)
      webhook.processor.ts       -- lógica de dispatch, HMAC, retry (usada pelo worker)
      webhook.crypto.ts          -- funções de geração de secret e assinatura HMAC
  worker.ts                      -- entry point do worker (polling loop, graceful shutdown)
```

---

## 15. Pontos em aberto

As seguintes informações não foram definidas na transcrição da reunião nem podem ser inferidas do repositório. Ficam registradas como pontos a decidir antes ou durante a implementação:

1. **Tamanho do batch do worker:** Sugerido 10 neste documento, mas não foi discutido na reunião. Deve ser validado com a tech lead.
2. **Formato exato da secret:** Este documento sugere `whsec_` + base64 de 32 bytes aleatórios. O formato não foi discutido na reunião.
3. **Comportamento quando webhook está `active = false`:** Este documento assume que webhooks inativos não geram eventos na outbox. Não foi explicitamente discutido.
4. **Política de retenção da tabela `webhook_deliveries`:** O histórico de entregas pode crescer indefinidamente. Nenhuma política de cleanup foi definida.
5. **Limite de webhooks por customer:** Não foi discutido. Pode-se implementar sem limite inicialmente e adicionar restrição se necessário.
6. **Criptografia at-rest das secrets no banco:** Não foi mencionado na transcrição. Deve ser discutido na revisão de segurança com Sofia.
7. **Variáveis de ambiente adicionais para o worker:** `WEBHOOK_BATCH_SIZE`, `WEBHOOK_POLLING_INTERVAL_MS`, `WEBHOOK_TIMEOUT_MS` -- não foram discutidas, mas são recomendadas como configurações para facilitar ajustes operacionais sem deploy.
