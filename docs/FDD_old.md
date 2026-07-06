# FDD -- Sistema de Webhooks de Notificacao de Pedidos

| Campo           | Valor                                                                 |
|-----------------|-----------------------------------------------------------------------|
| **Autor**       | Consolidado a partir da reuniao tecnica e documentos PRD/RFC          |
| **Status**      | Em elaboracao                                                         |
| **Data**        | 2026-07-04                                                            |
| **Pre-requisitos** | PRD e RFC aprovados                                                |

---

## 1. Contexto e motivacao tecnica

O Order Management System (OMS) e uma aplicacao Node.js/TypeScript estruturada em modulos independentes (`src/modules/`), cada um composto por controller, service, repository, routes e schemas Zod. O banco de dados e MySQL acessado via Prisma ORM. Toda entidade utiliza UUID como chave primaria (`@db.Char(36)`). O ciclo de vida de pedidos opera dentro de transacoes atomicas no metodo `OrderService.changeStatus`, que atualiza o status da order, insere registro em `order_status_history` e manipula estoque -- tudo dentro de `prisma.$transaction`.

Este FDD especifica a implementacao do sistema de webhooks outbound conforme arquitetura definida no RFC: padrao Transactional Outbox no MySQL existente, worker independente com polling, retry com backoff exponencial e Dead Letter Queue. O objetivo e fornecer detalhamento tecnico suficiente para que um desenvolvedor inicie a implementacao sem ambiguidades.

---

## 2. Objetivos tecnicos

1. Inserir eventos de webhook na tabela `webhook_outbox` dentro da transacao existente de `OrderService.changeStatus`, sem alterar a semantica transacional atual.
2. Implementar o modulo `src/modules/webhooks/` seguindo o padrao modular existente (controller, service, repository, routes, schemas).
3. Criar o worker como entry-point separado (`src/worker.ts`) com instancia propria de `PrismaClient`.
4. Garantir assinatura HMAC-SHA256 com secret por endpoint, incluindo suporte a rotacao com grace period.
5. Implementar retry com backoff exponencial (5 tentativas) e DLQ em tabela separada.
6. Expor endpoints REST para CRUD de configuracao, historico de entregas e replay administrativo.

---

## 3. Escopo e exclusoes

### Incluso neste FDD

- Modelagem das tabelas Prisma (webhook_configs, webhook_outbox, webhook_deliveries, webhook_dead_letters).
- Logica de insercao na outbox dentro da transacao de mudanca de status.
- Worker de despacho com polling, assinatura HMAC, envio HTTP e controle de retry.
- Endpoints CRUD de configuracao de webhook, historico de entregas e replay de DLQ.
- Matriz de erros, contratos HTTP e estrategias de resiliencia.

### Fora de escopo

- Notificacao por email ao cliente sobre falhas recorrentes (decisao da reuniao: fase futura).
- Rate limiting de saida (decisao da reuniao: observar antes de implementar).
- Dashboard visual (decisao da reuniao: projeto separado do time de frontend).
- Archival/cleanup de eventos antigos (mencionado como "30 dias" mas explicitamente fora desta fase).
- Escalabilidade para multiplos workers (decisao da reuniao: single-worker nesta fase).

---

## 4. Fluxos detalhados

### 4.1. Criacao do evento na outbox

**Trigger:** Chamada a `OrderService.changeStatus` com transicao de status valida.

**Fluxo:**

1. O metodo `changeStatus` inicia `prisma.$transaction(async (tx) => { ... })`.
2. Dentro da transacao, apos atualizar o status da order e inserir em `order_status_history`, o service verifica se existem webhooks ativos para o `customerId` da order que escutam o status de destino (`toStatus`).
3. Para cada webhook configurado que escuta o `toStatus`, uma funcao `publishWebhookEvent(tx, order, fromStatus, toStatus, webhookConfig)` insere um registro na tabela `webhook_outbox` com:
   - `id`: UUID gerado via `crypto.randomUUID()`.
   - `webhook_config_id`: referencia ao webhook configurado.
   - `event_type`: `"order.status_changed"`.
   - `payload`: JSON serializado (snapshot) contendo os dados do pedido no momento da transicao.
   - `status`: `"PENDING"`.
   - `attempt_count`: `0`.
   - `next_retry_at`: `null` (sera processado no proximo ciclo de polling).
   - `created_at`: timestamp atual.
4. Se a transacao commita, os eventos existem. Se sofre rollback, desaparecem junto.
5. Se nenhum webhook do customer escuta o `toStatus`, nenhum evento e inserido (filtragem na insercao, conforme decidido na reuniao).

**Funcao `publishWebhookEvent`:**

```typescript
async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: OrderWithRelations,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
  webhookConfig: { id: string; customerId: string },
): Promise<void> {
  const payload = {
    event_id: crypto.randomUUID(),
    event_type: 'order.status_changed',
    timestamp: new Date().toISOString(),
    data: {
      order_id: order.id,
      order_number: order.orderNumber,
      customer_id: order.customerId,
      from_status: fromStatus,
      to_status: toStatus,
      total_cents: order.totalCents,
    },
  };

  await tx.webhookOutbox.create({
    data: {
      id: payload.event_id,
      webhookConfigId: webhookConfig.id,
      eventType: payload.event_type,
      payload: JSON.stringify(payload),
      status: 'PENDING',
      attemptCount: 0,
      nextRetryAt: null,
    },
  });
}
```

