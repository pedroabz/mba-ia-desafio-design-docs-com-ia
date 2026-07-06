# RFC -- Sistema de Webhooks de Notificação de Pedidos

| Campo           | Valor                                                                 |
|-----------------|-----------------------------------------------------------------------|
| **Autor**       | Larissa (Tech Lead)                                                   |
| **Status**      | Em Revisão                                                            |
| **Data**        | 2026-07-05                                                            |
| **Revisores**   | Marcos (Product Manager), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança) |

---

## 1. Resumo executivo (TL;DR)

Propomos adicionar um sistema de webhooks outbound ao Order Management System para notificar clientes B2B sobre mudanças de status de pedidos em tempo quase real. A solução utiliza o padrão Transactional Outbox no MySQL existente, um worker independente com polling e autenticação HMAC-SHA256, garantindo consistência transacional e entrega at-least-once sem introduzir infraestrutura nova.

---

## 2. Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) solicitaram formalmente a capacidade de receber notificações quando o status de seus pedidos muda na plataforma. Atualmente, esses clientes dependem de polling sobre o endpoint `GET /orders` para detectar mudanças, o que torna a integração lenta e custosa. A Atlas Comercial sinalizou possibilidade de migração para um concorrente caso a funcionalidade não seja entregue até o fim do trimestre.

O requisito de negócio é que as notificações cheguem em menos de 10 segundos após a mudança de status. Os clientes receberão notificações (webhooks outbound), sem necessidade de enviar dados de volta.

Do ponto de vista arquitetural, o ponto crítico de integração é o método `changeStatus` do serviço de pedidos. Esse método opera dentro de uma transação SQL atômica que atualiza o status do pedido, insere registro no histórico (`order_status_history`) e ajusta estoque. Qualquer solução de notificação precisa se integrar a essa transação sem comprometer sua atomicidade nem acoplar o fluxo crítico à disponibilidade de sistemas externos.

---

## 3. Proposta técnica

### 3.1. Visão geral

A solução é composta por três componentes:

1. **Registro transacional de eventos (Outbox):** Na mesma transação SQL que altera o status do pedido, um evento é inserido em uma tabela de outbox no MySQL. O evento contém um snapshot do estado do pedido no momento da transição -- se a transação sofre rollback, o evento desaparece junto. Isso garante consistência sem coordenação distribuída ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

2. **Worker de despacho (processo separado):** Um processo Node.js independente da API consulta a outbox por polling a cada 2 segundos, seleciona eventos pendentes em ordem cronológica e realiza chamadas HTTP POST aos endpoints cadastrados. O worker compartilha a mesma base de código e banco de dados, mas roda em processo distinto para não ser afetado por restarts da API ([ADR-005](adrs/ADR-005-worker-polling-separado.md)).

3. **Módulo de configuração de webhooks:** Um novo módulo dentro da estrutura modular existente (`src/modules/`) que expõe endpoints REST para gestão de configurações de webhook, consulta de histórico de entregas e replay administrativo de eventos da DLQ.

### 3.2. Fluxo de dados (alto nível)

```
Mudança de status do pedido
        |
        v
[Transação SQL existente]
  - Atualiza status da order
  - Insere em order_status_history
  - Ajusta estoque
  - (NOVO) Insere evento na tabela outbox  <-- mesma transação
        |
        v
   COMMIT ou ROLLBACK
        |
   (se COMMIT)
        v
[Worker -- processo separado, polling 2s]
  - Lê eventos pendentes da outbox
  - Assina payload com HMAC-SHA256
  - Envia HTTP POST ao endpoint do cliente
  - Marca como entregue ou agenda retry
        |
   (se falha após 5 tentativas)
        v
[Dead Letter Queue -- tabela separada]
  - Evento preservado com motivo da falha
  - Replay manual via endpoint admin
```

### 3.3. Princípios arquiteturais da proposta

