# ADR-006: Reuso dos padrões existentes do projeto

**Status:** Aceita
**Data:** 2026-07-05
**Decisores:** Bruno (Engenheiro Pleno), Larissa (Tech Lead)

---

## Contexto

O projeto possui padrões de código bem estabelecidos e consistentes entre os módulos existentes (`auth`, `users`, `customers`, `products`, `orders`). Ao introduzir o módulo de webhooks, é necessário decidir se ele seguirá os mesmos padrões ou adotará abordagens diferentes.

Os padrões existentes incluem:

- **Estrutura de módulos:** cada domínio em `src/modules/<domínio>/` com `controller`, `service`, `repository`, `routes` e `schemas`.
- **Tratamento de erros:** classe base `AppError` (`src/shared/errors/app-error.ts`) com `statusCode`, `errorCode` e `details`; classes especializadas como `NotFoundError`, `ConflictError`, `InvalidStatusTransitionError`, `InsufficientStockError` (`src/shared/errors/http-errors.ts`); middleware centralizado que trata `AppError`, `ZodError` e `PrismaClientKnownRequestError` (`src/middlewares/error.middleware.ts`).
- **Códigos de erro:** convenção de prefixo por domínio em upper snake case (exemplos: `INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`, `INACTIVE_PRODUCT`).
- **Autenticação e autorização:** middleware `authenticate` e função `requireRole` (`src/middlewares/auth.middleware.ts`) com suporte a roles `ADMIN` e `OPERATOR`.
- **Logging:** Pino com redação automática de campos sensíveis (`authorization`, `password`, `token`) e transporte pretty para desenvolvimento (`src/shared/logger/index.ts`).
- **Validação:** schemas Zod para validação de entrada.
- **IDs:** UUID v4 para todas as entidades (`@default(uuid())` em `prisma/schema.prisma`).

## Decisão

O módulo de webhooks **reutilizará integralmente os padrões existentes** do projeto, sem introduzir frameworks, bibliotecas ou convenções novas.

### Estrutura do módulo

- O módulo será criado em `src/modules/webhooks/` seguindo a mesma organização dos demais módulos (controller, service, repository, routes, schemas).

### Tratamento de erros

- Erros específicos do módulo serão classes que estendem `AppError` ou suas subclasses (`NotFoundError`, `ConflictError`, etc.), seguindo o padrão já estabelecido.
- Os códigos de erro utilizarão o prefixo `WEBHOOK_` (exemplos: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28-09:29] Bruno, Larissa).
- O middleware de erro centralizado (`src/middlewares/error.middleware.ts`) já trata qualquer instância de `AppError` sem necessidade de alteração.

### Autenticação e autorização

- Endpoints CRUD de configuração de webhook utilizarão o middleware `authenticate` existente.
- O endpoint de replay de DLQ exigirá role `ADMIN` via `requireRole('ADMIN')` já disponível ([09:36] Larissa, Sofia).

### Logging

- O logger Pino existente (`src/shared/logger/index.ts`) será utilizado em todo o módulo, incluindo o worker. A secret do webhook deverá ser adicionada aos paths de redação do Pino para evitar vazamento em logs.

### Validação

- Schemas Zod para validação de entrada nos endpoints do módulo, conforme padrão dos demais módulos.

## Alternativas Consideradas

### 1. Introduzir framework ou biblioteca nova para o módulo de webhooks

Adotar uma biblioteca especializada em webhooks ou um framework diferente para o worker.

**Rejeitada porque:**
- Introduziria inconsistência na codebase, obrigando desenvolvedores a conhecer dois conjuntos de convenções.
- Os padrões existentes atendem todos os requisitos do módulo de webhooks sem lacunas identificadas.
- Aumentaria a superfície de dependências e a carga de manutenção.
- A equipe já domina os padrões atuais, reduzindo o tempo de desenvolvimento ([09:29] Bruno).

## Consequências

### Positivas

- **Consistência da codebase:** qualquer desenvolvedor que conheça um módulo existente será capaz de navegar e contribuir no módulo de webhooks sem curva de aprendizado adicional.
- **Reutilização do middleware de erro:** erros do módulo de webhooks serão automaticamente tratados e formatados pelo middleware centralizado, sem necessidade de código adicional.
- **Reutilização do sistema de autorização:** o controle de acesso ao endpoint de replay de DLQ (role `ADMIN`) é implementado com uma única chamada a `requireRole`, já existente e testada.
- **Logging uniforme:** o worker e a API usam o mesmo formato de log, facilitando correlação de eventos em ferramentas de observabilidade.

### Negativas

- **Acoplamento aos padrões atuais:** se os padrões existentes tiverem limitações que só se manifestem no contexto de webhooks (por exemplo, necessidade de logs estruturados específicos do worker), será necessário adaptá-los, potencialmente impactando outros módulos.
- **Dependência do PrismaClient por processo:** o worker precisa instanciar seu próprio `PrismaClient` (mesmo banco, instância separada), o que é uma consequência natural do modelo de processo separado (ADR-005), mas exige atenção para não reutilizar a instância singleton de `src/config/database.ts`.

### Trade-off explícito

Aceita-se o acoplamento aos padrões existentes em troca de consistência total na codebase, menor curva de aprendizado e reutilização de infraestrutura já testada e funcional.
