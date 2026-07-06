# FDD: Sistema de Webhooks para Notificação de Mudança de Status de Pedidos

## Metadados

| Campo            | Valor                                                  |
|------------------|--------------------------------------------------------|
| Feature          | Webhooks outbound para notificação de status de pedidos |
| Autor            | Larissa (Tech Lead)                                    |
| Status           | Em Elaboração                                          |
| Data             | 2026-07-05                                             |
| RFC relacionado  | [RFC: Sistema de Webhooks](RFC.md)                     |
| ADRs relacionadas| ADR-001 a ADR-006                                      |

---

## 1. Contexto e motivação técnica

Este documento detalha a implementação do sistema de webhooks outbound descrito no [RFC](RFC.md). O RFC estabelece a arquitetura em quatro pilares: Transactional Outbox no MySQL, worker de despacho em processo separado, autenticação HMAC-SHA256 com secret por endpoint e garantia at-least-once com retry e DLQ. As justificativas arquiteturais e alternativas descartadas estão documentadas no RFC e nas ADRs (001 a 006); este FDD não as repete.

O objetivo deste documento é fornecer especificação técnica suficiente para que um desenvolvedor inicie a implementação sem ambiguidades: contratos de API, fluxos internos, modelos de dados, estratégias de resiliência, integração com o código existente e critérios de aceite.

---

## 2. Objetivos técnicos

1. Inserir eventos de webhook atomicamente na mesma transação SQL que já executa a mudança de status do pedido (`changeStatus` em `src/modules/orders/order.service.ts`).
2. Implementar um worker Node.js em processo separado (`src/worker.ts`) que faça polling a cada 2 segundos na tabela `webhook_outbox` e despache chamadas HTTP para os endpoints dos clientes.
3. Expor endpoints REST para CRUD de configuração de webhooks, rotação de secret, consulta de histórico de entregas e replay de DLQ.
4. Assinar cada requisição de webhook com HMAC-SHA256 usando a secret do endpoint, com suporte a rotação com grace period de 24 horas.
5. Implementar retry com backoff exponencial (5 tentativas: 1m, 5m, 30m, 2h, 12h) e DLQ em tabela separada.
6. Manter consistência com os padrões existentes do projeto: módulo em `src/modules/webhooks/`, classes de erro com prefixo `WEBHOOK_`, logger Pino, validação Zod, middlewares de autenticação e autorização.

---

## 3. Escopo e exclusões

### Incluso nesta fase

- Tabelas `webhook_config`, `webhook_outbox`, `webhook_delivery_log` e `webhook_dead_letter` no Prisma schema.
- Módulo `src/modules/webhooks/` com controller, service, repository, routes e schemas.
- Worker de despacho em `src/worker.ts` com lógica de processamento em `src/modules/webhooks/webhook.processor.ts`.
- Integração com `OrderService.changeStatus` para inserir eventos na outbox dentro da transação existente.
- Endpoints CRUD de configuração, rotação de secret, consulta de entregas e replay de DLQ.
- Assinatura HMAC-SHA256, validação de HTTPS, filtragem de eventos por status.

### Exclusões explícitas

- **Notificação por email** em caso de falhas recorrentes (decisão [09:37] Larissa).
- **Rate limiting de envio** para clientes (decisão [09:39] — observar em produção antes de implementar).
- **Dashboard visual** para clientes (decisão [09:40] — projeto separado do time de frontend).
- **Estratégia de limpeza/arquivamento** da tabela outbox (mencionado como 30 dias, mas fora de escopo — [09:08] Diego).
- **Escalabilidade para múltiplos workers** (limitação conhecida — [09:13]).

---

## 4. Fluxos detalhados

### 4.1. Fluxo de inserção na outbox (síncrono, dentro da transação)

```
OrderController.changeStatus
  -> OrderService.changeStatus(id, input, userId)
       -> this.prisma.$transaction(async (tx) => {
            // 1. Busca order com items (existente)
            // 2. Valida transicao de status (existente)
            // 3. Debita/repoe estoque (existente)
            // 4. Atualiza status da order (existente)
            // 5. Insere no order_status_history (existente)
            // 6. **NOVO** — Chama publishWebhookEvent(tx, order, from, to)
            //    6a. Busca webhook_configs ativos para o customer_id
            //        que incluam `to` na lista de eventos filtrados
            //    6b. Para cada config encontrada, insere uma linha
            //        na webhook_outbox com payload renderizado (snapshot)
            //    6c. Se nenhum config encontrado, nao insere nada
            // 7. Retorna order atualizada (existente)
          })
```

A função `publishWebhookEvent` recebe o `tx` (Prisma `TransactionClient`) da transação corrente, garantindo atomicidade. Se a inserção na outbox falhar, toda a transação sofre rollback, incluindo a mudança de status.

### 4.2. Fluxo do worker (polling e despacho)