**Consistência transacional:** Evento existe se e somente se a transição de status commitou. A inserção na outbox participa da mesma transação ACID ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

**Entrega confiável (at-least-once):** Cada evento recebe um UUID único enviado no header `X-Event-Id`. O cliente é responsável por deduplicar. Exactly-once exigiria coordenação bidirecional desproporcional ao cenário ([ADR-004](adrs/ADR-004-garantia-at-least-once.md)).

**Resiliência a falhas:** Retry com backoff exponencial em 5 tentativas (1m, 5m, 30m, 2h, 12h), cobrindo uma janela de aproximadamente 15 horas. Após esgotamento, o evento é movido para uma DLQ persistida em tabela separada, com replay manual administrativo ([ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md)).

**Autenticidade e integridade:** HMAC-SHA256 com secret única por endpoint, rotacionável com grace period de 24 horas. HTTPS obrigatório ([ADR-003](adrs/ADR-003-autenticacao-hmac-sha256.md)).

**Snapshot na inserção:** Os dados do pedido são serializados no momento da transição de status, não no momento do envio. Garante que o evento reflete o estado exato de quando a mudança ocorreu.

**Ordenação por pedido:** Com worker único, eventos de um mesmo pedido são processados na ordem de criação. Não há garantia de ordenamento global -- limitação conhecida e aceita ([ADR-005](adrs/ADR-005-worker-polling-separado.md)).

### 3.4. Integração com a arquitetura existente

A proposta maximiza o reuso da infraestrutura e dos padrões já estabelecidos no projeto ([ADR-006](adrs/ADR-006-reuso-de-padroes-existentes.md)):

- **Banco de dados:** Novas tabelas no banco existente. Nenhum banco ou serviço adicional.
- **Estrutura modular:** O módulo de webhooks segue o padrão dos módulos existentes (controller, service, repository, routes, schemas).
- **Hierarquia de erros:** Novos erros seguem o padrão de erros do projeto com códigos prefixados `WEBHOOK_`.
- **Autenticação e autorização:** Reutiliza os middlewares de autenticação e autorização existentes. Endpoint de replay da DLQ exige role `ADMIN`.
- **Observabilidade:** Logger e middleware de erro centralizados, reutilizados sem alterações.
- **ORM:** O worker utiliza instância própria de conexão ao banco (por ser outro processo), apontando para o mesmo banco de dados.

Os detalhes de integração com cada componente do código estão especificados no [FDD](FDD.md).

---

## 4. Alternativas consideradas

### 4.1. Despacho síncrono dentro da transação de mudança de status

**Descrição:** Realizar a chamada HTTP ao endpoint do cliente de forma síncrona dentro do método `changeStatus`, na mesma transação que altera o status do pedido.

**Trade-offs que levaram ao descarte:**

- A transação de mudança de status já é pesada (atualiza order, insere histórico, ajusta estoque). Uma chamada HTTP bloqueante tornaria a latência dependente do tempo de resposta do cliente externo.
- Se o endpoint do cliente estiver lento ou offline, o fluxo de mudança de status ficaria travado para todos os pedidos, não apenas para os do cliente com problema.
- Não há resposta razoável para a falha do webhook dentro da transação: fazer rollback na mudança de status por causa de indisponibilidade de um sistema externo é inaceitável do ponto de vista de negócio.

**Fonte:** Bruno [09:04], Diego [09:06].

### 4.2. Fila externa (Redis Streams ou message broker)

**Descrição:** Utilizar Redis Streams, RabbitMQ ou outro sistema de mensageria como intermediário entre a transação de mudança de status e o worker de despacho.

**Trade-offs que levaram ao descarte:**

- Introduziria infraestrutura nova que o time precisaria provisionar, monitorar e manter. A equipe é pequena e o volume atual não justifica essa complexidade.
- Quebraria a garantia transacional atômica: a inserção na fila externa não participaria da transação SQL, abrindo janela para inconsistência (status muda mas evento não é registrado, ou vice-versa).
- O padrão outbox no MySQL existente resolve o problema com zero infraestrutura adicional.

