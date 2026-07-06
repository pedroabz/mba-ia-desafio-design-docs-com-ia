# Tracker de Rastreabilidade

**Feature:** Sistema de Webhooks de Notificação de Pedidos
**Data:** 2026-07-05
**Versão:** 1.4

---

## Sobre este documento

Este tracker mapeia cada decisão, alternativa considerada, restrição e trade-off registrados nos ADRs (`docs/adrs/ADR-001` a `ADR-006`), no RFC (`docs/RFC.md`), no FDD (`docs/FDD.md`) e no PRD (`docs/PRD.md`) à sua origem na transcrição da reunião (`TRANSCRICAO.md`), no código-fonte do repositório ou nos próprios documentos referenciados.

**Convenções de identificação:**

- `ADR-XXX` -- decisão principal do ADR
- `ADR-XXX-DET-NN` -- detalhamento da decisão
- `ADR-XXX-ALT-NN` -- alternativa considerada
- `ADR-XXX-RST-NN` -- restrição ou limitação conhecida
- `ADR-XXX-TRD-NN` -- trade-off explícito
- `RFC-CTX-NN` -- contexto e problema descrito no RFC
- `RFC-PROP-NN` -- proposta técnica do RFC
- `RFC-ALT-NN` -- alternativa considerada no RFC
- `RFC-QA-NN` -- questão em aberto registrada no RFC
- `RFC-IMP-NN` -- impacto identificado no RFC
- `RFC-RISCO-NN` -- risco identificado no RFC
- `RFC-REF-NN` -- referência cruzada do RFC a outros documentos
- `FDD-FR-NN` -- requisito funcional detalhado no FDD (endpoint, CRUD)
- `FDD-RNF-NN` -- requisito não funcional detalhado no FDD
- `FDD-CONTRATO-NN` -- contrato de API detalhado no FDD
- `FDD-ERRO-NN` -- código de erro especificado no FDD
- `FDD-FLUXO-NN` -- fluxo interno detalhado no FDD
- `FDD-INT-NN` -- ponto de integração com o sistema existente detalhado no FDD
- `FDD-IMPL-NN` -- decisão de implementação detalhada no FDD
- `PRD-FR-NN` -- requisito funcional descrito no PRD
- `PRD-RNF-NN` -- requisito não funcional descrito no PRD
- `PRD-ESCOPO-NN` -- item fora de escopo registrado no PRD
- `PRD-RISCO-NN` -- risco identificado no PRD
- `PRD-DECISAO-NN` -- decisão de produto registrada no PRD
- `PRD-DEP-NN` -- dependência identificada no PRD

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|--------------------|--------|-------------|
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Adotar padrão Transactional Outbox no MySQL existente, inserindo evento na tabela `webhook_outbox` dentro da mesma transação que muda o status do pedido | TRANSCRICAO | [09:06] Diego, [09:07] Diego, [09:08] Larissa |
| ADR-001-DET-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Payload do evento renderizado como snapshot no momento da inserção (não montado posteriormente a partir do `order_id`) | TRANSCRICAO | [09:52] Larissa, [09:52] Diego, [09:52] Bruno |
| ADR-001-DET-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Identificadores da tabela outbox em UUID, seguindo padrão do projeto | TRANSCRICAO | [09:51] Larissa, [09:51] Diego |
| ADR-001-DET-03 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Filtragem de eventos na inserção da outbox: se nenhum webhook configurado deseja aquele status, a linha não é inserida | TRANSCRICAO | [09:34] Bruno, [09:34] Diego |
| ADR-001-DET-04 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Ponto de integração no método `changeStatus` dentro do bloco `$transaction` existente, utilizando o mesmo `tx` | TRANSCRICAO | [09:41] Bruno, [09:41] Diego |
| ADR-001-DET-04 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | (Validação no código) Método `changeStatus` usa `this.prisma.$transaction(async (tx) => { ... })` nas linhas 131-178 | CODIGO | `src/modules/orders/order.service.ts` (linhas 126-179) |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa | Despacho síncrono dentro do service de orders -- rejeitada por risco de travamento e acoplamento à disponibilidade de sistemas externos | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa | Fila externa (Redis Streams ou similar) -- rejeitada por ser overengineering para o tamanho do time | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| ADR-001-RST-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Linhas processadas acumulam na tabela outbox; necessidade de arquivamento/limpeza periódica (sugestão de 30 dias, fora do escopo desta feature) | TRANSCRICAO | [09:08] Diego |
| ADR-001-TRD-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Aceita-se carga adicional no MySQL em troca de eliminar infraestrutura externa e garantir consistência transacional forte | TRANSCRICAO | [09:07] Diego |
| ADR-002 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | 5 tentativas com backoff exponencial (1m/5m/30m/2h/12h) e DLQ em tabela separada no MySQL | TRANSCRICAO | [09:15] Diego, [09:17] Diego, [09:17] Larissa |
| ADR-002-DET-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | DLQ em tabela separada (`webhook_dead_letter`) com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-002-DET-02 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | Reprocessamento manual via endpoint admin `POST /admin/webhooks/dead-letter/:id/replay`, exigindo role ADMIN com auditoria | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Alternativa | 3 tentativas (proposta do Bruno) -- rejeitada por cobrir janela insuficiente para manutenções planejadas de até 2 horas | TRANSCRICAO | [09:16] Bruno, [09:16] Diego |
| ADR-002-ALT-02 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Alternativa | Retry indefinido com backoff -- rejeitado porque eventos ficariam pendurados indefinidamente | TRANSCRICAO | [09:15] Diego |
| ADR-002-ALT-03 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Alternativa | DLQ na própria tabela outbox (marcação de status "failed") -- rejeitada por poluir a tabela principal | TRANSCRICAO | [09:17] Larissa |
| ADR-002-TRD-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Trade-off | Aceita-se complexidade de gerenciamento de retries e DLQ em troca de maximizar chance de entrega dentro de janela temporal realista (~15h) | TRANSCRICAO | [09:17] Marcos, [09:15] Diego |
| ADR-003 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | HMAC-SHA256 sobre o body do request, com secret única por endpoint e assinatura enviada no header `X-Signature` | TRANSCRICAO | [09:20] Sofia |
| ADR-003-DET-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Secret única por endpoint de webhook (não global); se vazar uma, não compromete os demais | TRANSCRICAO | [09:21] Sofia |
| ADR-003-DET-02 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Rotação de secrets via API com grace period de 24 horas, durante o qual ambas as secrets são válidas | TRANSCRICAO | [09:21] Sofia, [09:22] Sofia |
| ADR-003-DET-03 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | TLS obrigatório: URLs com esquema HTTP rejeitadas com erro de validação no cadastro | TRANSCRICAO | [09:23] Sofia |
| ADR-003-DET-04 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | (Contexto) Vazamento de secret em log de aplicação de cliente já foi observado anteriormente | TRANSCRICAO | [09:22] Diego |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Alternativa | Secret global da plataforma -- rejeitada porque vazamento de uma comprometeria todos os clientes | TRANSCRICAO | [09:21] Sofia |
| ADR-003-ALT-02 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Alternativa | JWT assinado por request -- rejeitada por complexidade desnecessária; HMAC-SHA256 é padrão de mercado (Stripe, GitHub) | TRANSCRICAO | [09:20] Sofia |
| ADR-003-TRD-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Trade-off | Aceita-se complexidade de gerenciar secrets individuais e grace period em troca de isolamento de segurança granular e aderência ao padrão de mercado | TRANSCRICAO | [09:21] Sofia, [09:22] Sofia |
| ADR-004 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Garantia at-least-once com mecanismo de deduplicação via header `X-Event-Id` (UUID único por evento) | TRANSCRICAO | [09:24] Diego, [09:25] Diego, [09:26] Larissa |
| ADR-004-DET-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Responsabilidade de deduplicação transferida ao cliente | TRANSCRICAO | [09:25] Sofia, [09:25] Diego |
| ADR-004-DET-02 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Headers enviados: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id`, `Content-Type` | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| ADR-004-DET-03 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Marcos documentará a garantia at-least-once de forma destacada no portal de desenvolvedor | TRANSCRICAO | [09:26] Marcos |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Alternativa | Garantia exactly-once -- rejeitada por exigir coordenação bidirecional e complexidade desproporcional; nenhum sistema de referência (Stripe, GitHub) oferece | TRANSCRICAO | [09:25] Diego |
| ADR-004-ALT-02 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Alternativa | At-least-once sem mecanismo de deduplicação -- rejeitada por não fornecer ferramenta ao cliente para diferenciar duplicatas | TRANSCRICAO | [09:25] Diego |
| ADR-004-TRD-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Trade-off | Aceita-se transferir responsabilidade de deduplicação ao cliente em troca de simplicidade no servidor e aderência ao padrão de mercado | TRANSCRICAO | [09:25] Diego, [09:26] Marcos |
| ADR-005 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Worker de despacho em processo Node.js separado com polling a cada 2 segundos na tabela outbox | TRANSCRICAO | [09:09] Diego, [09:10] Larissa, [09:11] Diego |
| ADR-005-DET-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Worker com entry-point próprio (analogamente a `src/server.ts`), iniciado por script dedicado (ex.: `npm run worker`) | TRANSCRICAO | [09:11] Larissa |
| ADR-005-DET-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | (Validação no código) Entry-point existente `src/server.ts` com bootstrap, Prisma Client e graceful shutdown confirmado | CODIGO | `src/server.ts` |
| ADR-005-DET-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | PrismaClient próprio do worker (instância separada, mesmo banco) | TRANSCRICAO | [09:30] Bruno |
| ADR-005-DET-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | (Validação no código) Singleton de PrismaClient em `src/config/database.ts` -- worker deverá instanciar o próprio | CODIGO | `src/config/database.ts` |
| ADR-005-DET-03 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Intervalo de polling de 2 segundos atende requisito de notificação em menos de 10 segundos | TRANSCRICAO | [09:02] Marcos, [09:09] Diego, [09:10] Marcos |
| ADR-005-RST-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Restrição | Garantia de ordenação válida apenas com single-worker; escalar para múltiplos workers exigirá particionamento por `order_id` ou lock pessimista | TRANSCRICAO | [09:12] Diego, [09:13] Diego, [09:13] Larissa |
| ADR-005-RST-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Restrição | Latência mínima de até 2 segundos entre inserção na outbox e detecção pelo worker | TRANSCRICAO | [09:10] Larissa |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Alternativa | Trigger do MySQL para notificar o worker -- rejeitada porque MySQL não possui `NOTIFY/LISTEN` nativo e improvisar soluções seria frágil | TRANSCRICAO | [09:09] Bruno, [09:09] Diego |
| ADR-005-ALT-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Alternativa | Worker dentro do processo da API (setInterval no mesmo processo) -- rejeitada por acoplamento de ciclo de vida e mistura de responsabilidades | TRANSCRICAO | [09:11] Diego |
| ADR-005-TRD-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Trade-off | Aceita-se latência de até 2s e custo de polling constante em troca de simplicidade, isolamento de processos e reutilização de infra existente | TRANSCRICAO | [09:09] Diego, [09:10] Larissa |
| ADR-006 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Módulo de webhooks reutilizará integralmente os padrões existentes do projeto sem introduzir frameworks ou convenções novas | TRANSCRICAO | [09:27] Bruno, [09:29] Bruno, [09:30] Larissa |
| ADR-006-DET-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Estrutura do módulo em `src/modules/webhooks/` com controller, service, repository, routes e schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-DET-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | (Validação no código) Padrão de módulos confirmado nos diretórios existentes: `auth`, `users`, `customers`, `products`, `orders` | CODIGO | `src/modules/*/` |
| ADR-006-DET-02 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Erros com prefixo `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) estendendo `AppError` | TRANSCRICAO | [09:28] Bruno, [09:29] Larissa |
| ADR-006-DET-02 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | (Validação no código) Classe `AppError` com `statusCode`, `errorCode`, `details`; classes especializadas (`NotFoundError`, `ConflictError`, `InvalidStatusTransitionError`, `InsufficientStockError`) | CODIGO | `src/shared/errors/app-error.ts`, `src/shared/errors/http-errors.ts` |
| ADR-006-DET-03 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Middleware de erro centralizado já trata `AppError`, `ZodError` e `PrismaClientKnownRequestError` sem necessidade de alteração | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DET-03 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | (Validação no código) Middleware de erro confirmado tratando as três categorias de erro | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-DET-04 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Logger Pino reutilizado em todo o módulo, incluindo o worker; secret do webhook deve ser adicionada aos paths de redação | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DET-04 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | (Validação no código) Logger Pino com redação automática de campos sensíveis (`authorization`, `password`, `token`) | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-DET-05 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Endpoint de replay de DLQ exige role `ADMIN` via `requireRole('ADMIN')` já disponível | TRANSCRICAO | [09:36] Larissa, [09:36] Sofia |
| ADR-006-DET-05 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | (Validação no código) Função `requireRole` com suporte a roles `ADMIN` e `OPERATOR` | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-DET-06 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | IDs UUID v4 para todas as entidades, conforme padrão `@default(uuid())` no schema Prisma | TRANSCRICAO | [09:51] Larissa |
| ADR-006-DET-06 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | (Validação no código) Todas as entidades usam `@id @default(uuid()) @db.Char(36)` | CODIGO | `prisma/schema.prisma` |
| ADR-006-ALT-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Alternativa | Introduzir framework ou biblioteca nova para o módulo de webhooks -- rejeitada por inconsistência na codebase e aumento de superfície de dependências | TRANSCRICAO | [09:29] Bruno |
| ADR-006-TRD-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Trade-off | Aceita-se acoplamento aos padrões existentes em troca de consistência total na codebase, menor curva de aprendizado e reutilização de infra testada | TRANSCRICAO | [09:27] Bruno, [09:30] Larissa |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram notificações em tempo real; hoje fazem polling em `GET /orders` | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Contexto | Requisito de negócio: notificação em menos de 10 segundos após transição de status | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Contexto | Fluxo exclusivamente outbound: sistema envia notificações para clientes, sem receber webhooks de volta | TRANSCRICAO | [09:02] Sofia, [09:02] Marcos |
| RFC-CTX-04 | `docs/RFC.md` | Contexto | Transação de `changeStatus` já é pesada: atualiza status, insere em `order_status_history`, manipula estoque; acoplamento síncrono comprometeria disponibilidade | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-04 | `docs/RFC.md` | Contexto | (Validação no código) Método `changeStatus` com `$transaction` contendo update de status, inserção em `orderStatusHistory` e chamadas a `debitStock`/`replenishStock` | CODIGO | `src/modules/orders/order.service.ts` (linhas 126-179) |
| RFC-PROP-01 | `docs/RFC.md` | Proposta | Pilar 1: Transactional Outbox no MySQL existente -- inserção do evento na mesma transação que muda o status, payload como snapshot, filtragem na inserção | TRANSCRICAO | [09:06] Diego, [09:07] Diego, [09:34] Bruno |
| RFC-PROP-01 | `docs/RFC.md` | Proposta | Referencia ADR-001 para detalhamento completo da decisão | ADR | `docs/adrs/ADR-001-outbox-no-mysql.md` |
| RFC-PROP-02 | `docs/RFC.md` | Proposta | Pilar 2: Worker de despacho em processo Node.js separado com polling a cada 2 segundos; ordenação garantida com single-worker (limitação conhecida) | TRANSCRICAO | [09:09] Diego, [09:10] Larissa, [09:11] Diego, [09:12] Diego |
| RFC-PROP-02 | `docs/RFC.md` | Proposta | Referencia ADR-005 para detalhamento completo da decisão | ADR | `docs/adrs/ADR-005-worker-polling-separado.md` |
| RFC-PROP-03 | `docs/RFC.md` | Proposta | Pilar 3: Autenticação HMAC-SHA256 com secret única por endpoint, rotação com grace period de 24h, TLS obrigatório | TRANSCRICAO | [09:20] Sofia, [09:21] Sofia, [09:22] Sofia, [09:23] Sofia |
| RFC-PROP-03 | `docs/RFC.md` | Proposta | Referencia ADR-003 para detalhamento completo da decisão | ADR | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` |
| RFC-PROP-04 | `docs/RFC.md` | Proposta | Pilar 4: Garantia at-least-once com retry (5 tentativas, backoff 1m/5m/30m/2h/12h), DLQ em tabela separada, reprocessamento manual via endpoint ADMIN | TRANSCRICAO | [09:15] Diego, [09:17] Diego, [09:18] Diego |
| RFC-PROP-04 | `docs/RFC.md` | Proposta | Referencia ADR-002 e ADR-004 para detalhamento completo das decisões | ADR | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md`, `docs/adrs/ADR-004-garantia-at-least-once.md` |
| RFC-PROP-05 | `docs/RFC.md` | Proposta | Integração com arquitetura existente: módulo em `src/modules/webhooks/`, `AppError` com prefixo `WEBHOOK_`, logger Pino, Zod, middlewares existentes; nenhuma biblioteca nova | TRANSCRICAO | [09:27] Bruno, [09:29] Bruno, [09:30] Larissa |
| RFC-PROP-05 | `docs/RFC.md` | Proposta | Referencia ADR-006 para detalhamento completo da decisão | ADR | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa | Despacho síncrono dentro do service de pedidos -- descartada por risco de travamento da transação e acoplamento à disponibilidade de sistemas externos | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa | Fila externa com Redis Streams ou broker de mensagens -- descartada por exigir infraestrutura adicional desproporcional ao tamanho do time | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| RFC-ALT-03 | `docs/RFC.md` | Alternativa | Trigger do MySQL para notificação reativa -- descartada porque MySQL não possui mecanismo nativo de notificação a processos externos | TRANSCRICAO | [09:09] Bruno, [09:09] Diego |
| RFC-QA-01 | `docs/RFC.md` | Questão em aberto | Rate limiting de envio para clientes: decidiu-se observar comportamento em produção antes de implementar throttling | TRANSCRICAO | [09:38] Diego, [09:39] Diego, [09:39] Larissa |
| RFC-QA-02 | `docs/RFC.md` | Questão em aberto | Notificação por email em caso de falhas recorrentes: explicitamente fora do escopo desta fase | TRANSCRICAO | [09:37] Marcos, [09:37] Larissa |
| RFC-QA-03 | `docs/RFC.md` | Questão em aberto | Estratégia de limpeza da tabela outbox: arquivamento após 30 dias mencionado mas marcado como fora do escopo | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | `docs/RFC.md` | Questão em aberto | Escalabilidade para múltiplos workers: necessitaria particionamento por `order_id` ou lock pessimista; sem plano definido | TRANSCRICAO | [09:13] Diego, [09:13] Larissa |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | Ponto de integração crítico: método `changeStatus` do `OrderService` será modificado para inserir eventos na outbox dentro da transação existente; única alteração em código de domínio existente | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| RFC-IMP-01 | `docs/RFC.md` | Impacto | (Validação no código) Método `changeStatus` usa `this.prisma.$transaction(async (tx) => { ... })` com update, history e estoque na mesma transação | CODIGO | `src/modules/orders/order.service.ts` (linhas 131-178) |
| RFC-IMP-02 | `docs/RFC.md` | Impacto | Carga adicional no banco: cada mudança de status com webhooks configurados gera uma escrita extra na mesma transação | TRANSCRICAO | [09:07] Bruno, [09:08] Diego |
| RFC-IMP-03 | `docs/RFC.md` | Impacto | Novo processo em produção: worker de despacho como processo Node.js adicional que precisa ser gerenciado | TRANSCRICAO | [09:11] Diego, [09:11] Larissa |
| RFC-RISCO-01 | `docs/RFC.md` | Risco | Acúmulo de eventos na outbox por falha do worker -- mitigação: monitoramento do tamanho da fila com alertas | TRANSCRICAO | [09:07] Bruno, [09:08] Diego |
| RFC-RISCO-02 | `docs/RFC.md` | Risco | Cliente com endpoint permanentemente indisponível gerando carga de retry -- mitigação: política de 5 tentativas com DLQ (~15h) | TRANSCRICAO | [09:15] Diego, [09:16] Diego |
| RFC-RISCO-03 | `docs/RFC.md` | Risco | Vazamento de secret de webhook -- mitigação: secret por endpoint para isolamento de blast radius e rotação via API com grace period | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| RFC-RISCO-04 | `docs/RFC.md` | Risco | Transação de mudança de status mais lenta devido à escrita na outbox -- mitigação: escrita simples de uma linha com índice otimizado | TRANSCRICAO | [09:08] Diego |
| RFC-REF-01 | `docs/RFC.md` | Referência | RFC referencia ADR-001 a ADR-006 como decisões arquiteturais que respaldam a proposta | RFC | `docs/RFC.md` seção "Decisões relacionadas" |
| FDD-FR-01 | `docs/FDD.md` | Requisito funcional | `POST /webhooks` — criar configuração de webhook com secret gerada pelo sistema, URL HTTPS obrigatória e lista de eventos filtráveis | TRANSCRICAO | [09:31] Marcos, [09:33] Bruno |
| FDD-FR-02 | `docs/FDD.md` | Requisito funcional | `GET /webhooks` — listar configurações de webhook, filtrável por `customerId`, com paginação; secret nunca incluída na resposta | TRANSCRICAO | [09:33] Bruno |
| FDD-FR-03 | `docs/FDD.md` | Requisito funcional | `PATCH /webhooks/:id` — atualizar URL, lista de eventos e estado ativo/inativo de um webhook | TRANSCRICAO | [09:33] Bruno |
| FDD-FR-04 | `docs/FDD.md` | Requisito funcional | `DELETE /webhooks/:id` — remover configuração de webhook; retorna 204 sem body | TRANSCRICAO | [09:33] Bruno |
| FDD-FR-05 | `docs/FDD.md` | Requisito funcional | `POST /webhooks/:id/rotate-secret` — rotacionar secret com grace period de 24 horas para a secret anterior | TRANSCRICAO | [09:21] Sofia, [09:22] Sofia |
| FDD-FR-06 | `docs/FDD.md` | Requisito funcional | `GET /webhooks/:id/deliveries` — histórico de tentativas de entrega com status HTTP, duração, payload e response body | TRANSCRICAO | [09:34] Marcos |
| FDD-FR-07 | `docs/FDD.md` | Requisito funcional | `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar evento da DLQ, exige role ADMIN, registra auditoria com `user_id` | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| FDD-FR-08 | `docs/FDD.md` | Requisito funcional | Filtragem de eventos na inserção da outbox: se nenhum webhook configurado deseja o status de destino, nenhuma linha é inserida | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno, [09:34] Diego |
| FDD-FR-09 | `docs/FDD.md` | Requisito funcional | Idempotência: `X-Event-Id` permanece o mesmo UUID em todas as tentativas de entrega do mesmo evento | TRANSCRICAO | [09:25] Diego |
| FDD-FR-10 | `docs/FDD.md` | Requisito funcional | HMAC-SHA256: header `X-Signature` contém `sha256=<hex>` calculado sobre o body cru com a secret do endpoint | TRANSCRICAO | [09:20] Sofia |
| FDD-RNF-01 | `docs/FDD.md` | Requisito não funcional | Timeout HTTP de 10 segundos para chamadas do worker ao endpoint do cliente; acima disso, tratado como falha | TRANSCRICAO | [09:42] Diego, [09:42] Sofia |
| FDD-RNF-02 | `docs/FDD.md` | Requisito não funcional | Payload máximo de 64 KB; eventos que excedam esse limite são rejeitados na inserção na outbox com log de erro | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| FDD-RNF-03 | `docs/FDD.md` | Requisito não funcional | TLS obrigatório: URLs com esquema HTTP são rejeitadas com erro `WEBHOOK_INVALID_URL` no cadastro | TRANSCRICAO | [09:23] Sofia |
| FDD-RNF-04 | `docs/FDD.md` | Requisito não funcional | Secret do webhook nunca aparece em logs (Pino redact) nem em endpoints de listagem (GET) | TRANSCRICAO | [09:29] Bruno; FDD | seção 10 critérios 14 e 15 |
| FDD-RNF-05 | `docs/FDD.md` | Requisito não funcional | Isolamento de processos: crash ou restart da API não afeta o worker, e vice-versa | TRANSCRICAO | [09:11] Diego |
| FDD-RNF-06 | `docs/FDD.md` | Requisito não funcional | Nenhuma dependência nova obrigatória; HMAC via `node:crypto` nativo, HTTP client via `node:https` ou `undici` bundled | FDD | seção 9 "Dependências e compatibilidade" |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | Payload do webhook: JSON com `event_id`, `event_type` ("order.status_changed"), `timestamp` ISO 8601, e `data` contendo `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`; sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | Headers enviados pelo worker: `Content-Type`, `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | Validação Zod para criação: `customerId` uuid obrigatório, `url` string URL HTTPS obrigatória, `events` array de OrderStatus com min 1 | FDD | seção 5.1 |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | Validação Zod para atualização: `url` opcional (HTTPS se presente), `events` opcional (min 1 se presente), `active` boolean opcional | FDD | seção 5.3 |
| FDD-CONTRATO-05 | `docs/FDD.md` | Contrato | Paginação em listagem e histórico de entregas: `page` (default 1), `pageSize` (default 20, max 100) | FDD | seções 5.2 e 5.6 |
| FDD-CONTRATO-06 | `docs/FDD.md` | Contrato | Resposta do replay (200): `message`, `outboxEventId`, `originalEventId`, `replayedBy`, `replayedAt` | FDD | seção 5.7 |
| FDD-CONTRATO-07 | `docs/FDD.md` | Contrato | Secret retornada apenas na criação (`POST /webhooks`) e na rotação (`POST /webhooks/:id/rotate-secret`); nunca em GET ou PATCH | FDD | seções 5.1 e 5.2 |
| FDD-ERRO-01 | `docs/FDD.md` | Código de erro | `WEBHOOK_NOT_FOUND` (404) — webhook config não encontrado pelo id | FDD | seção 6 "Matriz de erros" |
| FDD-ERRO-02 | `docs/FDD.md` | Código de erro | `WEBHOOK_INVALID_URL` (400) — URL não começa com `https://` | TRANSCRICAO | [09:23] Sofia; FDD | seção 6 |
| FDD-ERRO-03 | `docs/FDD.md` | Código de erro | `WEBHOOK_DUPLICATE_URL` (409) — já existe webhook ativo com mesma URL para o mesmo customer | FDD | seção 6 "Matriz de erros" |
| FDD-ERRO-04 | `docs/FDD.md` | Código de erro | `WEBHOOK_SECRET_REQUIRED` (400) — tentativa de operação sem secret configurada (erro interno) | TRANSCRICAO | [09:28] Bruno; FDD | seção 6 |
| FDD-ERRO-05 | `docs/FDD.md` | Código de erro | `WEBHOOK_INACTIVE` (422) — operação sobre webhook desativado | FDD | seção 6 "Matriz de erros" |
| FDD-ERRO-06 | `docs/FDD.md` | Código de erro | `WEBHOOK_DEAD_LETTER_NOT_FOUND` (404) — registro não encontrado na tabela de dead letter | FDD | seção 6 "Matriz de erros" |
| FDD-ERRO-07 | `docs/FDD.md` | Código de erro | `WEBHOOK_PAYLOAD_TOO_LARGE` (422) — payload renderizado excede 64 KB | TRANSCRICAO | [09:24] Diego; FDD | seção 6 |
| FDD-ERRO-08 | `docs/FDD.md` | Código de erro | `WEBHOOK_DELIVERY_FAILED` (interno/log) — falha na entrega HTTP, não exposto via API, apenas registrado em log | FDD | seção 6 "Matriz de erros" |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Inserção na outbox: dentro do `$transaction` do `changeStatus`, chama `publishWebhookEvent(tx, ...)` que busca configs ativas, filtra por status e insere snapshot na outbox | TRANSCRICAO | [09:06] Diego, [09:41] Bruno; FDD | seção 4.1 |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | Worker polling: bootstrap com PrismaClient próprio, loop a cada 2s, SELECT pendentes ORDER BY `created_at` ASC, marca PROCESSING, dispara HTTP POST, registra resultado no delivery log | TRANSCRICAO | [09:09] Diego, [09:10] Larissa; FDD | seção 4.2 |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Retry e backoff: intervalos fixos 1m/5m/30m/2h/12h definidos como constante `RETRY_INTERVALS_MS`; campo `next_retry_at` controla elegibilidade para nova tentativa | TRANSCRICAO | [09:17] Diego; FDD | seção 4.3 |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | DLQ e replay: após 5a falha, move para `webhook_dead_letter` e remove da outbox; replay via endpoint admin cria nova entrada na outbox com `attempt_count = 0` e remove da DLQ | TRANSCRICAO | [09:18] Diego; FDD | seção 4.4 |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | Tratamento de respostas HTTP: 2xx marca DELIVERED; 4xx (exceto 429), 5xx, timeout e erro de rede marcam falha e acionam retry; tratamento de 429 com Retry-After é ponto em aberto | FDD | seção 7 "Tratamento de respostas HTTP" |
| FDD-FLUXO-06 | `docs/FDD.md` | Fluxo | Lock otimista: status PROCESSING serve como lock; eventos em PROCESSING há mais de 5 minutos (configurável) são revertidos para PENDING no início de cada ciclo de polling | FDD | seção 7 "Lock otimista no worker" |
| FDD-INT-01 | `docs/FDD.md` | Integração | `src/modules/orders/order.service.ts` — extensão do `changeStatus` (linhas 126-179) para incluir chamada a `publishWebhookEvent(tx, ...)` após inserção no `orderStatusHistory` e antes da query de refresh | TRANSCRICAO | [09:40] Bruno, [09:41] Diego; CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | `src/shared/errors/` — novos erros em `src/modules/webhooks/webhook.errors.ts` estendendo `NotFoundError`, `BadRequestError`, `ConflictError` existentes | TRANSCRICAO | [09:28] Bruno; CODIGO | `src/shared/errors/http-errors.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | `src/middlewares/error.middleware.ts` — sem alteração necessária; já trata qualquer `AppError` automaticamente | TRANSCRICAO | [09:29] Bruno; CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | `src/middlewares/auth.middleware.ts` — reutilização de `authenticate` e `requireRole('ADMIN')` para endpoint de replay da DLQ | TRANSCRICAO | [09:36] Sofia, [09:36] Larissa; CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | `src/shared/logger/index.ts` — reutilização do Pino existente; worker usa `createLogger()` com `service: 'webhook-worker'`; adição de paths de redação `*.secret` e `*.previousSecret` | TRANSCRICAO | [09:29] Bruno; CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | `src/worker.ts` — novo entry-point análogo a `src/server.ts`, com bootstrap assíncrono, PrismaClient próprio e graceful shutdown SIGINT/SIGTERM; novo script `npm run worker` | TRANSCRICAO | [09:11] Larissa; CODIGO | `src/server.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | `prisma/schema.prisma` — quatro novos models: `WebhookConfig`, `WebhookOutbox`, `WebhookDeliveryLog`, `WebhookDeadLetter`; relação `Customer -> WebhookConfig[]` adicionada ao model Customer | FDD | seção 12.7; CODIGO | `prisma/schema.prisma` |
| FDD-INT-08 | `docs/FDD.md` | Integração | `src/routes/index.ts` — registro do módulo com `router.use('/webhooks', ...)` e `router.use('/admin/webhooks', ...)` | FDD | seção 12.8 |
| FDD-INT-09 | `docs/FDD.md` | Integração | `src/app.ts` — wiring de dependências seguindo padrão de `buildControllers`: `WebhookRepository`, `WebhookService`, `WebhookController` | FDD | seção 12.9 |
| FDD-IMPL-01 | `docs/FDD.md` | Decisão de implementação | Função `publishWebhookEvent` recebe `tx` (tipo `TxClient` já definido na linha 24 do `order.service.ts`) — `OrderService` não precisa conhecer o `WebhookRepository` | TRANSCRICAO | [09:41] Bruno, [09:41] Diego; FDD | seção 12.1 |
| FDD-IMPL-02 | `docs/FDD.md` | Decisão de implementação | Estrutura de arquivos do módulo: `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.admin.routes.ts`, `webhook.schemas.ts`, `webhook.errors.ts`, `webhook.processor.ts`, `webhook.signer.ts` | TRANSCRICAO | [09:27] Bruno; FDD | seção "Estrutura de arquivos" |
| FDD-IMPL-03 | `docs/FDD.md` | Decisão de implementação | Intervalos de retry definidos como constante `RETRY_INTERVALS_MS` com valores fixos em milissegundos, não calculados exponencialmente em runtime | FDD | seção 4.3 |
| FDD-IMPL-04 | `docs/FDD.md` | Decisão de implementação | HMAC-SHA256 e geração de secrets encapsulados em `webhook.signer.ts`; geração via `crypto.randomBytes` do `node:crypto` nativo | FDD | seções 9 e "Estrutura de arquivos" |
| FDD-IMPL-05 | `docs/FDD.md` | Decisão de implementação | Timeout HTTP implementado via `AbortController` com `setTimeout` no `fetch`/`undici` | FDD | seção 7 "Timeout de entrega HTTP" |
| FDD-IMPL-06 | `docs/FDD.md` | Decisão de implementação | Observabilidade via logs estruturados Pino (sem framework de métricas); 9 métricas mapeadas como pontos de instrumentação para futura extração por log aggregation | FDD | seção 8 "Observabilidade" |
| FDD-IMPL-07 | `docs/FDD.md` | Decisão de implementação | `event_id` (UUID) serve como correlation ID natural entre inserção na outbox, tentativas de entrega e delivery log; todos os logs incluem `eventId` | FDD | seção 8 "Tracing" |
| FDD-IMPL-08 | `docs/FDD.md` | Decisão de implementação (ponto em aberto) | Tempo de lock otimista (5 minutos sugerido para reverter PROCESSING para PENDING) não foi discutido na reunião; valor deve ser validado pela equipe | FDD | seção 7 "Lock otimista no worker" |
| FDD-IMPL-09 | `docs/FDD.md` | Decisão de implementação (ponto em aberto) | Tratamento diferenciado de respostas 4xx vs 5xx em relação a retries não foi discutido na reunião; implementação inicial tratará qualquer non-2xx como falha retentável | FDD | seção 7 "Tratamento de respostas HTTP" |
| FDD-IMPL-10 | `docs/FDD.md` | Decisão de implementação (ponto em aberto) | Integração com ferramentas de tracing distribuído (OpenTelemetry, Datadog) não foi discutida na reunião; fica como evolução futura | FDD | seção 8 "Tracing" |
| PRD-FR-01 | `docs/PRD.md` | Requisito funcional | Cadastrar configuração de webhook com URL de destino e eventos de interesse; credencial de segurança gerada pelo sistema e devolvida na criação | TRANSCRICAO | [09:31] Marcos, [09:33] Bruno |
| PRD-FR-02 | `docs/PRD.md` | Requisito funcional | Editar configuração de webhook existente: URL, eventos de interesse e estado ativo/inativo | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-03 | `docs/PRD.md` | Requisito funcional | Remover configuração de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-04 | `docs/PRD.md` | Requisito funcional | Listar configurações de webhook de um cliente; credencial de segurança nunca exibida na listagem | TRANSCRICAO | [09:33] Bruno |
| PRD-FR-05 | `docs/PRD.md` | Requisito funcional | Envio automático de notificação quando o status de um pedido muda, para webhooks ativos cujos eventos de interesse incluam o novo status; se nenhum webhook configurado, nenhuma notificação é gerada | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno, [09:34] Diego |
| PRD-FR-05 | `docs/PRD.md` | Requisito funcional | (Validação no código) Máquina de estados do pedido com transições válidas definidas em `canTransition`; seis status: PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED | CODIGO | `src/modules/orders/order.status.ts` (linhas 1-10) |
| PRD-FR-06 | `docs/PRD.md` | Requisito funcional | Registro indissociável da notificação junto com a mudança de status: se o status mudou, a notificação foi registrada; se houve falha, nenhuma notificação é gerada; não pode haver cenário em que um ocorre sem o outro | TRANSCRICAO | [09:06] Diego, [09:40] Bruno, [09:41] Diego |
| PRD-FR-06 | `docs/PRD.md` | Requisito funcional | (Validação no código) Método `changeStatus` usa `this.prisma.$transaction(async (tx) => { ... })` na mesma transação que atualiza status e insere histórico; ponto de integração para inserção na outbox | CODIGO | `src/modules/orders/order.service.ts` (linhas 131-178) |
| PRD-FR-07 | `docs/PRD.md` | Requisito funcional | Conteúdo da notificação: identificador único do evento, tipo do evento, data/hora, identificador e número do pedido, status anterior e novo, identificador do cliente, valor total; itens do pedido não incluídos | TRANSCRICAO | [09:43] Diego |
| PRD-FR-08 | `docs/PRD.md` | Requisito funcional | Cada notificação assinada com credencial exclusiva do destino; cliente valida autenticidade e integridade | TRANSCRICAO | [09:20] Sofia, [09:21] Sofia |
| PRD-FR-09 | `docs/PRD.md` | Requisito funcional | Rotação de credencial de segurança via API; credencial anterior permanece válida por 24 horas para migração sem interrupção | TRANSCRICAO | [09:21] Sofia, [09:22] Sofia |
| PRD-FR-10 | `docs/PRD.md` | Requisito funcional | URL de destino obrigatoriamente HTTPS; tentativas de cadastro com URL insegura recusadas | TRANSCRICAO | [09:23] Sofia |
| PRD-FR-11 | `docs/PRD.md` | Requisito funcional | Reenvio automático em caso de falha com intervalos crescentes, até 5 tentativas ao longo de ~15 horas; após esgotamento, notificação movida para área de falhas permanentes | TRANSCRICAO | [09:15] Diego, [09:17] Diego |
| PRD-FR-12 | `docs/PRD.md` | Requisito funcional | Reprocessamento manual de notificação com falha permanente por administrador, com registro de auditoria | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| PRD-FR-12 | `docs/PRD.md` | Requisito funcional | (Validação no código) Função `requireRole` já disponível com suporte a role `ADMIN`, reutilizada para restringir o endpoint de reprocessamento | CODIGO | `src/middlewares/auth.middleware.ts` (linhas 49-61) |
| PRD-FR-13 | `docs/PRD.md` | Requisito funcional | Histórico de entregas consultável via API: todas as tentativas (sucesso e falha) com código de resposta, tempo de resposta e conteúdo enviado | TRANSCRICAO | [09:34] Marcos |
| PRD-FR-14 | `docs/PRD.md` | Requisito funcional | Identificador único por notificação que permanece o mesmo em todas as tentativas de entrega, permitindo deduplicação pelo cliente | TRANSCRICAO | [09:25] Diego |
| PRD-RNF-01 | `docs/PRD.md` | Requisito não funcional | Latência entre mudança de status e envio da notificação inferior a 10 segundos no pior caso | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-RNF-02 | `docs/PRD.md` | Requisito não funcional | Notificações cujo conteúdo exceda 64 KB rejeitadas na origem, sem envio | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| PRD-RNF-03 | `docs/PRD.md` | Requisito não funcional | Chamadas ao destino do cliente interrompidas após 10 segundos sem resposta, tratadas como falha | TRANSCRICAO | [09:42] Diego, [09:42] Sofia |
| PRD-RNF-04 | `docs/PRD.md` | Requisito não funcional | Credenciais de segurança dos webhooks não devem aparecer em nenhum registro de log da aplicação | TRANSCRICAO | [09:29] Bruno |
| PRD-RNF-04 | `docs/PRD.md` | Requisito não funcional | (Validação no código) Logger Pino com redação automática de campos sensíveis; paths `*.password`, `*.token` já configurados; necessidade de adicionar `*.secret` e `*.previousSecret` | CODIGO | `src/shared/logger/index.ts` (linhas 4-11) |
| PRD-RNF-05 | `docs/PRD.md` | Requisito não funcional | Processo de envio de notificações opera independentemente da API principal; falha ou reinício de um não afeta o outro | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-06 | `docs/PRD.md` | Requisito não funcional | Feature segue os mesmos padrões de código, estrutura e convenções do restante da plataforma, sem bibliotecas ou frameworks adicionais | TRANSCRICAO | [09:27] Bruno, [09:30] Larissa |
| PRD-RNF-06 | `docs/PRD.md` | Requisito não funcional | (Validação no código) Padrão de módulos confirmado: cada domínio em `src/modules/` com controller, service, repository, routes e schemas | CODIGO | `src/modules/orders/`, `src/modules/customers/`, `src/modules/products/` |
| PRD-RNF-07 | `docs/PRD.md` | Requisito não funcional | Toda notificação é entregue pelo menos uma vez; em caso de incerteza, a plataforma reenvia e o cliente é responsável por tratar duplicatas com base no identificador único | TRANSCRICAO | [09:24] Diego, [09:25] Diego, [09:26] Marcos |
| PRD-ESCOPO-01 | `docs/PRD.md` | Fora de escopo | Notificação por email em caso de falhas recorrentes de webhook -- descartada para esta fase, reavaliação futura | TRANSCRICAO | [09:37] Marcos, [09:37] Larissa |
| PRD-ESCOPO-02 | `docs/PRD.md` | Fora de escopo | Dashboard visual para clientes acompanharem webhooks -- projeto separado do time de frontend | TRANSCRICAO | [09:39] Marcos, [09:40] Larissa |
| PRD-ESCOPO-03 | `docs/PRD.md` | Fora de escopo | Controle de volume de envio por cliente -- observar comportamento em produção antes de implementar qualquer mecanismo de limitação | TRANSCRICAO | [09:38] Diego, [09:39] Diego, [09:39] Larissa |
| PRD-ESCOPO-04 | `docs/PRD.md` | Fora de escopo | Estratégia de limpeza de dados históricos (arquivamento após 30 dias mencionado, mas fora do escopo desta feature) | TRANSCRICAO | [09:08] Diego |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | Acúmulo de notificações pendentes por falha no processo de envio; mitigação via monitoramento de volume e alertas | TRANSCRICAO | [09:07] Bruno, [09:08] Diego |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | Cliente com destino permanentemente indisponível gerando reenvios desnecessários; mitigação via política de 5 tentativas ao longo de ~15 horas, com notificação movida para área de falhas permanentes | TRANSCRICAO | [09:15] Diego, [09:16] Diego |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Comprometimento de credencial de segurança; mitigação via credencial isolada por destino e rotação com grace period de 24h | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| PRD-RISCO-04 | `docs/PRD.md` | Risco | Prazo insuficiente para entrega até fim do trimestre (3 sprints estimadas); mitigação via escopo deliberadamente limitado | TRANSCRICAO | [09:46] Larissa, [09:47] Marcos |
| PRD-RISCO-05 | `docs/PRD.md` | Risco | Crescimento da base de dados sem estratégia de limpeza; mencionada sugestão de 30 dias, mas sem definição formal | TRANSCRICAO | [09:08] Diego |
| PRD-DECISAO-01 | `docs/PRD.md` | Decisão de produto | Notificação registrada de forma indissociável da mudança de status, sem risco de perda; priorizada sobre alternativas mais simples | TRANSCRICAO | [09:06] Diego, [09:41] Diego |
| PRD-DECISAO-02 | `docs/PRD.md` | Decisão de produto | Envio desacoplado da operação de mudança de status, feito por processo dedicado; aceita-se latência de até alguns segundos | TRANSCRICAO | [09:09] Diego, [09:11] Diego |
| PRD-DECISAO-03 | `docs/PRD.md` | Decisão de produto | Reutilização da infraestrutura existente (mesmo banco, mesma linguagem, mesmas bibliotecas), sem componentes adicionais como filas ou mensageria externa | TRANSCRICAO | [09:07] Diego, [09:07] Larissa |
| PRD-DECISAO-04 | `docs/PRD.md` | Decisão de produto | Entrega garantida pelo menos uma vez, com responsabilidade de deduplicação no cliente; mesma abordagem de plataformas de referência no mercado | TRANSCRICAO | [09:25] Diego, [09:26] Marcos |
| PRD-DECISAO-05 | `docs/PRD.md` | Decisão de produto | Credencial de segurança isolada por destino; rotação autoatendimento via API com período de transição de 24 horas | TRANSCRICAO | [09:21] Sofia, [09:22] Sofia |
| PRD-DECISAO-06 | `docs/PRD.md` | Decisão de produto | Ordenação garantida apenas por pedido individual (não global); clientes nunca solicitaram ordenação global | TRANSCRICAO | [09:12] Diego, [09:14] Marcos |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | API de pedidos (módulo existente): geração de notificações depende da operação de mudança de status | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | (Validação no código) Método `changeStatus` no `OrderService` é o ponto de integração; transação existente com update de status, histórico e estoque | CODIGO | `src/modules/orders/order.service.ts` (linhas 126-179) |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Módulo de autenticação existente: endpoints de webhook utilizam sistema de autenticação e autorização já presente | TRANSCRICAO | [09:36] Larissa, [09:37] Sofia |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | (Validação no código) Middlewares `authenticate` e `requireRole` disponíveis para reutilização | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Módulo de clientes existente: configuração de webhook vinculada a cliente cadastrado na plataforma | TRANSCRICAO | [09:31] Marcos, [09:32] Larissa |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | (Validação no código) Model `Customer` existente no schema Prisma com `id` UUID e relação `orders`; nova relação `WebhookConfig[]` será adicionada | CODIGO | `prisma/schema.prisma` (linhas 40-54) |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Revisão de segurança por Sofia: mínimo de dois dias úteis para revisar implementação de credenciais e assinatura antes do deploy; bloqueia deploy final | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Documentação para clientes: Marcos se comprometeu a documentar integração no portal de desenvolvedores, incluindo deduplicação e validação de assinatura | TRANSCRICAO | [09:26] Marcos, [09:40] Marcos |

---

## Itens da transcrição não cobertos por ADRs

Os itens abaixo foram discutidos na reunião mas não geraram ADRs próprios. Estão registrados aqui para rastreabilidade completa.

| Item | Descrição | Fonte | Localização | Observação |
|------|-----------|--------|-------------|------------|
| Requisito de negócio | Clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) pedem notificação em tempo real; prazo fim do trimestre | TRANSCRICAO | [09:00] Marcos | Contexto motivador; pertence ao PRD |
| Timeout HTTP do worker | Timeout de 10 segundos para chamadas HTTP; acima disso, tratado como falha e agendado para retry | TRANSCRICAO | [09:42] Diego, [09:42] Sofia | Decisão técnica não coberta por ADR específico |
| Formato do payload | JSON com `event_id`, `event_type`, `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents`; sem items | TRANSCRICAO | [09:43] Diego | Detalhe de implementação; pertence ao FDD |
| Limite de tamanho de payload | 64KB de limite; erro caso ultrapasse | TRANSCRICAO | [09:24] Diego, [09:24] Larissa | Classificado como requisito não funcional, não como decisão arquitetural |
| Email como fallback | Notificação por email para cliente com webhook falhando -- explicitamente fora de escopo desta fase | TRANSCRICAO | [09:37] Larissa | Escopo futuro |
| Rate limiting de saída | Observar e decidir depois; registrado como ponto em aberto | TRANSCRICAO | [09:39] Diego, [09:39] Larissa | Ponto em aberto |
| Dashboard visual | Fora de escopo; projeto separado do time de frontend | TRANSCRICAO | [09:40] Larissa | Fora de escopo |
| Prazo estimado | 3 sprints incluindo revisão de segurança | TRANSCRICAO | [09:46] Larissa, [09:47] Larissa | Planejamento; pertence ao PRD |
| Revisão de segurança | 2 dias úteis reservados para Sofia revisar código de HMAC e geração de secret | TRANSCRICAO | [09:46] Sofia | Planejamento |

---

## Estatísticas de cobertura

### ADRs

| Métrica | Valor |
|---------|-------|
| Total de ADRs rastreados | 6 |
| Decisões principais mapeadas | 6 |
| Detalhamentos de decisões | 23 |
| Alternativas consideradas mapeadas | 10 |
| Restrições/limitações mapeadas | 4 |
| Trade-offs mapeados | 6 |
| Validações cruzadas com código-fonte | 10 |
| Itens da transcrição sem ADR correspondente | 9 |
| **Subtotal de linhas (ADRs)** | **49** |

### RFC

| Métrica | Valor |
|---------|-------|
| Itens de contexto mapeados | 5 (inclui 1 validação em código) |
| Propostas técnicas mapeadas | 10 (inclui 4 referências a ADRs) |
| Alternativas consideradas mapeadas | 3 |
| Questões em aberto mapeadas | 4 |
| Impactos identificados mapeados | 4 (inclui 1 validação em código) |
| Riscos identificados mapeados | 4 |
| Referências cruzadas mapeadas | 1 |
| Validações cruzadas com código-fonte | 2 |
| **Subtotal de linhas (RFC)** | **31** |

### FDD

| Métrica | Valor |
|---------|-------|
| Requisitos funcionais mapeados (FDD-FR) | 10 |
| Requisitos não funcionais mapeados (FDD-RNF) | 6 |
| Contratos de API mapeados (FDD-CONTRATO) | 7 |
| Códigos de erro mapeados (FDD-ERRO) | 8 |
| Fluxos internos mapeados (FDD-FLUXO) | 6 |
| Pontos de integração mapeados (FDD-INT) | 9 |
| Decisões de implementação mapeadas (FDD-IMPL) | 10 (inclui 3 pontos em aberto) |
| Validações cruzadas com código-fonte | 7 |
| **Subtotal de linhas (FDD)** | **56** |

### PRD

| Métrica | Valor |
|---------|-------|
| Requisitos funcionais mapeados (PRD-FR) | 14 (inclui 3 validações em código) |
| Requisitos não funcionais mapeados (PRD-RNF) | 7 (inclui 2 validações em código) |
| Itens fora de escopo mapeados (PRD-ESCOPO) | 4 |
| Riscos mapeados (PRD-RISCO) | 5 |
| Decisões de produto mapeadas (PRD-DECISAO) | 6 |
| Dependências mapeadas (PRD-DEP) | 5 (inclui 3 validações em código) |
| Validações cruzadas com código-fonte | 8 |
| Linhas com fonte TRANSCRICAO | 41 |
| Linhas com fonte CODIGO | 8 |
| **Subtotal de linhas (PRD)** | **49** |

### Consolidado

| Métrica | Valor |
|---------|-------|
| **Total de linhas no tracker (ADRs + RFC + FDD + PRD)** | **185** |
