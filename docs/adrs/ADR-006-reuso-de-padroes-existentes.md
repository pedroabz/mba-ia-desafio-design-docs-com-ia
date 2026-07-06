# ADR-006: Reusar padrões existentes do projeto para o módulo de webhooks

## Status

Aceita

## Contexto

O projeto já possui convenções bem definidas e aplicadas em todos os módulos existentes. O domínio de pedidos (`src/modules/orders/`) estabelece um padrão claro: cada módulo contém controller, service, repository, routes e schemas dentro de sua própria pasta em `src/modules/`. Além disso, a codebase conta com infraestrutura transversal madura:

- **Erros**: classe base `AppError` em `src/shared/errors/app-error.ts` com `statusCode`, `errorCode` e `details`; erros HTTP especializados em `src/shared/errors/http-errors.ts` (`NotFoundError`, `ConflictError`, etc.); códigos de erro padronizados como `INSUFFICIENT_STOCK` e `INVALID_STATUS_TRANSITION`.
- **Logger**: Pino configurado em `src/shared/logger/index.ts` com redact de campos sensíveis e nome de serviço base, utilizado em toda a aplicação.
- **Middleware de erros**: `src/middlewares/error.middleware.ts` trata de forma centralizada `AppError`, `ZodError` e `PrismaClientKnownRequestError`.
- **Middleware de autenticação**: `src/middlewares/auth.middleware.ts` com `authenticate` e `requireRole('ADMIN')`.
- **Schemas**: validação com Zod seguindo convenção de nomeação e estrutura dos módulos existentes.

A funcionalidade de webhooks precisa de controller, service, repository, rotas, schemas de validação, códigos de erro e logging — exatamente os mesmos building blocks que já existem. A questão é se devemos reusar esses padrões ou criar algo separado.

## Decisão

Máximo reuso do que já existe. O módulo de webhooks será estruturado como qualquer outro módulo do projeto:

- **Estrutura de pasta**: `src/modules/webhooks/` contendo `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts` e `webhook.schemas.ts`.
- **Worker**: entry point em `src/worker.ts`, lógica de processamento em `src/modules/webhooks/webhook.processor.ts`.
- **Erros**: classes estendendo `AppError` com códigos prefixados por `WEBHOOK_` — por exemplo `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`. O middleware de erros em `src/middlewares/error.middleware.ts` captura automaticamente sem necessidade de alteração.
- **Logger**: Pino já configurado em `src/shared/logger/index.ts`, utilizado diretamente sem adicionar dependência nova.
- **Schemas**: validação via Zod seguindo o mesmo padrão dos módulos existentes.
- **Autenticação**: rotas administrativas protegidas por `authenticate` e `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts`.

Nenhuma biblioteca, framework ou padrão arquitetural novo será introduzido para este módulo.

## Alternativas Consideradas

### 1. Criar framework/estrutura separada para webhooks

Criar uma camada ou mini-framework dedicado para webhooks com suas próprias convenções de roteamento, tratamento de erro e logging. Descartada porque fragmentaria a codebase, exigiria que a equipe mantivesse dois conjuntos de convenções em paralelo e aumentaria a carga cognitiva sem ganho proporcional.

### 2. Usar biblioteca externa de webhooks

Adotar uma biblioteca de mercado (ex: `standardwebhooks`, `svix`) para gerenciamento de webhooks. Descartada porque adicionaria uma dependência desnecessária, não se integraria naturalmente com os padrões existentes de `AppError`, Pino e middleware centralizado, e introduziria abstrações que a equipe precisaria aprender sem necessidade — já que os building blocks internos atendem todos os requisitos.

## Consequências

### Positivas

- **Curva de aprendizado zero** — qualquer desenvolvedor que já trabalhou em outro módulo do projeto entende imediatamente a estrutura de webhooks.
- **Consistência da codebase** — mesma convenção de nomeação, mesma organização de pastas, mesmos padrões de erro em toda a aplicação.
- **Middleware reutilizado sem alteração** — o error middleware em `src/middlewares/error.middleware.ts` captura erros de webhook automaticamente por serem instâncias de `AppError`.
- **Menos dependências** — nenhum pacote externo novo, reduzindo superfície de vulnerabilidade e complexidade de manutenção.
- **Logging uniforme** — Pino com a mesma configuração de redact e formato, facilitando observabilidade e busca centralizada de logs.

### Negativas

- **Acoplamento ao padrão atual** — se no futuro o time decidir migrar a arquitetura dos módulos (ex: para hexagonal ou CQRS), o módulo de webhooks precisará ser refatorado junto com os demais.
- **Sem otimizações específicas de webhook** — bibliotecas dedicadas podem oferecer funcionalidades prontas (fan-out, rate limiting por subscriber, dashboard de monitoramento) que precisariam ser implementadas manualmente caso se tornem requisitos futuros.
- **Rigidez de nomenclatura** — o prefixo `WEBHOOK_` nos códigos de erro e seguir exatamente a mesma estrutura de pastas pode parecer restritivo se o módulo crescer em complexidade, porém mantém a previsibilidade.