```
src/worker.ts — bootstrap
  -> Cria PrismaClient proprio (mesmo DATABASE_URL, instancia separada)
  -> Inicia loop de polling a cada 2 segundos

Ciclo de polling:
  1. SELECT eventos da webhook_outbox WHERE status = 'PENDING'
     AND (next_retry_at IS NULL OR next_retry_at <= NOW())
     ORDER BY created_at ASC LIMIT <batch_size>
  2. Para cada evento no batch:
     a. Marca status como 'PROCESSING' (lock otimista)
     b. Busca webhook_config correspondente (para obter URL e secret)
     c. Renderiza headers: X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type
     d. Calcula HMAC-SHA256 do body com a secret do endpoint
        - Se ha secret em rotacao (previous_secret com expiracao > now),
          assina apenas com a secret atual; cliente pode validar com qualquer uma
     e. Dispara HTTP POST para a URL do webhook com timeout de 10 segundos
     f. Registra resultado na webhook_delivery_log (status code, response body truncado, duracao)
     g. Se sucesso (2xx):
        - Marca evento como 'DELIVERED' na outbox
     h. Se falha (non-2xx, timeout, erro de rede):
        - Incrementa attempt_count
        - Se attempt_count < 5:
          - Calcula next_retry_at com base no backoff
          - Marca status como 'PENDING' com next_retry_at atualizado
        - Se attempt_count >= 5:
          - Move evento para webhook_dead_letter
          - Remove da outbox
  3. Aguarda 2 segundos (setTimeout) e volta ao passo 1
```

### 4.3. Fluxo de retry e backoff

| Tentativa | Intervalo após falha | `next_retry_at` calculado como         |
|-----------|---------------------|-----------------------------------------|
| 1a        | 1 minuto            | `NOW() + INTERVAL 1 MINUTE`            |
| 2a        | 5 minutos           | `NOW() + INTERVAL 5 MINUTE`            |
| 3a        | 30 minutos          | `NOW() + INTERVAL 30 MINUTE`           |
| 4a        | 2 horas             | `NOW() + INTERVAL 2 HOUR`              |
| 5a        | 12 horas            | `NOW() + INTERVAL 12 HOUR`             |

Os intervalos são valores fixos (não calculados exponencialmente em runtime). Podem ser definidos como constante:

```typescript
const RETRY_INTERVALS_MS = [
  1 * 60 * 1000,       // 1 minuto
  5 * 60 * 1000,       // 5 minutos
  30 * 60 * 1000,      // 30 minutos
  2 * 60 * 60 * 1000,  // 2 horas
  12 * 60 * 60 * 1000, // 12 horas
];
```

### 4.4. Fluxo de DLQ e replay

```
Replay (POST /admin/webhooks/dead-letter/:id/replay):
  1. Middleware authenticate + requireRole('ADMIN')
  2. Busca registro na webhook_dead_letter pelo id
  3. Se nao encontrado, retorna 404 WEBHOOK_DEAD_LETTER_NOT_FOUND
  4. Cria nova entrada na webhook_outbox com status 'PENDING',
     attempt_count = 0, next_retry_at = NULL, payload original
  5. Remove o registro da webhook_dead_letter
  6. Registra log de auditoria com o user_id do admin que executou o replay
  7. Retorna 200 com o novo evento criado na outbox
```

---

## 5. Contratos públicos

Base path: `/api/v1`

### 5.1. POST /webhooks — Criar configuração de webhook

Cria uma nova configuração de webhook para um cliente. A secret é gerada pelo sistema e devolvida na resposta.

**Autenticação:** Bearer JWT (qualquer role autenticada)

**Request:**

```http
POST /api/v1/webhooks HTTP/1.1
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://atlas-comercial.com/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"]
}
```

**Validações Zod:**
- `customerId`: `z.string().uuid()` — obrigatório
- `url`: `z.string().url()` — obrigatório, deve começar com `https://`
- `events`: `z.array(z.nativeEnum(OrderStatus)).min(1)` — ao menos um status

**Response 201 Created:**

```json
{
  "id": "f8e7d6c5-b4a3-2109-8765-432109876543",
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://atlas-comercial.com/webhooks/orders",
  "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"],
  "active": true,
  "secret": "whsec_k7G9mPqR2xVbN4cJfL8wTzYhA1sD6eU3",
  "createdAt": "2026-07-05T14:30:00.000Z",
  "updatedAt": "2026-07-05T14:30:00.000Z"
}
```

**Nota:** A `secret` só é retornada na criação e na rotação. Endpoints de listagem e edição nunca expõem a secret.

**Erros possíveis:**

| Status | Código                   | Condição                                      |
|--------|--------------------------|-----------------------------------------------|
| 400    | VALIDATION_ERROR         | Campos inválidos (Zod)                        |
| 400    | WEBHOOK_INVALID_URL      | URL não utiliza HTTPS                         |
| 401    | UNAUTHORIZED             | JWT ausente ou inválido                       |
| 404    | NOT_FOUND                | Customer não encontrado                       |
| 409    | WEBHOOK_DUPLICATE_URL    | Já existe webhook ativo com mesma URL para o customer |

---

