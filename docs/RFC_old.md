# RFC -- Sistema de Webhooks de Notificação de Pedidos

| Campo           | Valor                                                                 |
|-----------------|-----------------------------------------------------------------------|
| **Autor**       | Larissa (Tech Lead)                                                   |
| **Status**      | Em revisão                                                            |
| **Data**        | 2026-07-04                                                            |
| **Revisores**   | Marcos (Product Manager), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança) |

---

## 1. Resumo executivo (TL;DR)

Este RFC propõe a adição de um sistema de webhooks outbound ao Order Management System (OMS) para notificar clientes B2B sobre mudanças de status de pedidos em tempo real. A solução se baseia no padrão Transactional Outbox sobre o MySQL existente, com um worker independente que despacha notificações via HTTP. O mecanismo garante consistência transacional (evento existe se e somente se a transição de status commitou), entrega confiável com semântica at-least-once, retry com backoff exponencial e Dead Letter Queue para falhas permanentes. Autenticidade é garantida via HMAC-SHA256 com secret única por endpoint. A proposta não introduz infraestrutura nova além do que já existe no projeto.

---

## 2. Contexto e problema

O PRD (documento anterior na cadeia) detalha o problema de negócio e os requisitos. Em resumo: clientes B2B dependem de polling para detectar mudanças de status, o que gera latência, custo operacional e risco de churn. Este RFC descreve **como** o sistema será arquitetado para resolver esse problema.

O ponto de partida arquitetural é o método `changeStatus` do serviço de pedidos, que hoje opera dentro de uma transação SQL atômica que atualiza o status do pedido, insere registro no histórico (`order_status_history`) e ajusta estoque. Qualquer solução de notificação precisa se integrar a essa transação sem comprometer sua atomicidade nem acoplar o fluxo crítico à disponibilidade de sistemas externos.

---

## 3. Proposta técnica

### 3.1. Visão geral da arquitetura

A solução é composta por três componentes principais:

1. **Registro transacional de eventos (Outbox):** Na mesma transação SQL que altera o status do pedido, um evento é inserido em uma tabela de outbox no MySQL. O evento contém um snapshot do estado do pedido no momento da transição. Se a transação sofre rollback, o evento desaparece junto -- garantia de consistência sem coordenação distribuída.

2. **Worker de despacho (processo separado):** Um processo Node.js independente da API consulta a tabela de outbox por polling a cada 2 segundos, seleciona eventos pendentes em ordem cronológica e realiza chamadas HTTP POST aos endpoints cadastrados pelos clientes. O worker roda como entry-point separado do projeto, compartilhando a mesma base de código e o mesmo banco de dados, mas em processo distinto.

3. **Módulo de configuração de webhooks:** Um novo módulo dentro da estrutura modular existente (`src/modules/`) que expõe endpoints REST para CRUD de configurações de webhook, consulta de histórico de entregas e replay administrativo de eventos da DLQ.

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

### 3.3. Integração com a arquitetura existente

A proposta foi deliberadamente desenhada para maximizar o reuso da infraestrutura e dos padrões já estabelecidos:

- **Banco de dados:** Novas tabelas no MySQL existente (configuração de webhook, outbox, dead letter, histórico de entregas). Nenhum banco ou serviço adicional.
- **Estrutura modular:** O módulo de webhooks segue o padrão dos módulos existentes (controller, service, repository, routes, schemas).
- **Hierarquia de erros:** Novos erros estendem `AppError` com códigos prefixados `WEBHOOK_`.
- **Autenticação e autorização:** Reutiliza `authenticate` e `requireRole` do middleware existente. Endpoint de replay da DLQ exige role `ADMIN`.
- **Validação:** Schemas Zod seguindo o padrão dos demais módulos.
- **Observabilidade:** Logger Pino já existente. Middleware de erro centralizado trata os novos erros sem alteração.
- **ORM:** Prisma Client -- uma instância separada no worker (por ser outro processo), mas usando o mesmo schema e a mesma `DATABASE_URL`.

### 3.4. Decisões arquiteturais da proposta

