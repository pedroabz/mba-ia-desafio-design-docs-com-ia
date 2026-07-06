# Tracker de Rastreabilidade

Este documento mapeia cada decisão, alternativa, restrição e trade-off registrados nos ADRs às suas origens na transcrição da reunião técnica (`TRANSCRICAO.md`) ou no código-fonte do repositório.

## Legenda

- **Fonte TRANSCRICAO:** referência ao arquivo `TRANSCRICAO.md` com timestamp `[hh:mm]` e nome do participante que introduziu o argumento.
- **Fonte CODIGO:** referência ao caminho do arquivo no repositório que evidencia o item.

---

## Tabela de Rastreabilidade

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|--------|-------------|
| ADR-001 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Adotar Transactional Outbox no MySQL existente para entrega de eventos de webhook | TRANSCRICAO | [09:06] Diego |
| ADR-001-CTX-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Transação de mudança de status já é pesada (orders + history + estoque); chamada HTTP síncrona bloquearia a transação | TRANSCRICAO | [09:04] Bruno |
| ADR-001-CTX-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Rollback impossível se cliente estiver fora do ar após commit | TRANSCRICAO | [09:04] Bruno |
| ADR-001-CTX-03 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Equipe pequena; subir infraestrutura adicional é overengineering | TRANSCRICAO | [09:07] Diego |
| ADR-001-CTX-04 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Restrição | Inserção na outbox deve ocorrer dentro da mesma transação do changeStatus | CODIGO | `src/modules/orders/order.service.ts` |
| ADR-001-DET-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Chave primária UUID seguindo padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| ADR-001-DET-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Índice composto em (status, created_at) para leitura eficiente pelo worker | TRANSCRICAO | [09:08] Diego |
| ADR-001-DET-03 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Coluna status com valores: pending, processing, failed, delivered | TRANSCRICAO | [09:08] Diego |
| ADR-001-DET-04 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Decisão | Payload renderizado como snapshot no momento da inserção, não na hora do envio | TRANSCRICAO | [09:52] Larissa |
| ADR-001-ALT-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa | Dispatch síncrono no order service — descartado por bloquear transação e impossibilitar rollback | TRANSCRICAO | [09:04] Bruno |
| ADR-001-ALT-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Alternativa | Redis Streams — descartado por exigir infra extra e risco de inconsistência entre dois datastores | TRANSCRICAO | [09:07] Larissa |
| ADR-001-TRD-01 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Polling necessário (latência depende do intervalo) em troca de zero infraestrutura adicional | TRANSCRICAO | [09:07] Diego |
| ADR-001-TRD-02 | `docs/adrs/ADR-001-outbox-no-mysql.md` | Trade-off | Carga no banco com queries periódicas em troca de consistência transacional ACID | TRANSCRICAO | [09:07] Bruno |
| ADR-002 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | 5 tentativas de retry com backoff exponencial fixo (1m/5m/30m/2h/12h) | TRANSCRICAO | [09:17] Diego |
| ADR-002-DET-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | Janela total de aproximadamente 15 horas entre primeira falha e última tentativa | TRANSCRICAO | [09:17] Diego |
| ADR-002-DET-02 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | DLQ em tabela separada webhook_dead_letter com payload, motivo da falha e timestamp | TRANSCRICAO | [09:18] Diego |
| ADR-002-DET-03 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | Eventos movidos para DLQ são removidos da outbox principal, mantendo a tabela limpa | TRANSCRICAO | [09:18] Diego |
| ADR-002-DET-04 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | Replay manual via POST /admin/webhooks/dead-letter/:id/replay que reinsere na outbox como pending | TRANSCRICAO | [09:18] Diego |
| ADR-002-DET-05 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Decisão | Endpoint de replay exige role ADMIN com log de auditoria de quem acionou | TRANSCRICAO | [09:36] Sofia |
| ADR-002-ALT-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Alternativa | 3 tentativas com backoff agressivo — descartado por não cobrir manutenções de 2 horas | TRANSCRICAO | [09:16] Bruno |
| ADR-002-ALT-02 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Alternativa | Retry infinito sem limite de tentativas — descartado por risco de evento pendurado para sempre | TRANSCRICAO | [09:15] Diego |
| ADR-002-CTX-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Restrição | Clientes com janelas de manutenção planejada de até 2 horas já foram observados | TRANSCRICAO | [09:16] Diego |
| ADR-002-TRD-01 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Trade-off | Entrega não garantida em tempo real após 15h; evento só chega via replay manual | TRANSCRICAO | [09:17] Marcos |
| ADR-002-TRD-02 | `docs/adrs/ADR-002-retry-com-backoff-e-dlq.md` | Trade-off | Custo operacional de monitorar DLQ e acionar replay sem alertas automáticos | TRANSCRICAO | [09:18] Bruno |
| ADR-003 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | HMAC-SHA256 como mecanismo de autenticação de entregas de webhook | TRANSCRICAO | [09:20] Sofia |
| ADR-003-DET-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Assinatura enviada no header X-Signature sobre o body do request | TRANSCRICAO | [09:20] Sofia |
| ADR-003-DET-02 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Secret única por endpoint de webhook, não global da plataforma | TRANSCRICAO | [09:21] Sofia |
| ADR-003-DET-03 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | Rotação de secret com grace period de 24 horas; secret antiga válida em paralelo | TRANSCRICAO | [09:21] Sofia |
| ADR-003-DET-04 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Decisão | TLS obrigatório; URLs com esquema HTTP rejeitadas na validação de cadastro | TRANSCRICAO | [09:23] Sofia |
| ADR-003-ALT-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Alternativa | Secret global única para toda a plataforma — descartada pelo raio de impacto em caso de vazamento | TRANSCRICAO | [09:21] Sofia |
| ADR-003-CTX-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Restrição | Incidente anterior: cliente vazou secret em log de aplicação própria | TRANSCRICAO | [09:22] Diego |
| ADR-003-TRD-01 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Trade-off | Complexidade de manter até duas secrets válidas simultaneamente durante grace period de rotação | TRANSCRICAO | [09:21] Sofia |
| ADR-003-TRD-02 | `docs/adrs/ADR-003-autenticacao-hmac-sha256.md` | Trade-off | Segurança depende de o cliente implementar corretamente a verificação do HMAC no seu lado | TRANSCRICAO | [09:20] Sofia |
| ADR-004 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Garantia at-least-once com header X-Event-Id (UUID) para deduplicação client-side | TRANSCRICAO | [09:25] Diego |
| ADR-004-DET-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | UUID gerado como PK da webhook_outbox serve como event_id | TRANSCRICAO | [09:25] Diego |
| ADR-004-DET-02 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Header X-Event-Id incluído em toda requisição de dispatch com valor idêntico em retries | TRANSCRICAO | [09:25] Diego |
| ADR-004-DET-03 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Responsabilidade de deduplicação é do cliente (padrão Stripe/GitHub) | TRANSCRICAO | [09:25] Diego |
| ADR-004-DET-04 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Decisão | Documentação destacada no portal do desenvolvedor sobre responsabilidade de dedup | TRANSCRICAO | [09:26] Marcos |
| ADR-004-ALT-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Alternativa | Exactly-once com coordenação bidirecional (two-phase commit) — descartada por complexidade | TRANSCRICAO | [09:25] Diego |
| ADR-004-CTX-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Restrição | Projeto já utiliza UUID como padrão de identificação em todas as entidades | CODIGO | `prisma/schema.prisma` |
| ADR-004-TRD-01 | `docs/adrs/ADR-004-garantia-at-least-once.md` | Trade-off | Responsabilidade de dedup transferida ao cliente em troca de simplicidade e baixo acoplamento | TRANSCRICAO | [09:25] Sofia |
| ADR-005 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Worker em processo Node.js separado com polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| ADR-005-DET-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Entry-point dedicado em src/worker.ts, executado via script npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-005-DET-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | PrismaClient separado por processo; mesmo banco e DATABASE_URL, instância nova | TRANSCRICAO | [09:30] Bruno |
| ADR-005-DET-03 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Worker único inicialmente; ordenamento por order_id garantido apenas com single-worker | TRANSCRICAO | [09:12] Diego |
| ADR-005-DET-04 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Lógica de processamento em src/modules/webhooks/webhook.processor.ts | TRANSCRICAO | [09:28] Bruno |
| ADR-005-DET-05 | `docs/adrs/ADR-005-worker-polling-separado.md` | Decisão | Configuração de banco reutilizada de src/config/database.ts | CODIGO | `src/config/database.ts` |
| ADR-005-ALT-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Alternativa | MySQL trigger para notificação externa — descartada por limitação do MySQL (não notifica processo externo) | TRANSCRICAO | [09:09] Diego |
| ADR-005-ALT-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Alternativa | Worker dentro do processo da API (src/server.ts) — descartada por risco de perda em restart/crash | TRANSCRICAO | [09:11] Diego |
| ADR-005-ALT-03 | `docs/adrs/ADR-005-worker-polling-separado.md` | Alternativa | Redis ou message broker (RabbitMQ, SQS) — descartada por overengineering e inconsistência entre datastores | TRANSCRICAO | [09:07] Diego |
| ADR-005-CTX-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Restrição | MySQL não oferece NOTIFY/LISTEN nativo como PostgreSQL | TRANSCRICAO | [09:09] Diego |
| ADR-005-CTX-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Restrição | Requisito de entrega em menos de 10 segundos após mudança de status | TRANSCRICAO | [09:02] Marcos |
| ADR-005-CTX-03 | `docs/adrs/ADR-005-worker-polling-separado.md` | Restrição | API existente roda em src/server.ts como entry-point único atual | CODIGO | `src/server.ts` |
| ADR-005-TRD-01 | `docs/adrs/ADR-005-worker-polling-separado.md` | Trade-off | Latência mínima de até 2 segundos no pior caso em troca de simplicidade operacional | TRANSCRICAO | [09:10] Larissa |
| ADR-005-TRD-02 | `docs/adrs/ADR-005-worker-polling-separado.md` | Trade-off | Sem garantia de ordenamento global caso escale para múltiplos workers no futuro | TRANSCRICAO | [09:13] Larissa |
| ADR-005-TRD-03 | `docs/adrs/ADR-005-worker-polling-separado.md` | Trade-off | Polling consome recursos mesmo sem eventos pendentes; custo mitigado pelo índice (status, created_at) | TRANSCRICAO | [09:07] Bruno |
| ADR-006 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Máximo reuso dos padrões existentes; webhook como módulo igual aos demais | TRANSCRICAO | [09:30] Larissa |
| ADR-006-DET-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Estrutura em src/modules/webhooks/ com controller, service, repository, routes, schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-DET-02 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Códigos de erro prefixados por WEBHOOK_ (WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, etc.) | TRANSCRICAO | [09:28] Bruno |
| ADR-006-DET-03 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Logger Pino reutilizado sem dependência nova | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DET-04 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Middleware de erro centralizado captura AppError de webhooks sem alteração necessária | TRANSCRICAO | [09:29] Bruno |
| ADR-006-DET-05 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Rotas administrativas protegidas por authenticate + requireRole('ADMIN') existente | TRANSCRICAO | [09:36] Larissa |
| ADR-006-DET-06 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Decisão | Nenhuma biblioteca, framework ou padrão arquitetural novo será introduzido | TRANSCRICAO | [09:30] Larissa |
| ADR-006-CTX-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Restrição | Padrão de módulo já definido no projeto (controller, service, repository, routes, schemas) | CODIGO | `src/modules/orders/` |
| ADR-006-CTX-02 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Restrição | Classe AppError com statusCode, errorCode e details já existente | CODIGO | `src/shared/errors/app-error.ts` |
| ADR-006-CTX-03 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Restrição | Erros HTTP especializados (NotFoundError, ConflictError) já existentes | CODIGO | `src/shared/errors/http-errors.ts` |
| ADR-006-CTX-04 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Restrição | Pino configurado como logger do projeto com redact de campos sensíveis | CODIGO | `src/shared/logger/index.ts` |
| ADR-006-CTX-05 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Restrição | Middleware de erro centralizado já trata AppError, ZodError e PrismaClientKnownRequestError | CODIGO | `src/middlewares/error.middleware.ts` |
| ADR-006-CTX-06 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Restrição | Middleware auth com authenticate e requireRole já disponível | CODIGO | `src/middlewares/auth.middleware.ts` |
| ADR-006-ALT-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Alternativa | Framework/estrutura separada para webhooks — descartada por fragmentar a codebase e duplicar convenções | TRANSCRICAO | [09:27] Bruno |
| ADR-006-ALT-02 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Alternativa | Biblioteca externa de webhooks (standardwebhooks, svix) — descartada por dependência desnecessária e não integração com padrões internos | TRANSCRICAO | [09:30] Larissa |
| ADR-006-TRD-01 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Trade-off | Acoplamento ao padrão atual; refatoração futura (hexagonal, CQRS) impactaria webhooks junto | TRANSCRICAO | [09:27] Bruno |
| ADR-006-TRD-02 | `docs/adrs/ADR-006-reuso-de-padroes-existentes.md` | Trade-off | Sem otimizações específicas de webhook (fan-out, rate limiting por subscriber) que bibliotecas dedicadas ofereceriam | TRANSCRICAO | [09:30] Larissa |
| RFC-CTX-01 | `docs/RFC.md` | Contexto | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram notificações de mudança de status de pedidos; dependem atualmente de polling no GET /orders | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-02 | `docs/RFC.md` | Contexto | Atlas Comercial sinalizou possibilidade de migração para concorrente caso a funcionalidade não seja entregue até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| RFC-CTX-03 | `docs/RFC.md` | Contexto | Requisito de negócio: notificações em menos de 10 segundos após mudança de status | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-04 | `docs/RFC.md` | Contexto | Webhooks são exclusivamente outbound (plataforma envia, clientes recebem) | TRANSCRICAO | [09:02] Marcos |
| RFC-CTX-05 | `docs/RFC.md` | Contexto | Ponto crítico de integração: método changeStatus opera em transação atômica (atualiza order, insere histórico, ajusta estoque) | TRANSCRICAO | [09:04] Bruno |
| RFC-CTX-06 | `docs/RFC.md` | Contexto | Transação de mudança de status já existente no serviço de pedidos | CODIGO | `src/modules/orders/order.service.ts` |
| RFC-ALT-01 | `docs/RFC.md` | Alternativa | Despacho síncrono dentro da transação de mudança de status -- descartado por bloquear transação, travar fluxo e impossibilitar rollback razoável | TRANSCRICAO | [09:04] Bruno, [09:06] Diego |
| RFC-ALT-02 | `docs/RFC.md` | Alternativa | Fila externa (Redis Streams ou message broker) -- descartada por exigir infra nova, quebrar garantia transacional atômica e overengineering para equipe pequena | TRANSCRICAO | [09:07] Larissa, [09:07] Diego |
| RFC-DEC-01 | `docs/RFC.md` | Decisão | Solução composta por três componentes: outbox transacional, worker de despacho separado e módulo de configuração de webhooks | TRANSCRICAO | [09:06] Diego, [09:11] Larissa |
| RFC-DEC-02 | `docs/RFC.md` | Decisão | Inserção na outbox dentro da mesma transação SQL que altera status do pedido (consistência transacional ACID) | TRANSCRICAO | [09:06] Diego |
| RFC-DEC-03 | `docs/RFC.md` | Decisão | Worker em processo Node.js independente com polling a cada 2 segundos | TRANSCRICAO | [09:09] Diego, [09:10] Larissa |
| RFC-DEC-04 | `docs/RFC.md` | Decisão | Entrega at-least-once com UUID no header X-Event-Id para deduplicação client-side | TRANSCRICAO | [09:25] Diego |
| RFC-DEC-05 | `docs/RFC.md` | Decisão | Retry com backoff exponencial em 5 tentativas (1m/5m/30m/2h/12h), janela total de ~15 horas; DLQ em tabela separada com replay manual administrativo | TRANSCRICAO | [09:17] Diego, [09:18] Diego |
| RFC-DEC-06 | `docs/RFC.md` | Decisão | HMAC-SHA256 com secret única por endpoint, rotação com grace period de 24 horas, HTTPS obrigatório | TRANSCRICAO | [09:20] Sofia, [09:21] Sofia |
| RFC-DEC-07 | `docs/RFC.md` | Decisão | Payload renderizado como snapshot no momento da inserção na outbox, não na hora do envio | TRANSCRICAO | [09:52] Larissa, [09:52] Diego |
| RFC-DEC-08 | `docs/RFC.md` | Decisão | Ordenação por pedido garantida apenas com single-worker; sem garantia de ordenamento global | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| RFC-DEC-09 | `docs/RFC.md` | Decisão | Módulo de webhooks segue padrão modular existente (controller, service, repository, routes, schemas) em src/modules/webhooks/ | TRANSCRICAO | [09:27] Bruno, [09:30] Larissa |
| RFC-DEC-10 | `docs/RFC.md` | Decisão | Nenhuma infraestrutura, biblioteca ou padrão arquitetural novo introduzido; reuso de AppError, Pino, error middleware, authenticate/requireRole | TRANSCRICAO | [09:30] Larissa |
| RFC-DEC-11 | `docs/RFC.md` | Decisão | Instância própria de conexão ao banco no worker (processo distinto), mesmo banco de dados | TRANSCRICAO | [09:30] Bruno |
| RFC-QA-01 | `docs/RFC.md` | Questão em Aberto | Rate limiting de saída: se cliente tem muitos pedidos mudando simultaneamente, sistema dispara todas as notificações sem controle de vazão; decisão adiada para observação em produção | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| RFC-QA-02 | `docs/RFC.md` | Questão em Aberto | Notificação ao cliente sobre falhas recorrentes do webhook (email descartado para esta fase); mecanismo de alerta indefinido | TRANSCRICAO | [09:37] Marcos, [09:37] Larissa |
| RFC-QA-03 | `docs/RFC.md` | Questão em Aberto | Archival de eventos entregues na outbox (sugerido 30 dias); política de cleanup fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | `docs/RFC.md` | Questão em Aberto | Estratégia de escalabilidade do worker: single-worker deliberado para esta fase; opções futuras (particionamento por order_id, lock pessimista) mencionadas mas não decididas | TRANSCRICAO | [09:13] Diego |
| RFC-RISCO-01 | `docs/RFC.md` | Risco | Acúmulo de eventos na outbox pode degradar performance do banco; mitigação via índices otimizados e archival futuro | TRANSCRICAO | [09:07] Bruno, [09:08] Diego |
| RFC-RISCO-02 | `docs/RFC.md` | Risco | Worker fora do ar causa acúmulo sem despacho; eventos permanecem no banco e são processados ao reiniciar | TRANSCRICAO | [09:11] Diego |
| RFC-RISCO-03 | `docs/RFC.md` | Risco | Clientes com endpoints lentos geram backlog no single-worker; mitigação via timeout de 10 segundos | TRANSCRICAO | [09:42] Diego |
| RFC-RISCO-04 | `docs/RFC.md` | Risco | Vazamento de secret de um cliente; mitigação via secret única por endpoint, rotação com grace period e HTTPS obrigatório | TRANSCRICAO | [09:22] Diego, [09:21] Sofia |
| RFC-RISCO-05 | `docs/RFC.md` | Risco | Prazo apertado (compromisso com Atlas Comercial até fim do trimestre); estimativa de 3 sprints com revisão de segurança incluída | TRANSCRICAO | [09:45] Marcos, [09:46] Larissa |
| RFC-TRD-01 | `docs/RFC.md` | Trade-off | Transação de mudança de status estendida com inserção na outbox (única alteração em fluxo crítico existente); risco mitigado por ser operação SQL simples | TRANSCRICAO | [09:40] Bruno, [09:41] Diego |
| RFC-TRD-02 | `docs/RFC.md` | Trade-off | Worker como novo processo exige configuração de deploy e monitoramento adicionais em troca de isolamento da API | TRANSCRICAO | [09:11] Diego |
| FDD-FLUXO-01 | `docs/FDD.md` | Fluxo | Função `publishWebhookEvent(tx, order, fromStatus, toStatus)` inserida entre passos 5 e 6 do `changeStatus`, dentro da mesma transação ACID | TRANSCRICAO | [09:06] Diego, [09:41] Bruno |
| FDD-FLUXO-02 | `docs/FDD.md` | Fluxo | `publishWebhookEvent` consulta endpoints ativos do cliente, filtra por `eventFilters` vs. `toStatus`, insere snapshot na outbox; se nenhum endpoint corresponder, não insere nada (silencioso) | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno |
| FDD-FLUXO-03 | `docs/FDD.md` | Fluxo | Worker processa batch de eventos PENDING ordenados por `created_at ASC`, marca como PROCESSING, despacha HTTP POST, marca como DELIVERED ou agenda retry | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-04 | `docs/FDD.md` | Fluxo | Se endpoint foi deletado, worker move evento direto para DLQ com `failure_reason = "Endpoint removed"`, sem retry | TRANSCRICAO | [09:06] Diego |
| FDD-FLUXO-05 | `docs/FDD.md` | Fluxo | DLQ: após 5a tentativa falhada, inserir em `webhook_dead_letter`, remover da `webhook_outbox`, registrar log warn | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-06 | `docs/FDD.md` | Fluxo | Replay de DLQ: buscar registro, reinserir na outbox com `attempt_count = 0` e `status = PENDING`, remover da DLQ, registrar log info com `replayed_by` | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| FDD-FLUXO-07 | `docs/FDD.md` | Fluxo | Recuperação de eventos travados: na inicialização do worker, resetar eventos em PROCESSING há mais de 60 segundos para PENDING | TRANSCRICAO | [09:11] Diego |
| FDD-FLUXO-08 | `docs/FDD.md` | Fluxo | Rotação de secret: gerar nova com `crypto.randomBytes(32)`, mover atual para `previousSecret` com expiração em 24h, retornar nova secret na resposta | TRANSCRICAO | [09:21] Sofia |
| FDD-FR-01 | `docs/FDD.md` | Requisito Funcional | POST /api/v1/webhooks -- criar configuração de webhook com url (HTTPS obrigatório), customerId, eventFilters; secret gerada pelo servidor e devolvida na resposta | TRANSCRICAO | [09:31] Marcos, [09:23] Sofia |
| FDD-FR-02 | `docs/FDD.md` | Requisito Funcional | GET /api/v1/webhooks?customerId=... -- listar webhooks de um cliente com paginação; secret não incluída na listagem | TRANSCRICAO | [09:33] Bruno |
| FDD-FR-03 | `docs/FDD.md` | Requisito Funcional | PATCH /api/v1/webhooks/:id -- editar campos de configuração (url, eventFilters, active); secret não editável via PATCH | TRANSCRICAO | [09:33] Bruno |
| FDD-FR-04 | `docs/FDD.md` | Requisito Funcional | DELETE /api/v1/webhooks/:id -- remover webhook permanentemente; eventos pendentes na outbox descartados pelo worker ao detectar endpoint removido | TRANSCRICAO | [09:33] Bruno |
| FDD-FR-05 | `docs/FDD.md` | Requisito Funcional | POST /api/v1/webhooks/:id/rotate-secret -- rotação de secret com grace period de 24h para secret anterior | TRANSCRICAO | [09:21] Sofia |
| FDD-FR-06 | `docs/FDD.md` | Requisito Funcional | GET /api/v1/webhooks/:id/deliveries -- histórico de tentativas de entrega (sucesso e falha) com paginação | TRANSCRICAO | [09:34] Marcos |
| FDD-FR-07 | `docs/FDD.md` | Requisito Funcional | POST /api/v1/admin/webhooks/dead-letter/:id/replay -- replay de DLQ com role ADMIN e log de auditoria | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| FDD-FR-08 | `docs/FDD.md` | Requisito Funcional | Filtragem de eventos por status na inserção da outbox: se `eventFilters` definido, só insere para status presentes na lista; se nulo/vazio, todos os status geram evento | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno |
| FDD-CONTRATO-01 | `docs/FDD.md` | Contrato | Payload do webhook: JSON com `event_id`, `event_type` ("order.status_changed"), `timestamp`, `data` contendo `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-02 | `docs/FDD.md` | Contrato | Headers do dispatch HTTP: `Content-Type`, `X-Event-Id` (UUID da outbox), `X-Webhook-Id` (UUID do endpoint), `X-Signature` (HMAC-SHA256 hex), `X-Timestamp` (ISO 8601 do envio) | TRANSCRICAO | [09:44] Diego, [09:44] Sofia |
| FDD-CONTRATO-03 | `docs/FDD.md` | Contrato | Assinatura HMAC-SHA256: `createHmac('sha256', secret).update(body).digest('hex')` sobre o corpo JSON exato enviado | TRANSCRICAO | [09:20] Sofia |
| FDD-CONTRATO-04 | `docs/FDD.md` | Contrato | Tabela de retry com intervalos fixos: 1m, 5m, 30m, 2h, 12h (5 tentativas, janela total ~15h) | TRANSCRICAO | [09:17] Diego |
| FDD-RNF-01 | `docs/FDD.md` | Requisito Não Funcional | Timeout de 10 segundos por chamada HTTP de dispatch | TRANSCRICAO | [09:42] Diego |
| FDD-RNF-02 | `docs/FDD.md` | Requisito Não Funcional | Limite de payload de 64KB na inserção da outbox; erro se exceder | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| FDD-RNF-03 | `docs/FDD.md` | Requisito Não Funcional | TLS obrigatório: URLs HTTP rejeitadas na validação de cadastro | TRANSCRICAO | [09:23] Sofia |
| FDD-RNF-04 | `docs/FDD.md` | Requisito Não Funcional | Polling do worker a cada 2 segundos | TRANSCRICAO | [09:09] Diego |
| FDD-RNF-05 | `docs/FDD.md` | Requisito Não Funcional | Latência máxima de entrega abaixo de 10 segundos (2s polling + 10s timeout no pior caso) | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| FDD-ERRO-01 | `docs/FDD.md` | Erro Previsto | `WEBHOOK_NOT_FOUND` (404) -- webhook com ID informado não existe | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | `docs/FDD.md` | Erro Previsto | `WEBHOOK_INVALID_URL` (400) -- URL não utiliza esquema HTTPS | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-03 | `docs/FDD.md` | Erro Previsto | `WEBHOOK_PAYLOAD_TOO_LARGE` (422) -- payload serializado excede 64KB | TRANSCRICAO | [09:24] Diego |
| FDD-ERRO-04 | `docs/FDD.md` | Erro Previsto | `WEBHOOK_DUPLICATE_URL` (409) -- já existe webhook ativo para o mesmo customer+URL | TRANSCRICAO | [09:21] Sofia |
| FDD-ERRO-05 | `docs/FDD.md` | Erro Previsto | `WEBHOOK_DLQ_NOT_FOUND` (404) -- registro na DLQ não encontrado para replay | TRANSCRICAO | [09:18] Diego |
| FDD-ERRO-06 | `docs/FDD.md` | Erro Previsto | `WEBHOOK_DELIVERY_FAILED` (N/A, log interno) -- falha de entrega registrada pelo worker | TRANSCRICAO | [09:15] Diego |
| FDD-INT-01 | `docs/FDD.md` | Integração | Inserção da chamada `publishWebhookEvent` dentro do bloco `prisma.$transaction()` do método `changeStatus`, entre `orderStatusHistory.create` (linha 159) e `findUnique` final (linha 169) | CODIGO | `src/modules/orders/order.service.ts` |
| FDD-INT-02 | `docs/FDD.md` | Integração | Novos erros de webhook estendem `AppError` diretamente (não `NotFoundError`) para permitir `errorCode` com prefixo `WEBHOOK_` | CODIGO | `src/shared/errors/app-error.ts` |
| FDD-INT-03 | `docs/FDD.md` | Integração | Middleware de erro captura automaticamente novos erros `WEBHOOK_*` via `if (err instanceof AppError)` sem alteração necessária | CODIGO | `src/middlewares/error.middleware.ts` |
| FDD-INT-04 | `docs/FDD.md` | Integração | Endpoint de replay utiliza `authenticate` + `requireRole('ADMIN')` existentes; demais endpoints de CRUD utilizam apenas `authenticate` | CODIGO | `src/middlewares/auth.middleware.ts` |
| FDD-INT-05 | `docs/FDD.md` | Integração | Worker reutiliza `createPrismaClient` de `src/config/database.ts` para instância própria de PrismaClient | CODIGO | `src/config/database.ts` |
| FDD-INT-06 | `docs/FDD.md` | Integração | Entry point `src/worker.ts` segue estrutura de bootstrap de `src/server.ts`: imports de Prisma, logger e env; graceful shutdown com SIGINT/SIGTERM | CODIGO | `src/server.ts` |
| FDD-INT-07 | `docs/FDD.md` | Integração | Tipo `Controllers` em `src/routes/index.ts` deve ser estendido para incluir `webhooks: WebhookController`; `buildApiRouter` registra rotas de CRUD e admin | CODIGO | `src/routes/index.ts` |
| FDD-INT-08 | `docs/FDD.md` | Integração | `buildControllers` em `src/app.ts` deve instanciar WebhookRepository, WebhookService e WebhookController seguindo padrão dos demais módulos | CODIGO | `src/app.ts` |
| FDD-INT-09 | `docs/FDD.md` | Integração | Paginação dos endpoints de listagem e histórico reutiliza função `paginated()` de `src/shared/http/response.ts` | CODIGO | `src/shared/http/response.ts` |
| FDD-INT-10 | `docs/FDD.md` | Integração | Adicionar `'*.secret'` ao array `redactPaths` em `src/shared/logger/index.ts` (linha 4) para evitar vazamento de secrets nos logs | CODIGO | `src/shared/logger/index.ts` |
| FDD-INT-11 | `docs/FDD.md` | Integração | Quatro novas tabelas em `prisma/schema.prisma`: `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`; relação reversa `webhookEndpoints` no model `Customer` existente | CODIGO | `prisma/schema.prisma` |
| FDD-INT-12 | `docs/FDD.md` | Integração | UUID como PK seguindo padrão existente `@default(uuid()) @db.Char(36)`; exceto `webhook_outbox.id` que é gerado no application layer (corresponde ao `event_id`) | TRANSCRICAO | [09:51] Larissa |
| FDD-INT-13 | `docs/FDD.md` | Integração | Índice composto `[status, createdAt]` na outbox para otimizar query principal do worker | TRANSCRICAO | [09:08] Diego |
| FDD-INT-14 | `docs/FDD.md` | Integração | Constraint `@@unique([customerId, url])` em `webhook_endpoints` para prevenir duplicação de URL por cliente | TRANSCRICAO | [09:21] Sofia |
| FDD-DEC-01 | `docs/FDD.md` | Decisão de Implementação | Payload renderizado como snapshot no momento da inserção na outbox, não sob demanda no envio | TRANSCRICAO | [09:52] Larissa, [09:52] Diego |
| FDD-DEC-02 | `docs/FDD.md` | Decisão de Implementação | PrismaClient separado por processo no worker; mesmo banco e DATABASE_URL, instância nova | TRANSCRICAO | [09:30] Bruno |
| FDD-DEC-03 | `docs/FDD.md` | Decisão de Implementação | Nenhuma dependência nova; HMAC via `node:crypto`, HTTP dispatch via `fetch` nativo do Node.js 20+ | TRANSCRICAO | [09:30] Larissa |
| FDD-DEC-04 | `docs/FDD.md` | Decisão de Implementação | Payload enxuto: não inclui items do pedido para evitar inflação; cliente consulta `GET /orders/:id` se precisar de detalhes | TRANSCRICAO | [09:43] Diego |
| PRD-RF-01 | `docs/PRD.md` | Requisito Funcional | Cadastro de endpoint de webhook para um cliente, com URL de destino e lista de status de interesse; credencial gerada pelo sistema e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | `docs/PRD.md` | Requisito Funcional | Edição de webhook existente (URL, status de interesse, estado ativo/inativo) | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-03 | `docs/PRD.md` | Requisito Funcional | Remoção de webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | `docs/PRD.md` | Requisito Funcional | Listagem dos webhooks configurados para um determinado cliente | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-05 | `docs/PRD.md` | Requisito Funcional | Registro automático de evento de notificação atômico com a transação de mudança de status; rollback elimina o evento | TRANSCRICAO | [09:06] Diego, [09:40] Bruno |
| PRD-RF-06 | `docs/PRD.md` | Requisito Funcional | Filtragem de eventos de acordo com os status de interesse configurados em cada webhook; status não selecionados não geram evento | TRANSCRICAO | [09:33] Marcos, [09:34] Bruno |
| PRD-RF-07 | `docs/PRD.md` | Requisito Funcional | Consulta de histórico de entregas de um webhook, incluindo status de sucesso/falha, tempo de resposta e informações de erro | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-08 | `docs/PRD.md` | Requisito Funcional | Retry automático com backoff exponencial: 5 tentativas nos intervalos de 1m, 5m, 30m, 2h, 12h | TRANSCRICAO | [09:15] Diego, [09:17] Diego |
| PRD-RF-09 | `docs/PRD.md` | Requisito Funcional | Após esgotar retries, evento movido para Dead Letter Queue persistida com payload original e motivo da falha | TRANSCRICAO | [09:17] Diego, [09:18] Diego |
| PRD-RF-10 | `docs/PRD.md` | Requisito Funcional | Administrador (role ADMIN) pode reprocessar eventos da DLQ com registro de auditoria | TRANSCRICAO | [09:18] Diego, [09:36] Sofia |
| PRD-RF-11 | `docs/PRD.md` | Requisito Funcional | Rotação de credencial com grace period de 24 horas; credencial anterior permanece válida em paralelo com a nova | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-01 | `docs/PRD.md` | Requisito Não Funcional | Latência de entrega < 10 segundos entre commit da transação de status e recebimento pelo cliente (p95) | TRANSCRICAO | [09:02] Marcos, [09:10] Larissa |
| PRD-RNF-02 | `docs/PRD.md` | Requisito Não Funcional | Consistência transacional: evento existe se e somente se a transação de mudança de status commitou com sucesso | TRANSCRICAO | [09:06] Diego |
| PRD-RNF-03 | `docs/PRD.md` | Requisito Não Funcional | Isolamento de processos: worker roda como processo separado da API | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-04 | `docs/PRD.md` | Requisito Não Funcional | Autenticidade do payload via assinatura criptográfica com credencial única por endpoint | TRANSCRICAO | [09:20] Sofia |
| PRD-RNF-05 | `docs/PRD.md` | Requisito Não Funcional | TLS obrigatório; URLs HTTP rejeitadas na validação de cadastro | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-06 | `docs/PRD.md` | Requisito Não Funcional | Rotação de credenciais com grace period de 24 horas | TRANSCRICAO | [09:21] Sofia |
| PRD-RNF-07 | `docs/PRD.md` | Requisito Não Funcional | Proteção contra replay via timestamp no header | TRANSCRICAO | [09:44] Diego |
| PRD-RNF-08 | `docs/PRD.md` | Requisito Não Funcional | Timeout de entrega de 10 segundos para resposta do endpoint do cliente | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-09 | `docs/PRD.md` | Requisito Não Funcional | Limite de payload de 64KB; erro (não truncamento) caso exceda | TRANSCRICAO | [09:24] Diego, [09:24] Larissa |
| PRD-RNF-10 | `docs/PRD.md` | Requisito Não Funcional | Semântica de entrega at-least-once; deduplicação via identificador único é responsabilidade do cliente | TRANSCRICAO | [09:24] Diego, [09:25] Diego |
| PRD-RNF-11 | `docs/PRD.md` | Requisito Não Funcional | Ordenação garantida por pedido individual com worker único; sem garantia de ordering global | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| PRD-RNF-12 | `docs/PRD.md` | Requisito Não Funcional | Observabilidade via logs estruturados com reutilização do logger Pino existente | TRANSCRICAO | [09:29] Bruno |
| PRD-RNF-13 | `docs/PRD.md` | Requisito Não Funcional | Aderência aos padrões existentes: estrutura modular, hierarquia de erros, validação de schemas, middleware de autorização | TRANSCRICAO | [09:27] Bruno, [09:30] Larissa |
| PRD-ESCOPO-01 | `docs/PRD.md` | Fora de Escopo | Notificação por email quando webhook falha repetidamente -- adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESCOPO-02 | `docs/PRD.md` | Fora de Escopo | Dashboard visual para clientes acompanharem webhooks -- projeto separado do time de frontend | TRANSCRICAO | [09:39] Marcos, [09:40] Larissa |
| PRD-ESCOPO-03 | `docs/PRD.md` | Fora de Escopo | Rate limiting de saída -- adiado para observação em produção | TRANSCRICAO | [09:38] Diego, [09:39] Larissa |
| PRD-ESCOPO-04 | `docs/PRD.md` | Fora de Escopo | Garantia de exactly-once delivery -- semântica é at-least-once; deduplicação é responsabilidade do cliente | TRANSCRICAO | [09:24] Diego, [09:25] Diego |
| PRD-ESCOPO-05 | `docs/PRD.md` | Fora de Escopo | Ordering global de eventos entre pedidos diferentes -- garantia apenas por pedido com worker único | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| PRD-ESCOPO-06 | `docs/PRD.md` | Fora de Escopo | Múltiplos workers em paralelo -- decisão deliberada de manter worker único nesta fase | TRANSCRICAO | [09:13] Diego |
| PRD-ESCOPO-07 | `docs/PRD.md` | Fora de Escopo | Archival/cleanup de eventos entregues -- mencionado como 30 dias, mas fora do escopo desta feature | TRANSCRICAO | [09:08] Diego |
| PRD-DECISAO-01 | `docs/PRD.md` | Decisão de Produto | Notificação assíncrona com registro atômico: despacho assíncrono, registro do evento atômico com transação de status, sem infraestrutura adicional | TRANSCRICAO | [09:04] Bruno, [09:06] Diego, [09:07] Larissa |
| PRD-DECISAO-02 | `docs/PRD.md` | Decisão de Produto | Detecção de eventos por verificação periódica a cada 2 segundos; latência mínima de 2s aceita dado requisito de < 10s | TRANSCRICAO | [09:09] Diego, [09:10] Larissa |
| PRD-DECISAO-03 | `docs/PRD.md` | Decisão de Produto | Entrega at-least-once (não exactly-once); deduplicação via identificador único é responsabilidade do cliente, seguindo padrão Stripe/GitHub | TRANSCRICAO | [09:24] Diego, [09:25] Diego |
| PRD-DECISAO-04 | `docs/PRD.md` | Decisão de Produto | 5 tentativas com backoff crescente (1m/5m/30m/2h/12h), depois DLQ; 3 tentativas descartadas por insuficientes para cobrir janelas de manutenção de 2 horas | TRANSCRICAO | [09:15] Diego, [09:16] Diego, [09:17] Diego |
| PRD-DECISAO-05 | `docs/PRD.md` | Decisão de Produto | Credencial única por endpoint com rotação e grace period de 24h; credencial global descartada por risco de vazamento generalizado | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| PRD-DECISAO-06 | `docs/PRD.md` | Decisão de Produto | Payload como snapshot no momento da transição, não sob demanda no envio; garante que evento reflete estado exato do pedido na transição | TRANSCRICAO | [09:51] Larissa, [09:52] Diego |
| PRD-RISCO-01 | `docs/PRD.md` | Risco | Prazo apertado: compromisso com Atlas Comercial até fim do trimestre; estimativa de 3 sprints com revisão de segurança incluída | TRANSCRICAO | [09:45] Marcos, [09:46] Larissa |
| PRD-RISCO-02 | `docs/PRD.md` | Risco | Worker fora do ar causa acúmulo de eventos sem despacho; eventos permanecem persistidos e são processados ao reiniciar | TRANSCRICAO | [09:11] Diego |
| PRD-RISCO-03 | `docs/PRD.md` | Risco | Acúmulo de eventos não processados degrada performance do banco; mitigação via índices otimizados e archival futuro | TRANSCRICAO | [09:07] Bruno, [09:08] Diego |
| PRD-RISCO-04 | `docs/PRD.md` | Risco | Clientes com endpoints lentos geram backlog no worker; mitigação via timeout de 10 segundos | TRANSCRICAO | [09:42] Diego |
| PRD-RISCO-05 | `docs/PRD.md` | Risco | Vazamento de credencial do cliente; mitigação via credencial única por endpoint, rotação com grace period e HTTPS obrigatório | TRANSCRICAO | [09:21] Sofia, [09:22] Diego |
| PRD-RISCO-06 | `docs/PRD.md` | Risco | Duplicação de eventos causa efeitos colaterais no sistema do cliente; semântica at-least-once documentada, identificador único enviado para deduplicação | TRANSCRICAO | [09:24] Diego, [09:26] Marcos |
| PRD-RISCO-07 | `docs/PRD.md` | Risco | Perda de ordenação ao escalar para múltiplos workers no futuro; limitação conhecida e documentada | TRANSCRICAO | [09:12] Diego, [09:13] Larissa |
| PRD-DEP-01 | `docs/PRD.md` | Dependência | Transação de mudança de status de pedido: ponto de inserção do evento de notificação; a feature depende de estender a transação existente | CODIGO | `src/modules/orders/order.service.ts` |
| PRD-DEP-02 | `docs/PRD.md` | Dependência | Sistema de autenticação JWT: protege endpoints de configuração de webhook e endpoint admin de replay | CODIGO | `src/middlewares/auth.middleware.ts` |
| PRD-DEP-03 | `docs/PRD.md` | Dependência | Hierarquia de erros existente: novos erros do módulo de webhooks devem seguir o padrão AppError | CODIGO | `src/shared/errors/app-error.ts` |
| PRD-DEP-04 | `docs/PRD.md` | Dependência | Middleware de erro centralizado: já trata AppError, ZodError e PrismaClientKnownRequestError; deve funcionar com erros do novo módulo sem alteração | CODIGO | `src/middlewares/error.middleware.ts` |
| PRD-DEP-05 | `docs/PRD.md` | Dependência | Logger Pino existente: reutilizado para logs do worker e do módulo de webhooks | CODIGO | `src/shared/logger/index.ts` |
| PRD-DEP-06 | `docs/PRD.md` | Dependência | Estrutura modular existente: novo módulo segue o padrão de organização (controller, service, repository, routes, schemas) | CODIGO | `src/modules/orders/` |
| PRD-DEP-07 | `docs/PRD.md` | Dependência | Configuração de banco de dados reutilizada via createPrismaClient; worker usa instância própria de PrismaClient | CODIGO | `src/config/database.ts` |
| PRD-DEP-08 | `docs/PRD.md` | Dependência | Nenhuma nova dependência de pacote: criptografia via node:crypto e HTTP dispatch via fetch nativo | TRANSCRICAO | [09:30] Larissa |
| PRD-PRAZO-01 | `docs/PRD.md` | Prazo | Estimativa de 3 sprints incluindo revisão de segurança da Sofia | TRANSCRICAO | [09:46] Larissa |
| PRD-PRAZO-02 | `docs/PRD.md` | Prazo | Atlas Comercial espera entrega até fim do trimestre; Marcos confirmará prazo com os clientes | TRANSCRICAO | [09:45] Marcos, [09:47] Marcos |
| PRD-PRAZO-03 | `docs/PRD.md` | Prazo | Revisão de segurança: Sofia reserva pelo menos 2 dias úteis para revisar código de autenticação e geração de credenciais antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-CTX-01 | `docs/PRD.md` | Contexto | Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram formalmente notificações de mudança de status de pedidos | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | `docs/PRD.md` | Contexto | Atlas Comercial sinalizou possibilidade de migração para concorrente caso funcionalidade não seja entregue até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | `docs/PRD.md` | Contexto | Requisito de tempo real dos clientes é modesto: qualquer coisa abaixo de 10 segundos é aceitável | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-04 | `docs/PRD.md` | Contexto | Webhooks são exclusivamente outbound (plataforma envia, clientes recebem) | TRANSCRICAO | [09:02] Marcos, [09:03] Sofia |
| PRD-CTX-05 | `docs/PRD.md` | Contexto | Máquina de estados do pedido (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED) é o modelo existente no sistema | CODIGO | `prisma/schema.prisma` |

---

## Cobertura

Total de itens rastreados: **209 linhas** cobrindo os 6 ADRs, o RFC, o FDD e o PRD.

Para cada ADR foram mapeados:
- A decisão principal
- Detalhes específicos da decisão
- Restrições de contexto (oriundas da transcrição ou do código)
- Alternativas consideradas e descartadas
- Trade-offs aceitos

Para o RFC foram mapeados:
- Contexto e problema (6 itens)
- Alternativas consideradas (2 itens)
- Decisões referenciadas (11 itens)
- Questões em aberto (4 itens)
- Riscos (5 itens)
- Trade-offs (2 itens)

Para o FDD foram mapeados:
- Fluxos detalhados (8 itens)
- Requisitos funcionais (8 itens)
- Contratos de API e payload (4 itens)
- Requisitos não funcionais (5 itens)
- Erros previstos (6 itens)
- Integrações com sistema existente (14 itens)
- Decisões de implementação (4 itens)

Para o PRD foram mapeados:
- Requisitos funcionais (11 itens)
- Requisitos não funcionais (13 itens)
- Itens fora de escopo (7 itens)
- Decisões de produto (6 itens)
- Riscos (7 itens)
- Dependências (8 itens)
- Prazo e cronograma (3 itens)
- Contexto e motivação (5 itens)

## Observações

1. Os timestamps indicam o momento aproximado em que a decisão ou argumento foi apresentado na reunião. Quando a decisão foi consolidada ao longo de múltiplas falas, o timestamp referencia a fala que introduziu o argumento principal.

2. Itens com fonte `CODIGO` referenciam artefatos existentes no repositório que validam o contexto ou a restrição mencionada nos ADRs.

3. A alternativa "JWT assinado por request" documentada no ADR-003 (seção Alternativas Consideradas, item 2) **não possui correspondência explícita na transcrição** da reunião. Essa alternativa aparece apenas como conteúdo do ADR sem ter sido discutida verbalmente na call. Por esse motivo, não foi incluída na tabela de rastreabilidade, pois não é possível atribuir-lhe uma fonte verificável sem inventar dados.

4. A alternativa "Sem mecanismo de deduplicação" documentada no ADR-004 (seção Alternativas Consideradas, item 2) também **não possui correspondência direta na transcrição**. Ninguém sugeriu entregar at-least-once sem event_id na reunião. Aplica-se o mesmo critério do item anterior.

5. As decisões do RFC (prefixo `RFC-DEC-`) referenciam os ADRs correspondentes no corpo do documento `docs/RFC.md`. Cada decisão do RFC aponta para o ADR que a detalha, mantendo a rastreabilidade entre os dois documentos. As linhas do RFC complementam as dos ADRs sem duplicar o detalhamento -- o RFC registra a decisão no nível da proposta técnica, enquanto os ADRs detalham justificativa e alternativas.

6. O item RFC-CTX-06 referencia o arquivo `src/modules/orders/order.service.ts` como evidência de que a transação atômica de mudança de status (método `changeStatus`) existe no código atual, validando o contexto descrito na seção 2 do RFC.

7. Os itens do FDD com fonte `CODIGO` referenciam arquivos do repositório que serão alterados ou reutilizados durante a implementação. Cada referência foi verificada contra o estado atual do código: os caminhos de arquivo, números de linha e estruturas mencionadas no FDD correspondem ao que existe no repositório.

8. Os erros `WEBHOOK_SECRET_REQUIRED` e `WEBHOOK_ENDPOINT_INACTIVE` listados na matriz de erros do FDD (seção 7) **não possuem correspondência direta na transcrição**. Foram definidos no FDD como cenários defensivos de implementação sem que tenham sido discutidos na reunião. Por esse motivo, não foram incluídos na tabela de rastreabilidade.

9. Os itens FDD-FLUXO e FDD-INT complementam as decisões registradas nos ADRs e no RFC, detalhando **como** implementar (ponto de inserção no código, assinatura de função, estrutura de arquivos) sem repetir o **porquê** (justificativa e alternativas), que permanece nos documentos anteriores.

10. Os itens do PRD com prefixo `PRD-RF-` e `PRD-RNF-` registram os requisitos no nível de produto (o que o sistema deve fazer e com quais restrições). Os detalhes de como implementar cada requisito estão nos documentos posteriores (RFC para a proposta técnica, FDD para a especificação de implementação). O PRD não detalha endpoints, payloads ou estrutura de banco.

11. As métricas de sucesso do PRD (seção 4) -- taxa de entrega de 95% na primeira tentativa, 99% com retries e redução de 80% de polling -- **não foram discutidas na reunião**. O próprio PRD registra explicitamente que são estimativas de produto a serem validadas após o lançamento. Por esse motivo, não foram incluídas como linhas na tabela de rastreabilidade.

12. O critério de go/no-go "48 horas de execução em staging" mencionado na seção 12 do PRD **não foi discutido na reunião**. O próprio PRD registra que é recomendação de produto a ser validada com a tech lead. Não foi incluído na tabela por não possuir fonte verificável.

13. Os itens `PRD-DEP-*` com fonte `CODIGO` referenciam componentes existentes no repositório dos quais a feature depende. Cada caminho de arquivo foi verificado contra o estado atual do código e corresponde a um artefato real.