### 5.2. GET /webhooks — Listar webhooks

Lista as configurações de webhook. Filtrável por `customerId`.

**Autenticação:** Bearer JWT (qualquer role autenticada)

**Request:**

```http
GET /api/v1/webhooks?customerId=a1b2c3d4-e5f6-7890-abcd-ef1234567890&page=1&pageSize=20 HTTP/1.1
Authorization: Bearer <jwt>
```

**Query params:**
- `customerId`: `z.string().uuid().optional()`
- `page`: `z.coerce.number().int().min(1).default(1)`
- `pageSize`: `z.coerce.number().int().min(1).max(100).default(20)`

**Response 200 OK:**

```json
{
  "data": [
    {
      "id": "f8e7d6c5-b4a3-2109-8765-432109876543",
      "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "url": "https://atlas-comercial.com/webhooks/orders",
      "events": ["PAID", "PROCESSING", "SHIPPED", "DELIVERED", "CANCELLED"],
      "active": true,
      "createdAt": "2026-07-05T14:30:00.000Z",
      "updatedAt": "2026-07-05T14:30:00.000Z"
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

**Nota:** A `secret` não é incluída na listagem.

---

### 5.3. PATCH /webhooks/:id — Atualizar configuração de webhook

Permite atualizar URL, lista de eventos e estado ativo/inativo.

**Autenticação:** Bearer JWT (qualquer role autenticada)

**Request:**

```http
PATCH /api/v1/webhooks/f8e7d6c5-b4a3-2109-8765-432109876543 HTTP/1.1
Authorization: Bearer <jwt>
Content-Type: application/json