As seguintes decisões de arquitetura sustentam a proposta. Os detalhes completos de justificativa, alternativas descartadas e consequências de cada decisão serão formalizados nos ADRs correspondentes.

**Outbox transacional no MySQL:** Evento é persistido na mesma transação que muda o status. Garante consistência sem infraestrutura de mensageria externa.

**Worker com polling de 2 segundos:** Abordagem simples que atende o requisito de latência (< 10s). MySQL não oferece mecanismo reativo nativo comparável ao NOTIFY/LISTEN do PostgreSQL.

**Single-worker com ordenação implícita:** Processamento sequencial por timestamp garante ordenação por pedido. Limitação conhecida: não há garantia de ordering global, e escalar para múltiplos workers exigirá particionamento ou lock.

**Semântica at-least-once com idempotência delegada:** Cada evento recebe um UUID único (`event_id`). O cliente é responsável por deduplicar. Exactly-once exigiria coordenação bidirecional desproporcional ao cenário.

**Retry com backoff exponencial (5 tentativas) e DLQ:** Intervalos de 1m, 5m, 30m, 2h e 12h cobrem uma janela de ~15 horas. Após esgotamento, evento vai para tabela separada de dead letter com replay manual administrativo.

**HMAC-SHA256 com secret por endpoint:** Assinatura sobre o corpo da requisição. Secret única por webhook, rotacionável com grace period de 24 horas para migração gradual do cliente.

**Payload como snapshot na inserção:** Dados do pedido são serializados no momento da transição, não no momento do envio. Garante que o evento reflete o estado exato de quando a transição ocorreu.

**HTTPS obrigatório:** URLs de webhook devem usar TLS. Cadastros com HTTP são rejeitados na validação.

---

## 4. Alternativas consideradas

### 4.1. Despacho síncrono dentro da transação de mudança de status

**Descrição:** Ao invés de registrar o evento na outbox para processamento posterior, a chamada HTTP ao endpoint do cliente seria feita de forma síncrona dentro do método `changeStatus`, na mesma transação que altera o status.

**Trade-offs que levaram ao descarte:**
- A transação de mudança de status já é pesada (atualiza order, insere histórico, ajusta estoque). Adicionar uma chamada HTTP bloqueante tornaria a latência da transação dependente do tempo de resposta do cliente externo.
- Se o endpoint do cliente estiver lento ou offline, o fluxo de mudança de status ficaria travado para todos os pedidos, não apenas para os do cliente com problema.
- Não há resposta razoável para a falha do webhook dentro da transação: fazer rollback na mudança de status por causa de indisponibilidade de um sistema externo é inaceitável do ponto de vista de negócio.

**Fonte:** [09:04] Bruno, [09:06] Diego.

### 4.2. Fila externa (Redis Streams ou message broker)

**Descrição:** Utilizar Redis Streams, RabbitMQ ou outro sistema de mensageria como intermediário entre a transação de mudança de status e o worker de despacho, ao invés de uma tabela de outbox no MySQL.

**Trade-offs que levaram ao descarte:**
- Introduziria uma dependência de infraestrutura nova que o time precisaria operar, monitorar e manter. O time é pequeno e o volume atual não justifica essa complexidade adicional.
- Quebraria a garantia transacional atômica: a inserção na fila externa não participaria da transação SQL, abrindo janela para inconsistência (status muda mas evento não é registrado, ou vice-versa).
- O padrão outbox no MySQL existente resolve o problema com zero infraestrutura adicional e garante consistência por construção.

**Fonte:** [09:07] Larissa, Diego.

### 4.3. Mecanismo reativo no banco (triggers MySQL)

**Descrição:** Utilizar triggers do MySQL para notificar o worker de forma reativa quando um novo evento é inserido na outbox, eliminando a necessidade de polling.

**Trade-offs que levaram ao descarte:**
- MySQL não possui mecanismo de notificação de processos externos (diferente do NOTIFY/LISTEN do PostgreSQL). Triggers executam SQL, mas não conseguem avisar um processo Node.js externo de forma direta.
- Implementações alternativas (escrever em arquivo, chamar endpoint) seriam frágeis e difíceis de manter.
- Polling de 2 segundos atende o requisito de latência (< 10s) com implementação simples e previsível.