**Observacao:** A funcao recebe o `tx` (transaction client) da transacao em andamento do `OrderService.changeStatus`. Isso garante atomicidade sem alterar a assinatura da transacao existente. A funcao e importada no `OrderService` a partir do modulo de webhooks.

### 4.2. Processamento pelo worker

**Trigger:** Loop de polling a cada 2 segundos.

**Fluxo:**

1. O worker inicia em `src/worker.ts` como processo Node.js separado, instanciando seu proprio `PrismaClient` com a mesma `DATABASE_URL`.
2. O loop principal executa a cada 2 segundos:
   a. Consulta `webhook_outbox` por registros com `status = 'PENDING'` e (`next_retry_at IS NULL` OU `next_retry_at <= NOW()`), ordenados por `created_at ASC`, limitados a um batch (sugestao: 10 registros por ciclo).
   b. Para cada evento no batch:
      - Marca o registro como `status = 'PROCESSING'` para evitar reprocessamento concorrente (relevante para eventual escala futura).
      - Carrega a configuracao do webhook (`webhook_configs`) para obter `url`, `secret` e `previous_secret`.
      - Monta os headers (ver secao 5.2).
      - Gera a assinatura HMAC-SHA256 sobre o corpo do payload usando a `secret` ativa.
      - Envia HTTP POST ao endpoint do cliente com timeout de 10 segundos.
      - Se resposta 2xx: marca como `DELIVERED`, registra em `webhook_deliveries`.
      - Se resposta nao-2xx ou timeout: incrementa `attempt_count`, calcula `next_retry_at` com backoff, marca como `PENDING` para retry. Registra a tentativa em `webhook_deliveries`.
      - Se `attempt_count` atingiu 5: move para DLQ (ver secao 4.4).

3. O worker trata graceful shutdown via `SIGINT`/`SIGTERM`, seguindo o mesmo padrao de `src/server.ts`:
   - Para de buscar novos batches.
   - Aguarda o batch atual terminar.
   - Desconecta o `PrismaClient`.
   - Encerra o processo.

### 4.3. Retry com backoff exponencial

Quando uma tentativa de envio falha (resposta nao-2xx, timeout, erro de rede), o worker calcula o proximo horario de retry com base no numero de tentativas ja realizadas:

| Tentativa | Intervalo de backoff | `next_retry_at` (a partir da falha) |
|-----------|---------------------|-------------------------------------|
| 1a falha  | 1 minuto            | `NOW() + 1 min`                     |
| 2a falha  | 5 minutos           | `NOW() + 5 min`                     |
| 3a falha  | 30 minutos          | `NOW() + 30 min`                    |
| 4a falha  | 2 horas             | `NOW() + 2 h`                       |
| 5a falha  | --                  | Move para DLQ                       |

**Implementacao:**

```typescript
const BACKOFF_INTERVALS_MS = [
  1 * 60 * 1000,       // 1 minuto
  5 * 60 * 1000,       // 5 minutos
  30 * 60 * 1000,      // 30 minutos
  2 * 60 * 60 * 1000,  // 2 horas
];

function getNextRetryAt(attemptCount: number): Date | null {
  if (attemptCount >= BACKOFF_INTERVALS_MS.length) return null; // vai para DLQ
  return new Date(Date.now() + BACKOFF_INTERVALS_MS[attemptCount]);
}
```

O campo `next_retry_at` e atualizado no registro da outbox. O worker so seleciona registros cujo `next_retry_at` ja passou ou e `null`.

### 4.4. Dead Letter Queue (DLQ)

Quando o `attempt_count` atinge 5 (apos a 5a tentativa falhada):

1. O worker insere um registro na tabela `webhook_dead_letters` contendo:
   - `id`: UUID.
   - `outbox_id`: referencia ao evento original.
   - `webhook_config_id`: referencia a configuracao.
   - `event_type`: tipo do evento.
   - `payload`: payload original completo.
   - `last_error`: mensagem/codigo do ultimo erro (ex: `"HTTP 503"`, `"TIMEOUT"`, `"ECONNREFUSED"`).
   - `last_http_status`: codigo HTTP da ultima tentativa (se aplicavel).
   - `failed_at`: timestamp.
2. O registro na outbox e removido (ou marcado como `DEAD_LETTER`).
3. O administrador pode reprocessar via `POST /api/v1/admin/webhooks/dead-letter/:id/replay`:
   - Valida que o usuario tem role `ADMIN`.
   - Cria um novo registro na outbox com `status = 'PENDING'`, `attempt_count = 0`, `next_retry_at = null`.
   - Registra em log (Pino) quem executou o replay, incluindo `user.id` e `user.email` para auditoria.
   - Remove o registro da DLQ (ou marca como `REPLAYED`).

---

## 5. Contratos publicos

### 5.1. Endpoints HTTP

Todos os endpoints seguem o padrao existente: prefixo `/api/v1`, autenticacao via `authenticate` middleware, validacao via `validate` middleware com schemas Zod.

#### 5.1.1. Criar configuracao de webhook

```
POST /api/v1/webhooks
```

**Autorizacao:** qualquer role autenticada (ADMIN ou OPERATOR).

**Request body:**

```json
{
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://atlas-comercial.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"]
}
```

**Validacoes:**
- `customerId`: UUID valido. O customer deve existir no banco.
- `url`: string, deve iniciar com `https://`. URLs `http://` sao rejeitadas.
- `events`: array nao vazio de valores do enum `OrderStatus`.

**Response 201:**