{
  "url": "https://atlas-comercial.com/webhooks/v2/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": false
}
```

**Validações Zod:**
- `url`: `z.string().url().optional()` — se presente, deve ser HTTPS
- `events`: `z.array(z.nativeEnum(OrderStatus)).min(1).optional()`
- `active`: `z.boolean().optional()`

**Response 200 OK:**

```json
{
  "id": "f8e7d6c5-b4a3-2109-8765-432109876543",
  "customerId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "url": "https://atlas-comercial.com/webhooks/v2/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "active": false,
  "createdAt": "2026-07-05T14:30:00.000Z",
  "updatedAt": "2026-07-05T15:00:00.000Z"
}
```

**Erros possíveis:**

| Status | Código                   | Condição                               |
|--------|--------------------------|----------------------------------------|
| 400    | VALIDATION_ERROR         | Campos inválidos (Zod)                 |
| 400    | WEBHOOK_INVALID_URL      | URL não utiliza HTTPS                  |
| 401    | UNAUTHORIZED             | JWT ausente ou inválido                |
| 404    | WEBHOOK_NOT_FOUND        | Webhook não encontrado                 |

---

### 5.4. DELETE /webhooks/:id — Remover configuração de webhook

**Autenticação:** Bearer JWT (qualquer role autenticada)

**Request:**

```http
DELETE /api/v1/webhooks/f8e7d6c5-b4a3-2109-8765-432109876543 HTTP/1.1
Authorization: Bearer <jwt>
```

**Response 204 No Content** (sem body)

**Erros possíveis:**

| Status | Código            | Condição                  |
|--------|-------------------|---------------------------|
| 401    | UNAUTHORIZED      | JWT ausente ou inválido   |
| 404    | WEBHOOK_NOT_FOUND | Webhook não encontrado    |

---

### 5.5. POST /webhooks/:id/rotate-secret — Rotacionar secret

Gera uma nova secret para o endpoint. A secret anterior permanece válida por 24 horas (grace period).

**Autenticação:** Bearer JWT (qualquer role autenticada)

**Request:**

```http
POST /api/v1/webhooks/f8e7d6c5-b4a3-2109-8765-432109876543/rotate-secret HTTP/1.1
Authorization: Bearer <jwt>
```

**Response 200 OK:**

```json
{
  "id": "f8e7d6c5-b4a3-2109-8765-432109876543",
  "secret": "whsec_nEw5eCr3tG3nEr4tEdH3r3xYz9876543",
  "previousSecretExpiresAt": "2026-07-06T15:00:00.000Z"
}
```

**Erros possíveis:**

| Status | Código            | Condição                  |
|--------|-------------------|---------------------------|
| 401    | UNAUTHORIZED      | JWT ausente ou inválido   |
| 404    | WEBHOOK_NOT_FOUND | Webhook não encontrado    |

---

### 5.6. GET /webhooks/:id/deliveries — Histórico de entregas

Retorna o histórico de tentativas de entrega de um webhook.

**Autenticação:** Bearer JWT (qualquer role autenticada)

**Request:**

```http
GET /api/v1/webhooks/f8e7d6c5-b4a3-2109-8765-432109876543/deliveries?page=1&pageSize=20 HTTP/1.1
Authorization: Bearer <jwt>
```

**Query params:**
- `page`: `z.coerce.number().int().min(1).default(1)`
- `pageSize`: `z.coerce.number().int().min(1).max(100).default(20)`

**Response 200 OK:**

```json
{
  "data": [
    {
      "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
      "webhookConfigId": "f8e7d6c5-b4a3-2109-8765-432109876543",
      "eventId": "c9d8e7f6-a5b4-3210-9876-543210987654",
      "eventType": "order.status_changed",
      "httpStatus": 200,
      "success": true,
      "durationMs": 245,
      "attemptNumber": 1,
      "requestPayload": { "event_id": "...", "event_type": "order.status_changed", "..." : "..." },
      "responseBody": "{\"received\": true}",
      "createdAt": "2026-07-05T14:30:02.000Z"
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

---

### 5.7. POST /admin/webhooks/dead-letter/:id/replay — Reprocessar evento da DLQ

Recoloca um evento da Dead Letter Queue na outbox como pendente para reprocessamento.

**Autenticação:** Bearer JWT com role `ADMIN` (via `requireRole('ADMIN')`)

**Request:**

```http
POST /api/v1/admin/webhooks/dead-letter/abc12345-d6e7-8901-f234-567890abcdef/replay HTTP/1.1
Authorization: Bearer <jwt-admin>
```

**Response 200 OK:**

```json
{
  "message": "Evento reenfileirado com sucesso",
  "outboxEventId": "new-uuid-gerado-para-outbox",
  "originalEventId": "c9d8e7f6-a5b4-3210-9876-543210987654",
  "replayedBy": "admin-user-id",
  "replayedAt": "2026-07-05T16:00:00.000Z"
}
```

**Erros possíveis:**

| Status | Código                         | Condição                                |
|--------|--------------------------------|-----------------------------------------|
| 401    | UNAUTHORIZED                   | JWT ausente ou inválido                 |
| 403    | FORBIDDEN                      | Usuário não possui role ADMIN           |
| 404    | WEBHOOK_DEAD_LETTER_NOT_FOUND  | Registro não encontrado na DLQ         |

---

### Payload do webhook (enviado pelo worker)

Conforme definido na reunião ([09:43] Diego):

```json
{
  "event_id": "c9d8e7f6-a5b4-3210-9876-543210987654",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-05T14:30:01.000Z",
  "data": {
    "order_id": "b7a6c5d4-e3f2-1098-7654-321098765432",
    "order_number": "ORD-000001",
    "from_status": "PAID",
    "to_status": "PROCESSING",
    "customer_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
    "total_cents": 15000
  }
}
```

### Headers enviados pelo worker

| Header          | Valor                                    | Descrição                                    |
|-----------------|------------------------------------------|----------------------------------------------|
| Content-Type    | `application/json`                       | Tipo de conteúdo                             |
| X-Event-Id      | UUID do evento                           | Identificador único para deduplicação        |
| X-Signature     | `sha256=<hex>`                           | HMAC-SHA256 do body com a secret do endpoint |
| X-Timestamp     | ISO 8601 do instante de envio            | Para detecção de replay attack pelo cliente  |
| X-Webhook-Id    | UUID da configuração do webhook          | Identifica qual cadastro originou o envio    |

---

## 6. Matriz de erros

Todos os erros do módulo de webhooks estendem `AppError` (`src/shared/errors/app-error.ts`) e são tratados automaticamente pelo `errorMiddleware` existente (`src/middlewares/error.middleware.ts`).

| Código de erro                   | HTTP | Classe base              | Condição de disparo                                              |
|----------------------------------|------|--------------------------|------------------------------------------------------------------|
| `WEBHOOK_NOT_FOUND`              | 404  | `NotFoundError`          | Webhook config não encontrado pelo id                            |
| `WEBHOOK_INVALID_URL`            | 400  | `BadRequestError`        | URL não começa com `https://`                                    |
| `WEBHOOK_DUPLICATE_URL`          | 409  | `ConflictError`          | Já existe webhook ativo com mesma URL para o mesmo customer      |
| `WEBHOOK_SECRET_REQUIRED`        | 400  | `BadRequestError`        | Tentativa de operação sem secret configurada (erro interno)      |
| `WEBHOOK_INACTIVE`               | 422  | `UnprocessableEntityError`| Operação sobre webhook desativado (quando aplicável)             |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND`  | 404  | `NotFoundError`          | Registro não encontrado na tabela de dead letter                 |
| `WEBHOOK_PAYLOAD_TOO_LARGE`      | 422  | `UnprocessableEntityError`| Payload renderizado excede 64 KB                                 |
| `WEBHOOK_DELIVERY_FAILED`        | —    | Interno (log)            | Falha na entrega HTTP (não exposto via API, registrado em log)   |

---

## 7. Estratégias de resiliência

### Timeout de entrega HTTP

- **Valor:** 10 segundos ([09:42] Diego).
- Implementar via `AbortController` com `setTimeout` no `fetch`/`undici` ou equivalente.
- Timeout conta como falha e aciona o fluxo de retry.

### Retry com backoff

- **Total de tentativas:** 5.
- **Intervalos fixos:** 1m, 5m, 30m, 2h, 12h.
- **Cobertura total:** aproximadamente 15 horas entre primeira falha e última tentativa.
- O campo `next_retry_at` na outbox controla quando o evento pode ser buscado novamente pelo worker.
- O worker só seleciona eventos cujo `next_retry_at` seja `NULL` (primeira tentativa) ou `<= NOW()`.

### Dead Letter Queue

- Após 5a tentativa falhada, o evento é movido para `webhook_dead_letter`.
- A outbox fica limpa — sem eventos permanentemente falhados poluindo o polling.
- Reprocessamento manual via endpoint admin com auditoria.

### Tratamento de respostas HTTP

| Resposta do endpoint             | Ação do worker                                    |
|----------------------------------|---------------------------------------------------|
| 2xx                              | Marca como `DELIVERED`                            |
| 4xx (exceto 429)                 | Marca como falha, aciona retry                    |
| 429 (Too Many Requests)          | Marca como falha, aciona retry (respeitar Retry-After é um ponto em aberto) |
| 5xx                              | Marca como falha, aciona retry                    |
| Timeout (10s)                    | Marca como falha, aciona retry                    |
| Erro de rede (DNS, conexão)      | Marca como falha, aciona retry                    |

**Ponto em aberto:** a reunião não discutiu se respostas 4xx (erros do cliente) devem ter tratamento diferenciado em relação a retries. A implementação inicial tratará qualquer non-2xx como falha retentável.

### Lock otimista no worker

Para evitar que um restart do worker reprocesse eventos já em andamento, o status `PROCESSING` serve como lock. Se o worker morrer durante o processamento, o evento ficará travado em `PROCESSING`. Deve-se implementar um mecanismo de recuperação:

- Eventos em status `PROCESSING` há mais de 5 minutos (configurável) devem ser revertidos para `PENDING` no início de cada ciclo de polling. Isso cobre cenários de crash do worker.

**Ponto em aberto:** o tempo de lock (5 minutos sugerido) não foi discutido na reunião. O valor deve ser validado pela equipe antes da implementação.

---

## 8. Observabilidade

### Métricas sugeridas

O projeto não possui framework de métricas (Prometheus, StatsD etc.) instalado. As métricas abaixo devem ser emitidas via logs estruturados Pino, permitindo extração por ferramentas de log aggregation. Caso o time opte por adicionar um framework de métricas no futuro, esses pontos de instrumentação já estarão mapeados.

| Métrica                              | Tipo    | Descrição                                                  |
|--------------------------------------|---------|------------------------------------------------------------|
| `webhook.event.enqueued`             | Counter | Evento inserido na outbox                                  |
| `webhook.delivery.attempt`           | Counter | Tentativa de entrega (inclui retries)                      |
| `webhook.delivery.success`           | Counter | Entrega bem-sucedida (2xx)                                 |
| `webhook.delivery.failure`           | Counter | Entrega falhada (non-2xx, timeout, erro de rede)           |
| `webhook.delivery.duration_ms`       | Gauge   | Duração da chamada HTTP em milissegundos                   |
| `webhook.dead_letter.moved`          | Counter | Evento movido para DLQ após esgotar tentativas             |
| `webhook.dead_letter.replayed`       | Counter | Evento reprocessado da DLQ via endpoint admin              |
| `webhook.outbox.pending_count`       | Gauge   | Quantidade de eventos pendentes na outbox (emitido por ciclo de polling) |
| `webhook.worker.poll_cycle_duration` | Gauge   | Duração de um ciclo completo de polling do worker           |

### Logs estruturados (Pino)

Utilizar o logger Pino existente em `src/shared/logger/index.ts`. A instância do worker deve ser criada via `createLogger()` com campo `service` ajustado para `'webhook-worker'` para distinguir dos logs da API.

Exemplos de logs:

```typescript
// Evento enfileirado (dentro da transacao)
logger.info({ eventId, orderId, customerId, toStatus }, 'webhook.event.enqueued');

// Tentativa de entrega
logger.info({ eventId, webhookConfigId, url, attemptNumber }, 'webhook.delivery.attempt');

// Entrega bem-sucedida
logger.info({ eventId, webhookConfigId, httpStatus, durationMs }, 'webhook.delivery.success');

// Entrega falhada
logger.warn({ eventId, webhookConfigId, httpStatus, error, attemptNumber, nextRetryAt }, 'webhook.delivery.failure');

// Movido para DLQ
logger.error({ eventId, webhookConfigId, totalAttempts, lastError }, 'webhook.dead_letter.moved');

// Replay executado
logger.info({ deadLetterId, eventId, replayedBy }, 'webhook.dead_letter.replayed');
```

### Redação de dados sensíveis

A secret do webhook deve ser adicionada aos paths de redação do Pino. No `createLogger()`, adicionar:

```typescript
'*.secret',
'*.previousSecret',
```

### Tracing

O projeto não possui OpenTelemetry ou tracing distribuído configurado. O `event_id` (UUID) serve como correlation ID natural entre a inserção na outbox, as tentativas de entrega e o registro no delivery log. Todos os logs devem incluir `eventId` para permitir rastreamento end-to-end.

**Ponto em aberto:** a integração com ferramentas de tracing distribuído (OpenTelemetry, Datadog, etc.) não foi discutida na reunião e fica como evolução futura.

---

## 9. Dependências e compatibilidade

### Dependências existentes (nenhuma nova necessária)

| Dependência       | Uso no módulo de webhooks                                |
|-------------------|----------------------------------------------------------|
| `@prisma/client`  | Acesso ao banco (novas tabelas via schema)               |
| `zod`             | Validação de entrada nos endpoints                       |
| `pino`            | Logging estruturado (API e worker)                       |
| `jsonwebtoken`    | Autenticação JWT nos endpoints (via middleware existente) |
| `uuid` (v4)       | Geração de event_id na outbox                            |
| `express`         | Rotas HTTP do módulo                                     |

### Dependências novas potenciais

| Dependência       | Uso                                       | Observação                                         |
|-------------------|-------------------------------------------|----------------------------------------------------|
| Nenhuma obrigatória| —                                         | HMAC-SHA256 via `node:crypto` nativo. HTTP client via `node:https` ou `undici` (já bundled no Node.js moderno). |

**Nota:** A geração de secrets pode utilizar `crypto.randomBytes` do módulo `node:crypto` nativo, sem necessidade de biblioteca externa.

### Compatibilidade

- **Node.js:** versão compatível com o projeto atual (verificar `engines` no `package.json`). O módulo `node:crypto` para HMAC e `fetch`/`undici` para HTTP client estão disponíveis a partir do Node.js 18+.
- **MySQL:** nenhuma feature específica de versão necessária. As novas tabelas utilizam tipos já presentes no schema (`Char(36)`, `DateTime`, `Json`, `Text`, `Int`, `Boolean`).
- **Prisma:** a adição de novas tabelas requer uma nova migration (`npx prisma migrate dev`).

---

## 10. Critérios de aceite técnicos

### Funcionais

1. **Atomicidade:** ao mudar o status de um pedido que possui webhook configurado, o evento deve ser inserido na outbox dentro da mesma transação SQL. Se a transação falhar, nenhum evento é persistido.
2. **Despacho:** o worker deve entregar o webhook ao endpoint do cliente em até 10 segundos após a mudança de status (2s polling + tempo de processamento + tempo de rede).
3. **Retry:** uma falha de entrega deve resultar em até 5 tentativas com os intervalos 1m/5m/30m/2h/12h. Após a 5a falha, o evento deve estar na tabela `webhook_dead_letter`.
4. **HMAC:** o header `X-Signature` deve conter `sha256=<hex>` onde `<hex>` é o HMAC-SHA256 do body cru usando a secret do endpoint. O cliente deve conseguir validar recalculando.
5. **Rotação de secret:** após rotação, a secret anterior deve permanecer válida por 24 horas. Eventos enviados durante o grace period devem ser assinados com a secret nova.
6. **HTTPS obrigatório:** tentativa de cadastrar webhook com URL HTTP deve retornar 400 `WEBHOOK_INVALID_URL`.
7. **Filtragem de eventos:** se o webhook está configurado para ouvir apenas `SHIPPED` e `DELIVERED`, mudanças para `PAID` ou `PROCESSING` não devem gerar evento na outbox.
8. **Idempotência:** o `X-Event-Id` deve ser o mesmo UUID em todas as tentativas de entrega do mesmo evento.
9. **Replay:** um admin deve conseguir reprocessar um evento da DLQ via POST, e o evento deve voltar a outbox como pendente.
10. **Histórico:** o endpoint de deliveries deve retornar todas as tentativas de entrega do webhook, incluindo falhas, com status HTTP, duração e payload.

### Não-funcionais

11. **Payload máximo:** eventos com payload renderizado acima de 64 KB devem ser rejeitados na inserção (não inseridos na outbox), com log de erro.
12. **Timeout HTTP:** chamadas ao endpoint do cliente devem ser abortadas após 10 segundos.
13. **Isolamento:** crash ou restart da API não deve afetar o worker, e vice-versa.
14. **Secret nunca em log:** a secret do webhook não deve aparecer em nenhum log (Pino redact).
15. **Secret nunca em listagem:** endpoints GET nunca devem retornar a secret.

---

## 11. Riscos e mitigação

| # | Risco                                                                 | Probabilidade | Impacto | Mitigação                                                                                              |
|---|-----------------------------------------------------------------------|---------------|---------|--------------------------------------------------------------------------------------------------------|
| 1 | Worker morre e eventos acumulam na outbox                             | Baixa         | Alto    | Monitorar `webhook.outbox.pending_count`; alertar quando ultrapassar threshold (a definir)             |
| 2 | Transação de `changeStatus` fica mais lenta com escrita na outbox     | Baixa         | Baixo   | É uma única inserção simples; monitorar duração da transação via logs existentes                       |
| 3 | Cliente com endpoint permanentemente fora do ar gera carga de retry   | Média         | Baixo   | Política de 5 tentativas com DLQ; evento sai da fila ativa após ~15h                                  |
| 4 | Vazamento de secret do webhook                                        | Baixa         | Médio   | Secret isolada por endpoint; rotação via API com grace period de 24h; Pino redact configurado          |
| 5 | Eventos em `PROCESSING` ficam travados se worker crashar              | Baixa         | Médio   | Mecanismo de recuperação: reverter `PROCESSING` para `PENDING` após timeout configurável               |
| 6 | Tabela outbox cresce indefinidamente                                  | Média         | Médio   | Fora de escopo desta fase; registrar como dívida técnica para próxima iteração (sugestão: limpeza 30d) |
| 7 | Payload do webhook excede 64 KB por dados inesperados                 | Muito baixa   | Baixo   | Validação no momento da inserção na outbox; payload atual (sem items) tem tamanho fixo pequeno         |

---

## 12. Integração com o sistema existente

### 12.1. `src/modules/orders/order.service.ts` — Extensão do `changeStatus`

O método `changeStatus` (linhas 126-179) será estendido para incluir a inserção na outbox dentro do bloco `this.prisma.$transaction(async (tx) => { ... })`. A alteração será adicionada após a inserção no `orderStatusHistory` (linhas 159-167) e antes da query de refresh (linha 169).

A função `publishWebhookEvent` receberá o `tx` (tipo `Prisma.TransactionClient`, alias `TxClient` já definido na linha 24 do arquivo) e os dados necessários para montar o snapshot do payload:

```typescript
// Dentro do bloco da transacao, apos insercao no historico:
await publishWebhookEvent(tx, {
  orderId: id,
  orderNumber: order.orderNumber,  // vem do select do findUnique (linha 132)
  customerId: order.customerId,
  fromStatus: from,
  toStatus: to,
  totalCents: order.totalCents,
});
```

O `OrderService` não precisará conhecer o `WebhookRepository` — a função `publishWebhookEvent` será importada do módulo de webhooks e encapsulará a lógica de consulta de configs e inserção na outbox.

### 12.2. `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` — Reutilização

Os novos erros do módulo seguirão o padrão existente. Exemplos de classes a criar em `src/modules/webhooks/webhook.errors.ts`:

```typescript
export class WebhookNotFoundError extends NotFoundError {
  constructor() { super('Webhook'); }
  // errorCode sera 'NOT_FOUND' herdado, pode-se sobrescrever para 'WEBHOOK_NOT_FOUND'
}

export class WebhookInvalidUrlError extends BadRequestError {
  constructor() { super('Webhook URL must use HTTPS', 'WEBHOOK_INVALID_URL'); }
}

export class WebhookDuplicateUrlError extends ConflictError {
  constructor() { super('A webhook with this URL already exists for this customer', 'WEBHOOK_DUPLICATE_URL'); }
}
```

### 12.3. `src/middlewares/error.middleware.ts` — Sem alteração

O middleware já captura qualquer instância de `AppError` e retorna `{ error: { code, message, details } }` com o `statusCode` correto. Como todos os novos erros estendem `AppError`, serão tratados automaticamente sem nenhuma modificação neste arquivo.

### 12.4. `src/middlewares/auth.middleware.ts` — `requireRole('ADMIN')` para replay

O endpoint de replay da DLQ utilizará os middlewares existentes:

```typescript
router.post(
  '/admin/webhooks/dead-letter/:id/replay',
  authenticate,
  requireRole('ADMIN'),
  validate({ params: deadLetterIdParamSchema }),
  controller.replay,
);
```

A função `requireRole` (linha 49 do arquivo) já aceita roles variádicas e retorna `ForbiddenError` quando a role não confere. Nenhuma alteração necessária.

### 12.5. `src/shared/logger/index.ts` — Pino existente

O logger será reutilizado tanto na API (módulo de webhooks) quanto no worker. Para o worker, criar instância via `createLogger()` (já exportada) e configurar `base.service` como `'webhook-worker'` para distinguir nos logs.

Adicionar paths de redação para secrets:

```typescript
const redactPaths = [
  // ... paths existentes ...
  '*.secret',
  '*.previousSecret',
  '*.webhookSecret',
];
```

### 12.6. `src/server.ts` — Referência para `src/worker.ts`

O worker seguirá a mesma estrutura do `src/server.ts` (linhas 1-27): bootstrap assíncrono, graceful shutdown com `SIGINT`/`SIGTERM`, desconexão do PrismaClient. A diferença é que em vez de iniciar o Express, inicia o loop de polling:

```typescript
// src/worker.ts (estrutura analogica)
async function bootstrap(): Promise<void> {
  const prisma = createPrismaClient(); // instancia propria
  const processor = new WebhookProcessor(prisma);

  processor.start(); // inicia polling

  const shutdown = async (signal: string): Promise<void> => {
    logger.info({ signal }, 'worker_shutdown_initiated');
    processor.stop(); // para o polling graciosamente
    await prisma.$disconnect();
    process.exit(0);
  };

  process.on('SIGINT', () => void shutdown('SIGINT'));
  process.on('SIGTERM', () => void shutdown('SIGTERM'));
}
```

O `package.json` deverá incluir um novo script: `"worker": "tsx src/worker.ts"` (ou equivalente compilado).

### 12.7. `prisma/schema.prisma` — Novas tabelas

Quatro novos models a adicionar ao schema existente. O schema atual utiliza UUIDs `@db.Char(36)` para todos os IDs e `@@map()` para nomear tabelas em snake_case.

```prisma
model WebhookConfig {
  id              String    @id @default(uuid()) @db.Char(36)
  customerId      String    @db.Char(36)
  url             String    @db.VarChar(2048)
  secret          String    @db.VarChar(255)
  previousSecret  String?   @db.VarChar(255)
  previousSecretExpiresAt DateTime?
  events          Json      // Array de OrderStatus que o webhook escuta
  active          Boolean   @default(true)
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  customer Customer @relation(fields: [customerId], references: [id])

  outboxEvents    WebhookOutbox[]
  deliveryLogs    WebhookDeliveryLog[]
  deadLetters     WebhookDeadLetter[]

  @@index([customerId])
  @@index([active])
  @@map("webhook_configs")
}

model WebhookOutbox {
  id              String    @id @default(uuid()) @db.Char(36)
  webhookConfigId String    @db.Char(36)
  eventId         String    @unique @db.Char(36) // UUID unico do evento, enviado como X-Event-Id
  eventType       String    @db.VarChar(100)     // ex: "order.status_changed"
  payload         Json                           // snapshot do payload renderizado
  status          String    @db.VarChar(20)      // PENDING, PROCESSING, DELIVERED
  attemptCount    Int       @default(0)
  nextRetryAt     DateTime?
  createdAt       DateTime  @default(now())
  updatedAt       DateTime  @updatedAt

  webhookConfig WebhookConfig @relation(fields: [webhookConfigId], references: [id])

  @@index([status, nextRetryAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDeliveryLog {
  id              String    @id @default(uuid()) @db.Char(36)
  webhookConfigId String    @db.Char(36)
  eventId         String    @db.Char(36)
  attemptNumber   Int
  httpStatus      Int?
  success         Boolean
  durationMs      Int
  requestPayload  Json
  responseBody    String?   @db.Text
  errorMessage    String?   @db.Text
  createdAt       DateTime  @default(now())

  webhookConfig WebhookConfig @relation(fields: [webhookConfigId], references: [id])

  @@index([webhookConfigId])
  @@index([eventId])
  @@index([createdAt])
  @@map("webhook_delivery_logs")
}

model WebhookDeadLetter {
  id              String    @id @default(uuid()) @db.Char(36)
  webhookConfigId String    @db.Char(36)
  eventId         String    @db.Char(36)
  eventType       String    @db.VarChar(100)
  payload         Json
  failureReason   String    @db.Text
  totalAttempts   Int
  lastAttemptAt   DateTime
  createdAt       DateTime  @default(now())

  webhookConfig WebhookConfig @relation(fields: [webhookConfigId], references: [id])

  @@index([webhookConfigId])
  @@index([createdAt])
  @@map("webhook_dead_letters")
}
```

**Nota:** a relação `Customer -> WebhookConfig` requer adicionar o campo `webhooks WebhookConfig[]` ao model `Customer` existente no schema.

### 12.8. `src/routes/index.ts` — Registro do módulo

O tipo `Controllers` deverá ser estendido para incluir `webhooks: WebhookController` e o router deverá registrar as rotas:

```typescript
router.use('/webhooks', buildWebhookRouter(controllers.webhooks));
router.use('/admin/webhooks', buildWebhookAdminRouter(controllers.webhooks));
```

### 12.9. `src/app.ts` — Wiring de dependências

Seguindo o padrão de `buildControllers` (linhas 26-53), adicionar:

```typescript
const webhookRepository = new WebhookRepository(prisma);
const webhookService = new WebhookService(webhookRepository, prisma);
const webhookController = new WebhookController(webhookService);
```

---

## Estrutura de arquivos do módulo

```
src/modules/webhooks/
  webhook.controller.ts     — handlers HTTP
  webhook.service.ts        — lógica de negócio (CRUD, rotação, replay)
  webhook.repository.ts     — queries Prisma
  webhook.routes.ts         — rotas Express (CRUD + deliveries)
  webhook.admin.routes.ts   — rotas Express (replay DLQ, requireRole ADMIN)
  webhook.schemas.ts        — schemas Zod de validação
  webhook.errors.ts         — classes de erro com prefixo WEBHOOK_
  webhook.processor.ts      — lógica de polling, despacho HTTP, retry e DLQ
  webhook.signer.ts         — funções de HMAC-SHA256 e geração de secret

src/worker.ts               — entry-point do processo worker
```