**Fonte:** [09:09] Bruno, Diego.

---

## 5. Questões em aberto

### 5.1. Rate limiting de saída

Se um cliente tem muitos pedidos mudando de status simultaneamente, o sistema dispara todas as notificações sem controle de vazão. Isso pode sobrecarregar o endpoint do cliente. A questão foi levantada por Diego ([09:38]) e deliberadamente adiada: o time vai observar o comportamento em produção e implementar rate limiting se/quando se tornar um problema real.

### 5.2. Notificação ao cliente sobre falhas recorrentes do webhook

Marcos levantou ([09:37]) a possibilidade de enviar email ao cliente quando o webhook falha repetidamente (por exemplo, 3 falhas seguidas). Larissa descartou explicitamente para esta fase, adiando para uma próxima iteração após medir o impacto da feature em produção. O mecanismo de notificação de falhas (email ou outro canal) permanece indefinido.

### 5.3. Archival de eventos entregues

Diego mencionou ([09:08]) que eventos entregues na outbox poderiam ser arquivados após 30 dias para evitar crescimento ilimitado da tabela. A política de archival/cleanup não foi definida e foi explicitamente colocada fora do escopo desta feature. O critério de retenção e o mecanismo de limpeza permanecem em aberto.

### 5.4. Estratégia de escalabilidade do worker

A decisão de manter single-worker é deliberada para esta fase, mas a estratégia concreta para escalar no futuro (particionamento por order_id, lock pessimista ou outra abordagem) não foi definida. Diego mencionou as opções ([09:13]) sem que nenhuma fosse escolhida.

---

## 6. Impacto e riscos

### Impacto no sistema existente

- **Ponto de alteração crítico:** A transação de mudança de status no serviço de pedidos precisa ser estendida para incluir a inserção na outbox. Esta é a única alteração no código existente que afeta fluxo crítico. O risco é mitigado pelo fato de a inserção ser uma operação SQL simples dentro de uma transação já existente.
- **Schema do banco:** Novas tabelas e uma nova migration no Prisma. Nenhuma alteração em tabelas existentes.
- **Novo processo:** O worker roda como processo separado, o que exige configuração de deploy e monitoramento adicionais.
- **Superfície de API:** Novos endpoints REST para CRUD, histórico e replay. Não alteram endpoints existentes.

### Riscos principais

| Risco | Severidade | Mitigação |
|-------|-----------|-----------|
| Acúmulo de eventos na outbox degrada performance do banco e do worker | Média | Índices otimizados na outbox. Archival planejado para fase futura. |
| Worker fora do ar causa acúmulo sem despacho | Alta | Eventos permanecem no banco e são processados quando o worker reinicia. Nenhum evento é perdido. Monitoramento do processo é necessário. |
| Clientes com endpoints lentos geram backlog no single-worker | Média | Timeout de 10 segundos por chamada. Limitação conhecida do single-worker. |
| Vazamento de secret de um cliente | Alta | Secret única por endpoint isola o impacto. Rotação com grace period de 24h. Transporte exclusivamente via HTTPS. |
| Prazo apertado (compromisso com Atlas Comercial até fim do trimestre) | Alta | Escopo bem delimitado, itens fora de escopo claramente definidos. Estimativa de 3 sprints aprovada. Revisão de segurança de 2 dias incluída no cronograma. |

---

## 7. Decisões relacionadas (ADRs)

As decisões arquiteturais descritas neste RFC devem ser formalizadas nos seguintes ADRs:

| ADR | Decisão |
|-----|---------|
| ADR-001 | Adoção do padrão Transactional Outbox no MySQL para registro de eventos de webhook |
| ADR-002 | Worker com polling de 2 segundos como mecanismo de despacho |
| ADR-003 | Semântica at-least-once com idempotência delegada ao cliente |
| ADR-004 | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada |
| ADR-005 | HMAC-SHA256 com secret por endpoint e rotação com grace period |
| ADR-006 | Payload como snapshot renderizado na inserção do evento |

**Nota:** Os ADRs acima ainda não foram redigidos. Os identificadores são sugestões de organização para a formalização futura.