```json
{
  "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "customerId": "550e8400-e29b-41d4-a716-446655440000",
  "url": "https://atlas-comercial.com/webhooks/orders",
  "events": ["SHIPPED", "DELIVERED"],
  "secret": "whsec_k7Gj9mPqR2xL5nW8vY1bF4hT6dA0cE3",
  "active": true,
  "createdAt": "2026-07-04T10:00:00.000Z",
  "updatedAt": "2026-07-04T10:00:00.000Z"
}
```

**Nota:** A `secret` e gerada pelo sistema (32 bytes aleatorios, codificados em base64 com prefixo `whsec_`) e devolvida **apenas na criacao**. Chamadas subsequentes de GET nao retornam a secret.

**Response 400 (URL invalida):**

```json
{
  "error": {
    "code": "WEBHOOK_INVALID_URL",
    "message": "Webhook URL must use HTTPS"
  }
}
```

**Response 404 (customer nao encontrado):**

```json
{
  "error": {
    "code": "NOT_FOUND",
    "message": "Customer not found"
  }
}
```

#### 5.1.2. Listar webhooks de um customer

```
GET /api/v1/webhooks?customerId=550e8400-e29b-41d4-a716-446655440000&page=1&pageSize=20
```

**Autorizacao:** qualquer role autenticada.

**Response 200:**