**Fonte:** Larissa e Diego [09:07].

---

## 5. Questões em aberto

### 5.1. Rate limiting de saída

Se um cliente tem muitos pedidos mudando de status simultaneamente, o sistema dispara todas as notificações sem controle de vazão, podendo sobrecarregar o endpoint do cliente. A questão foi levantada por Diego [09:38] e deliberadamente adiada: o time vai observar o comportamento em produção e implementar rate limiting se/quando se tornar um problema real. Larissa confirmou: "observar e decidir depois" [09:39].

### 5.2. Notificação ao cliente sobre falhas recorrentes do webhook

Marcos perguntou [09:37] se seria possível enviar email ao cliente quando o webhook falha repetidamente. Larissa descartou explicitamente para esta fase: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." O mecanismo de alerta de falhas para o cliente permanece indefinido.

### 5.3. Archival de eventos entregues

Diego mencionou [09:08] que eventos entregues na outbox poderiam ser arquivados após 30 dias para evitar crescimento ilimitado da tabela. A política de archival/cleanup foi explicitamente colocada fora do escopo desta feature.

### 5.4. Estratégia de escalabilidade do worker

A decisão de manter single-worker é deliberada para esta fase. Diego mencionou [09:13] opções para o futuro (particionamento por `order_id`, lock pessimista), mas nenhuma foi escolhida. A estratégia concreta de escalabilidade permanece em aberto.

---

## 6. Impacto e riscos

### Impacto no sistema existente

- **Ponto de alteração crítico:** A transação de mudança de status no serviço de pedidos será estendida para incluir a inserção na outbox. Esta é a única alteração no código existente que afeta fluxo crítico. O risco é mitigado pelo fato de a inserção ser uma operação SQL simples dentro de uma transação já existente.
- **Schema do banco:** Novas tabelas e nova migration no Prisma. Nenhuma alteração em tabelas existentes.
- **Novo processo:** O worker exige configuração de deploy e monitoramento adicionais.
- **Superfície de API:** Novos endpoints REST. Nenhum endpoint existente é alterado.

### Riscos

| Risco | Severidade | Mitigação |
|-------|-----------|-----------|
| Acúmulo de eventos na outbox degrada performance do banco | Média | Índices otimizados na outbox. Archival planejado para fase futura. |
| Worker fora do ar causa acúmulo sem despacho | Alta | Eventos permanecem no banco e são processados quando o worker reinicia. Nenhum evento é perdido. Monitoramento do processo é necessário. |
| Clientes com endpoints lentos geram backlog no single-worker | Média | Timeout de 10 segundos por chamada HTTP. Limitação conhecida do single-worker. |
| Vazamento de secret de um cliente | Alta | Secret única por endpoint isola o impacto. Rotação com grace period de 24h. HTTPS obrigatório. |
| Prazo apertado (compromisso com Atlas Comercial até fim do trimestre) | Alta | Escopo bem delimitado, itens fora de escopo claramente definidos. Estimativa de 3 sprints com revisão de segurança incluída. |

---

## 7. Decisões relacionadas (ADRs)

| ADR | Decisão |
|-----|---------|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Transactional Outbox no MySQL para registro de eventos |
| [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| [ADR-003](adrs/ADR-003-autenticacao-hmac-sha256.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period |
| [ADR-004](adrs/ADR-004-garantia-at-least-once.md) | Garantia at-least-once com `X-Event-Id` para deduplicação client-side |
| [ADR-005](adrs/ADR-005-worker-polling-separado.md) | Worker em processo separado com polling a cada 2 segundos |
| [ADR-006](adrs/ADR-006-reuso-de-padroes-existentes.md) | Reuso dos padrões existentes do projeto para o módulo de webhooks |