```json
{
  "data": [
    {
      "id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "customerId": "550e8400-e29b-41d4-a716-446655440000",
      "url": "https://atlas-comercial.com/webhooks/orders",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-07-04T10:00:00.000Z",
      "updatedAt": "2026-07-04T10:00:00.000Z"
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

**Nota:** O campo `secret` nao e retornado na listagem.

#### 5.1.3. Atualizar configuracao de webhook

```
PATCH /api/v1/webhooks/:id
```

**Autorizacao:** qualquer role autenticada.

**Request body (parcial):**

```json
{
  "url": "https://atlas-comercial.com/webhooks/v2/orders",
  "events": ["PAID", "SHIPPED", "DELIVERED"],
  "active": false
}
```

Todos os campos sao opcionais (partial update, como no padrao de `PATCH /customers/:id`). A `secret` nao pode ser alterada via PATCH; para rotacao, usar o endpoint especifico.

**Response 200:** Objeto webhook atualizado (sem `secret`).

**Response 404:**

```json
{
  "error": {
    "code": "WEBHOOK_NOT_FOUND",
    "message": "Webhook not found"
  }
}
```

#### 5.1.4. Remover configuracao de webhook

```
DELETE /api/v1/webhooks/:id
```

**Autorizacao:** qualquer role autenticada.

**Response 204:** Sem corpo.

**Response 404:** `WEBHOOK_NOT_FOUND`.

#### 5.1.5. Rotacionar secret

```
POST /api/v1/webhooks/:id/rotate-secret
```

**Autorizacao:** qualquer role autenticada.

**Request body:** Nenhum.

**Comportamento:**
1. Gera nova secret.
2. Move a secret atual para `previous_secret` com `previous_secret_expires_at = NOW() + 24h`.
3. Retorna a nova secret na resposta.

**Response 200:**

```json
{
  "secret": "whsec_N3wS3cr3tG3n3r4t3dH3r3R4nd0m1y",
  "previousSecretExpiresAt": "2026-07-05T10:00:00.000Z"
}
```

**Response 404:** `WEBHOOK_NOT_FOUND`.

#### 5.1.6. Listar historico de entregas

```
GET /api/v1/webhooks/:id/deliveries?page=1&pageSize=20
```

**Autorizacao:** qualquer role autenticada.

**Response 200:**

```json
{
  "data": [
    {
      "id": "d1e2f3a4-b5c6-7890-abcd-ef1234567890",
      "eventId": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
      "eventType": "order.status_changed",
      "httpStatus": 200,
      "success": true,
      "durationMs": 245,
      "attemptNumber": 1,
      "error": null,
      "createdAt": "2026-07-04T10:00:02.000Z"
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

#### 5.1.7. Replay de evento da DLQ

```
POST /api/v1/admin/webhooks/dead-letter/:id/replay
```

**Autorizacao:** role `ADMIN` obrigatoria (via `requireRole('ADMIN')`).

**Request body:** Nenhum.

**Response 200:**

```json
{
  "message": "Event requeued for delivery",
  "outboxId": "new-uuid-do-evento-reinserido"
}
```

**Response 403 (role insuficiente):**

```json
{
  "error": {
    "code": "FORBIDDEN",
    "message": "Insufficient permissions"
  }
}
```

**Response 404:** `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

#### 5.1.8. Listar eventos na DLQ

```
GET /api/v1/admin/webhooks/dead-letter?page=1&pageSize=20
```

**Autorizacao:** role `ADMIN` obrigatoria.

**Response 200:**

```json
{
  "data": [
    {
      "id": "dl-uuid",
      "webhookConfigId": "config-uuid",
      "eventType": "order.status_changed",
      "payload": { "...": "..." },
      "lastError": "HTTP 503",
      "lastHttpStatus": 503,
      "failedAt": "2026-07-04T22:00:00.000Z"
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

### 5.2. Headers do webhook HTTP POST (enviados pelo worker ao endpoint do cliente)

| Header           | Valor                                      | Descricao                                                        |
|------------------|--------------------------------------------|------------------------------------------------------------------|
| `Content-Type`   | `application/json`                         | Tipo do corpo da requisicao.                                     |
| `X-Event-Id`     | UUID do evento (ex: `a1b2c3d4-...`)        | Identificador unico do evento para deduplicacao pelo cliente.    |
| `X-Webhook-Id`   | UUID da configuracao de webhook            | Identifica qual cadastro de webhook gerou o envio.               |
| `X-Signature`    | HMAC-SHA256 hex-encoded                    | Assinatura do corpo do request para validacao de autenticidade.  |
| `X-Timestamp`    | ISO 8601 (ex: `2026-07-04T10:00:02.000Z`) | Timestamp do envio para deteccao de replay attack pelo cliente.  |

**Calculo da assinatura:**

```typescript
import { createHmac } from 'node:crypto';

function signPayload(payload: string, secret: string): string {
  return createHmac('sha256', secret).update(payload).digest('hex');
}
```

O corpo e assinado como string JSON exatamente como sera enviado (sem reformatacao). O cliente recalcula o HMAC com a mesma secret e compara.

**Grace period de rotacao:** Durante o grace period de 24 horas, o worker gera a assinatura com a `secret` atual. Se o cliente ja rotacionou do lado dele, ele pode verificar com a `previous_secret` ate o vencimento. O worker **nao** envia duas assinaturas; a assinatura e sempre feita com a secret ativa mais recente. E responsabilidade do cliente tentar verificar com ambas as chaves durante o periodo de transicao.

**Payload do webhook POST (corpo enviado ao cliente):**

```json
{
  "event_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890",
  "event_type": "order.status_changed",
  "timestamp": "2026-07-04T10:00:00.000Z",
  "data": {
    "order_id": "550e8400-e29b-41d4-a716-446655440000",
    "order_number": "ORD-000042",
    "customer_id": "660e8400-e29b-41d4-a716-446655440000",
    "from_status": "PROCESSING",
    "to_status": "SHIPPED",
    "total_cents": 150000
  }
}
```

---

## 6. Matriz de erros previstos

Todos os erros seguem o padrao existente: estendem `AppError` (ou suas subclasses como `NotFoundError`, `BadRequestError`, `ConflictError`) e sao tratados automaticamente pelo `errorMiddleware` em `src/middlewares/error.middleware.ts`.

| Codigo de erro                     | HTTP Status | Classe base              | Quando ocorre                                                                      |
|------------------------------------|-------------|--------------------------|------------------------------------------------------------------------------------|
| `WEBHOOK_NOT_FOUND`                | 404         | `NotFoundError`          | GET/PATCH/DELETE de webhook com id inexistente.                                    |
| `WEBHOOK_INVALID_URL`             | 400         | `BadRequestError`        | URL do webhook nao usa HTTPS.                                                      |
| `WEBHOOK_SECRET_REQUIRED`         | 400         | `BadRequestError`        | Tentativa de criar webhook sem que o sistema consiga gerar secret (erro interno).   |
| `WEBHOOK_CUSTOMER_NOT_FOUND`      | 404         | `NotFoundError`          | `customerId` informado na criacao nao corresponde a nenhum customer existente.      |
| `WEBHOOK_DUPLICATE_URL`           | 409         | `ConflictError`          | Ja existe webhook ativo para o mesmo `customerId` + `url`.                         |
| `WEBHOOK_PAYLOAD_TOO_LARGE`       | 422         | `UnprocessableEntityError` | Payload do evento excede 64KB (verificado na insercao da outbox).                |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND`   | 404         | `NotFoundError`          | Tentativa de replay de id inexistente na DLQ.                                      |
| `WEBHOOK_ALREADY_REPLAYED`        | 409         | `ConflictError`          | Evento da DLQ ja foi reprocessado anteriormente.                                   |
| `VALIDATION_ERROR`                | 400         | `ValidationError`        | Falha na validacao Zod dos schemas de entrada (padrao existente).                  |
| `FORBIDDEN`                       | 403         | `ForbiddenError`         | Tentativa de acessar endpoint admin sem role ADMIN (padrao existente).              |

**Implementacao sugerida (novo arquivo `src/modules/webhooks/webhook.errors.ts`):**

```typescript
import {
  BadRequestError,
  NotFoundError,
  ConflictError,
  UnprocessableEntityError,
} from '../../shared/errors/index.js';

export class WebhookNotFoundError extends NotFoundError {
  constructor() {
    super('Webhook');
    // Nota: NotFoundError gera codigo 'NOT_FOUND' por padrao.
    // Para usar 'WEBHOOK_NOT_FOUND', sobrescrever this.errorCode aqui
    // ou ajustar o construtor de NotFoundError para aceitar code opcional.
  }
}

export class WebhookInvalidUrlError extends BadRequestError {
  constructor() {
    super('Webhook URL must use HTTPS', 'WEBHOOK_INVALID_URL');
  }
}

export class WebhookDuplicateUrlError extends ConflictError {
  constructor() {
    super(
      'Webhook already exists for this customer and URL',
      'WEBHOOK_DUPLICATE_URL',
    );
  }
}

export class WebhookPayloadTooLargeError extends UnprocessableEntityError {
  constructor(sizeBytes: number) {
    super(
      'Webhook event payload exceeds 64KB limit',
      'WEBHOOK_PAYLOAD_TOO_LARGE',
      { sizeBytes, limitBytes: 65536 },
    );
  }
}

export class WebhookDeadLetterNotFoundError extends NotFoundError {
  constructor() {
    super('Webhook dead letter entry');
  }
}

export class WebhookAlreadyReplayedError extends ConflictError {
  constructor() {
    super(
      'Dead letter entry has already been replayed',
      'WEBHOOK_ALREADY_REPLAYED',
    );
  }
}
```

**Nota sobre `WebhookNotFoundError`:** A classe `NotFoundError` existente gera o codigo `NOT_FOUND` fixo. Para manter o prefixo `WEBHOOK_` conforme decidido na reuniao, a subclasse deve sobrescrever o `errorCode` para `WEBHOOK_NOT_FOUND`. Alternativamente, o construtor de `NotFoundError` pode ser estendido para aceitar um `code` opcional -- o que exigiria uma alteracao minima em `src/shared/errors/http-errors.ts`. A decisao de como implementar fica a criterio do desenvolvedor, desde que o codigo de erro retornado ao cliente use o prefixo `WEBHOOK_`.

---

## 7. Estrategias de resiliencia

### 7.1. Timeout

- Timeout de **10 segundos** para cada chamada HTTP ao endpoint do cliente.
- Utilizar `AbortController` com `setTimeout` na `fetch` nativa do Node.js 20+:

```typescript
async function sendWebhook(
  url: string,
  payload: string,
  headers: Record<string, string>,
): Promise<Response> {
  const controller = new AbortController();
  const timeout = setTimeout(() => controller.abort(), 10_000);
  try {
    return await fetch(url, {
      method: 'POST',
      headers: { ...headers, 'Content-Type': 'application/json' },
      body: payload,
      signal: controller.signal,
    });
  } finally {
    clearTimeout(timeout);
  }
}
```

### 7.2. Retries

- **5 tentativas** no total (a primeira tentativa mais 4 retries).
- Backoff exponencial fixo: 1m, 5m, 30m, 2h. Apos a 5a tentativa falhada, evento vai para DLQ.
- O worker nao bloqueia esperando o backoff; ele grava `next_retry_at` no banco e segue processando outros eventos. No proximo ciclo de polling, so seleciona eventos cujo `next_retry_at` ja passou.

### 7.3. Criterios de falha

Uma tentativa e considerada falha quando:
- O endpoint retorna status HTTP fora da faixa 2xx (200-299).
- A requisicao excede o timeout de 10 segundos (`AbortError`).
- Ocorre erro de rede (DNS, conexao recusada, TLS invalido).

### 7.4. Fallback (DLQ)

- Apos esgotamento das 5 tentativas, o evento e preservado na tabela `webhook_dead_letters`.
- Nenhum evento e descartado silenciosamente.
- O replay manual via endpoint admin reinicia o ciclo de tentativas (o evento volta para a outbox com `attempt_count = 0`).

### 7.5. Protecao do worker

- O worker processa em batches pequenos (10 eventos por ciclo) para evitar sobrecarga.
- Cada evento e processado sequencialmente dentro do batch (garantia de ordenacao por `created_at`).
- Se o worker cai, eventos permanecem com `status = 'PENDING'` ou `status = 'PROCESSING'` no banco. Ao reiniciar, o worker retoma de onde parou. Eventos em `PROCESSING` que nao foram finalizados devem ser recuperados: no inicio do worker, marcar eventos em `PROCESSING` ha mais de 60 segundos de volta para `PENDING`.

---

## 8. Observabilidade

### 8.1. Logs

Todos os logs utilizam o logger Pino existente (`src/shared/logger/index.ts`). O worker instancia seu proprio logger com o campo `service: 'webhook-worker'` para diferenciacao nos logs.

| Evento de log                     | Nivel   | Campos                                                                                    |
|-----------------------------------|---------|-------------------------------------------------------------------------------------------|
| Worker iniciado                   | `info`  | `{ pollingIntervalMs: 2000 }`                                                            |
| Worker parado (shutdown)          | `info`  | `{ signal: 'SIGTERM' \| 'SIGINT' }`                                                      |
| Evento processado com sucesso     | `info`  | `{ eventId, webhookConfigId, url, httpStatus, durationMs }`                               |
| Evento falhou (retry agendado)    | `warn`  | `{ eventId, webhookConfigId, url, error, attemptCount, nextRetryAt }`                     |
| Evento movido para DLQ            | `error` | `{ eventId, webhookConfigId, url, lastError, attemptCount }`                              |
| Replay de DLQ executado           | `info`  | `{ deadLetterId, eventId, replayedBy: { userId, email } }`                                |
| Erro inesperado no worker         | `error` | `{ err, context: 'worker_loop' }`                                                        |
| Batch processado                  | `debug` | `{ batchSize, processedCount, failedCount, durationMs }`                                  |

### 8.2. Metricas

A transcricao da reuniao nao definiu requisitos especificos de metricas ou sistema de monitoramento. Como o projeto atual nao possui infraestrutura de metricas (Prometheus, Datadog, etc.), a implementacao inicial se apoia exclusivamente em logs estruturados. Metricas podem ser extraidas dos logs pelo sistema de observabilidade que a equipe utilizar.

Metricas recomendadas para extracao dos logs:
- Total de eventos processados (sucesso vs. falha) por intervalo de tempo.
- Latencia de entrega (diferenca entre `created_at` da outbox e timestamp do envio bem-sucedido).
- Tamanho da fila de outbox pendente.
- Quantidade de eventos na DLQ.

### 8.3. Tracing

O projeto atual nao possui instrumentacao de distributed tracing (OpenTelemetry, Jaeger, etc.). Nao foi discutido na reuniao. O `eventId` (UUID unico por evento) serve como correlacao entre o log de insercao na outbox (API), os logs de tentativa de envio (worker) e o registro na tabela de deliveries.

---

## 9. Dependencias e compatibilidade

### 9.1. Dependencias de runtime

- **Nenhuma nova dependencia NPM.** A feature utiliza:
  - `node:crypto` (nativo): geracao de secrets (`crypto.randomBytes`) e HMAC-SHA256 (`crypto.createHmac`).
  - `fetch` (nativo, Node.js 20+): chamadas HTTP ao endpoint do cliente. O projeto ja exige `node >= 20` no `package.json` (`"engines": { "node": ">=20" }`).
  - Prisma Client: mesmo ORM ja em uso.
  - Pino: mesmo logger ja em uso.
  - Zod: mesma lib de validacao ja em uso.

### 9.2. Dependencias de banco

- **Novas tabelas:** `webhook_configs`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letters`.
- **Nenhuma alteracao em tabelas existentes.** A insercao na outbox e feita pelo application layer dentro da transacao, nao por trigger ou foreign key em tabelas existentes.
- **Nova migration Prisma** necessaria.
- **Nota:** O modelo `Customer` existente em `prisma/schema.prisma` precisa receber a relacao inversa `webhookConfigs WebhookConfig[]` para que o Prisma aceite a foreign key de `WebhookConfig.customerId`. Essa e uma alteracao declarativa (nao gera coluna nova na tabela `customers`).

### 9.3. Dependencias de infraestrutura

- **Novo processo:** O worker roda como processo separado. Necessita de scripts adicionais no `package.json`:
  ```json
  "worker": "tsx watch --env-file=.env src/worker.ts",
  "worker:start": "node --env-file=.env dist/worker.js"
  ```
- **Mesma `DATABASE_URL`:** Nao requer banco ou instancia adicional.

### 9.4. Compatibilidade

- A feature nao altera nenhum endpoint existente. Todos os endpoints atuais continuam funcionando sem mudanca.
- A unica alteracao em codigo existente e a insercao da chamada a `publishWebhookEvent` dentro do metodo `OrderService.changeStatus`. Se nenhum webhook estiver configurado para o customer/status, a funcao nao executa nenhuma query adicional (short-circuit).

---

## 10. Criterios de aceite tecnicos

### Consistencia transacional

- [ ] A insercao na `webhook_outbox` ocorre dentro da mesma `prisma.$transaction` que atualiza o status da order e insere em `order_status_history`.
- [ ] Se a transacao sofre rollback (ex: estoque insuficiente), nenhum registro existe na outbox.
- [ ] Se a transacao commita e existem webhooks configurados para o `toStatus`, existe pelo menos um registro correspondente na outbox.
- [ ] O payload armazenado na outbox e um snapshot do estado da order no momento da transicao (nao e renderizado sob demanda).

### Worker

- [ ] O worker roda como processo Node.js separado, iniciado via `npm run worker`.
- [ ] O worker utiliza instancia propria de `PrismaClient` (nao compartilha com a API).
- [ ] O polling ocorre a cada 2 segundos.
- [ ] Graceful shutdown via `SIGINT`/`SIGTERM` aguarda batch atual antes de encerrar.
- [ ] Eventos em `PROCESSING` ha mais de 60 segundos sao recuperados para `PENDING` na inicializacao do worker.

### Entrega HTTP

- [ ] O worker envia HTTP POST ao endpoint cadastrado com headers `X-Event-Id`, `X-Webhook-Id`, `X-Signature`, `X-Timestamp` e `Content-Type: application/json`.
- [ ] A assinatura e calculada com HMAC-SHA256 sobre o corpo do request usando a secret ativa do webhook.
- [ ] Timeout de 10 segundos por chamada.
- [ ] Respostas 2xx sao tratadas como sucesso; demais como falha.

### Retry e DLQ

- [ ] Apos falha, o evento e agendado para retry com backoff: 1m, 5m, 30m, 2h.
- [ ] Apos 5 tentativas falhadas, o evento e movido para `webhook_dead_letters`.
- [ ] Endpoint `POST /api/v1/admin/webhooks/dead-letter/:id/replay` reinsere o evento na outbox com `attempt_count = 0`.
- [ ] O endpoint de replay exige role `ADMIN` e registra em log quem executou a acao.

### Seguranca

- [ ] URLs HTTP sao rejeitadas no cadastro com erro `WEBHOOK_INVALID_URL`.
- [ ] Secrets sao geradas pelo sistema com `crypto.randomBytes(32)`.
- [ ] Rotacao de secret preserva a secret anterior por 24 horas em `previous_secret`.
- [ ] A secret nao e retornada em endpoints de listagem (GET), apenas na criacao e na rotacao.

### Configuracao e consulta

- [ ] CRUD completo funciona: criar (POST), listar (GET), atualizar (PATCH), remover (DELETE).
- [ ] Historico de entregas retorna status, duracao, tentativa, erro e timestamp.
- [ ] Paginacao segue o padrao existente (`paginated()` de `src/shared/http/response.ts`).

### Erros

- [ ] Todos os erros do modulo utilizam a hierarquia de `AppError` com codigos prefixados `WEBHOOK_`.
- [ ] O `errorMiddleware` existente trata os erros sem alteracao.

### Aderencia a padroes

- [ ] Modulo organizado em `src/modules/webhooks/` com controller, service, repository, routes, schemas e errors.
- [ ] Schemas de validacao escritos com Zod.
- [ ] Logs escritos com Pino.
- [ ] Rotas registradas no router central (`src/routes/index.ts`).

---

## 11. Riscos e mitigacao

| # | Risco | Mitigacao tecnica |
|---|-------|-------------------|
| R1 | **Transacao de `changeStatus` fica mais lenta** pela consulta de webhooks configurados e insercao na outbox. | A consulta de webhooks e uma leitura simples por `customerId` + `active = true` com indice. A insercao na outbox e um `INSERT` unitario. Impacto estimado: < 5ms adicionais. Monitorar com logs de `durationMs` do `request-logger`. |
| R2 | **Acumulo de eventos na outbox** se o worker cair ou ficar parado. | Eventos permanecem no banco e sao processados quando o worker reinicia. Monitorar tamanho da tabela via query periodica. Alertar se outbox pendente ultrapassar threshold (definir em operacao). |
| R3 | **Worker em `PROCESSING` cai e deixa eventos travados.** | Na inicializacao, o worker reseta eventos em `PROCESSING` ha mais de 60 segundos para `PENDING`. |
| R4 | **Clientes com endpoints lentos causam backlog.** | Timeout de 10 segundos limita o tempo por tentativa. O batch size de 10 limita o impacto por ciclo. Limitacao conhecida do single-worker. |
| R5 | **Secret comprometida de um cliente.** | Secret unica por endpoint isola o impacto. Endpoint de rotacao com grace period de 24h permite troca sem interrupcao. Comunicacao exclusivamente via HTTPS. |
| R6 | **Payload excede 64KB.** | Verificacao de tamanho antes da insercao na outbox. Payload e enxuto por design (sem items do pedido). Erro `WEBHOOK_PAYLOAD_TOO_LARGE` impede insercao. |
| R7 | **Concorrencia futura com multiplos workers.** | Campo `status = 'PROCESSING'` atua como lock otimista. Para escala real, sera necessario lock pessimista ou particionamento por `order_id` (fora de escopo desta fase). |

---

## 12. Integracao com o sistema existente

### 12.1. `src/modules/orders/order.service.ts`

**O que muda:** O metodo `changeStatus` (linha 126) e o unico ponto de alteracao em codigo existente que afeta fluxo critico. Dentro da transacao `prisma.$transaction` (linha 131), apos inserir o registro em `orderStatusHistory` (linha 159) e antes de fazer o `findUnique` final para retornar o `refreshed` (linha 169), adiciona-se a chamada a `publishWebhookEvent(tx, order, from, to)`. A funcao e importada de `src/modules/webhooks/webhook.outbox.ts`. Internamente, ela consulta `webhook_configs` por `customerId` + `active = true` + `events` contendo `toStatus`, e insere na outbox para cada match encontrado. Se nenhum webhook estiver configurado, a funcao retorna sem executar nenhuma query de insercao. A order usada como base para o snapshot e o objeto `order` ja disponivel na transacao (carregado na linha 132 com `include: { items: true }`), o que evita query adicional.

### 12.2. `src/app.ts`

**O que muda:** O arquivo `src/app.ts` e onde os modulos sao instanciados (funcao `buildControllers`, linhas 26-53) e o router e montado. Sera necessario:
1. Instanciar `WebhookRepository`, `WebhookService` e `WebhookController` seguindo o padrao identico ao das linhas 42-44 (onde `OrderRepository`, `OrderService` e `OrderController` sao criados).
2. Adicionar `webhooks: webhookController` ao objeto `Controllers` retornado por `buildControllers`.
3. O `OrderService` precisara receber uma dependencia adicional (funcao ou referencia ao modulo de webhooks) para conseguir chamar `publishWebhookEvent` dentro da transacao. A abordagem mais simples e importar a funcao diretamente no `order.service.ts`, sem alterar o construtor do `OrderService`.

### 12.3. `src/routes/index.ts`

**O que muda:** O arquivo centraliza o registro de todos os routers da API (funcao `buildApiRouter`, linhas 21-31). Sera necessario:
1. Importar `buildWebhookRouter` de `src/modules/webhooks/webhook.routes.ts` e `buildAdminWebhookRouter` de `src/modules/webhooks/webhook.admin.routes.ts`.
2. Registrar `router.use('/webhooks', buildWebhookRouter(controllers.webhooks))` para endpoints de CRUD, rotacao de secret e historico de entregas.
3. Registrar `router.use('/admin/webhooks', buildAdminWebhookRouter(controllers.webhooks))` para endpoints de DLQ (listagem e replay).
4. Estender o tipo `Controllers` (linha 13) para incluir `webhooks: WebhookController`.

### 12.4. `src/middlewares/auth.middleware.ts`

**O que muda:** Nenhuma alteracao e necessaria neste arquivo. O middleware `authenticate` (linha 27) e reutilizado diretamente nos routers de webhook, aplicado com `router.use(authenticate)` da mesma forma que os demais modulos fazem (ex: `src/modules/orders/order.routes.ts`, linha 13). O middleware `requireRole('ADMIN')` (linha 49) e aplicado especificamente nos endpoints admin de DLQ como middleware de rota, seguindo o padrao do projeto. Nao ha necessidade de criar novos middlewares de autorizacao.

### 12.5. `prisma/schema.prisma`

**O que muda:** Adicionar os quatro novos models (`WebhookConfig`, `WebhookOutbox`, `WebhookDelivery`, `WebhookDeadLetter`) seguindo as convencoes existentes: UUID como PK (`@id @default(uuid()) @db.Char(36)`), `@@map()` para nomes snake_case no banco, `@updatedAt` onde aplicavel, indices nos campos de consulta frequente. O modelo `Customer` existente (linha 40) precisa receber a relacao inversa `webhookConfigs WebhookConfig[]` -- isso e uma alteracao puramente declarativa no Prisma que nao gera coluna nova na tabela `customers`. Gerar nova migration com `npx prisma migrate dev`.

**Modelos Prisma sugeridos:**

```prisma
model WebhookConfig {
  id                       String    @id @default(uuid()) @db.Char(36)
  customerId               String    @db.Char(36)
  url                      String    @db.VarChar(2048)
  secret                   String    @db.VarChar(255)
  previousSecret           String?   @db.VarChar(255)
  previousSecretExpiresAt  DateTime?
  events                   Json
  active                   Boolean   @default(true)
  createdAt                DateTime  @default(now())
  updatedAt                DateTime  @updatedAt

  customer    Customer            @relation(fields: [customerId], references: [id])
  outbox      WebhookOutbox[]
  deliveries  WebhookDelivery[]
  deadLetters WebhookDeadLetter[]

  @@unique([customerId, url])
  @@index([customerId, active])
  @@map("webhook_configs")
}

model WebhookOutbox {
  id              String    @id @db.Char(36)
  webhookConfigId String    @db.Char(36)
  eventType       String    @db.VarChar(100)
  payload         Json
  status          String    @default("PENDING") @db.VarChar(20)
  attemptCount    Int       @default(0)
  nextRetryAt     DateTime?
  createdAt       DateTime  @default(now())

  webhookConfig WebhookConfig @relation(fields: [webhookConfigId], references: [id], onDelete: Cascade)

  @@index([status, nextRetryAt])
  @@index([createdAt])
  @@map("webhook_outbox")
}

model WebhookDelivery {
  id              String   @id @default(uuid()) @db.Char(36)
  webhookConfigId String   @db.Char(36)
  outboxId        String   @db.Char(36)
  eventType       String   @db.VarChar(100)
  httpStatus      Int?
  success         Boolean
  durationMs      Int
  attemptNumber   Int
  error           String?  @db.Text
  createdAt       DateTime @default(now())

  webhookConfig WebhookConfig @relation(fields: [webhookConfigId], references: [id], onDelete: Cascade)

  @@index([webhookConfigId, createdAt])
  @@map("webhook_deliveries")
}

model WebhookDeadLetter {
  id              String    @id @default(uuid()) @db.Char(36)
  outboxId        String    @db.Char(36)
  webhookConfigId String    @db.Char(36)
  eventType       String    @db.VarChar(100)
  payload         Json
  lastError       String?   @db.Text
  lastHttpStatus  Int?
  failedAt        DateTime  @default(now())
  replayedAt      DateTime?
  replayedBy      String?   @db.Char(36)

  webhookConfig WebhookConfig @relation(fields: [webhookConfigId], references: [id], onDelete: Cascade)

  @@index([webhookConfigId])
  @@index([failedAt])
  @@map("webhook_dead_letters")
}
```

---

## 13. Estrutura de arquivos do modulo

```
src/
  modules/
    webhooks/
      webhook.controller.ts        # Handlers HTTP (CRUD, rotate-secret, deliveries)
      webhook.admin.controller.ts  # Handlers HTTP admin (DLQ list, replay)
      webhook.service.ts           # Logica de negocio (CRUD, rotacao, historico)
      webhook.repository.ts        # Acesso ao banco (configs, deliveries, dead letters)
      webhook.outbox.ts            # Funcao publishWebhookEvent(tx, ...) e repository da outbox
      webhook.processor.ts         # Logica de processamento: buscar pendentes, enviar, registrar
      webhook.routes.ts            # Rotas CRUD + rotate-secret + deliveries
      webhook.admin.routes.ts      # Rotas admin (DLQ)
      webhook.schemas.ts           # Schemas Zod de validacao de entrada
      webhook.errors.ts            # Classes de erro com prefixo WEBHOOK_
      webhook.crypto.ts            # Funcoes de geracao de secret e assinatura HMAC
  worker.ts                        # Entry-point do worker (polling loop, graceful shutdown)
```

---

## 14. Informacoes em aberto

As seguintes informacoes nao foram definidas na transcricao da reuniao nem podem ser inferidas do repositorio. Ficam registradas como pontos a decidir antes ou durante a implementacao:

1. **Tamanho do batch do worker:** Sugerido 10 neste documento, mas nao foi discutido na reuniao. Deve ser validado com a tech lead.
2. **Formato exato da secret:** Este documento sugere `whsec_` + base64 de 32 bytes aleatorios. O formato nao foi discutido na reuniao.
3. **Comportamento quando webhook esta `active = false`:** Este documento assume que webhooks inativos nao geram eventos na outbox. Nao foi explicitamente discutido.
4. **Politica de retencao da tabela `webhook_deliveries`:** O historico de entregas pode crescer indefinidamente. Nenhuma politica de cleanup foi definida.
5. **Endpoint para obter dados de uma configuracao individual de webhook (`GET /api/v1/webhooks/:id`):** Nao foi mencionado na reuniao. A listagem filtra por `customerId`, mas acesso direto por `id` pode ser util. Recomenda-se incluir por consistencia com os demais modulos (todos possuem `GET /:id`).
6. **Limite de webhooks por customer:** Nao foi discutido. Pode-se implementar sem limite inicialmente e adicionar restricao se necessario.
