# RFC: Sistema de Webhooks para Notificação de Mudança de Status de Pedidos

## Metadados

| Campo      | Valor                                      |
|------------|--------------------------------------------|
| Autor      | Larissa (Tech Lead)                        |
| Status     | Em Revisão                                 |
| Data       | 2026-07-05                                 |
| Revisores  | Larissa, Marcos, Bruno, Diego, Sofia       |

---

## Resumo executivo (TL;DR)

Propomos adicionar um sistema de webhooks outbound ao OMS para notificar clientes B2B em tempo quase real sobre mudanças de status de pedidos. A abordagem utiliza o padrão Transactional Outbox no MySQL existente, um worker separado para despacho assíncrono e autenticação HMAC-SHA256 por endpoint, oferecendo garantia at-least-once sem introdução de infraestrutura externa.

---

## Contexto e problema

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) solicitaram formalmente notificações em tempo real sobre mudanças de status dos pedidos. Atualmente, esses clientes fazem polling periódico no endpoint `GET /orders` para detectar alterações, o que torna a integração lenta e custosa para eles. A Atlas Comercial sinalizou possibilidade de migração para concorrente caso a funcionalidade não seja entregue até o fim do trimestre.

O requisito de negócio é que a notificação chegue em menos de 10 segundos após a transição de status. O fluxo é exclusivamente outbound: o sistema envia notificações para os clientes, sem receber webhooks de volta.

No estado atual do repositório, a transação de mudança de status (`changeStatus` em `src/modules/orders/order.service.ts`) já é pesada: atualiza o status do pedido, insere registro em `order_status_history` e manipula estoque (`debitStock`/`replenishStock`), tudo dentro de uma única transação Prisma. Qualquer acoplamento síncrono com sistemas externos nesse ponto comprometeria a disponibilidade e a latência da operação.

---

## Proposta técnica

A solução se baseia em quatro pilares arquiteturais, cada um respaldado por uma decisão formal (ADRs):

### 1. Transactional Outbox no MySQL existente

A inserção do evento de webhook ocorre dentro da mesma transação SQL que já realiza a mudança de status do pedido. Isso garante atomicidade: se a transação principal sofrer rollback, o evento é descartado junto; se commitou, o evento está persistido. Não há janela de inconsistência entre a mudança de status e o registro do evento.

O payload do evento é armazenado como snapshot no momento da inserção, refletindo o estado exato do pedido no instante da transição. Eventos são filtrados na inserção: se nenhum webhook configurado para o cliente deseja receber aquele status, a linha não é criada.

Decisão detalhada em [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### 2. Worker de despacho em processo separado

Um processo Node.js independente da API faz polling na tabela outbox a cada 2 segundos, processa eventos pendentes em batch e dispara as chamadas HTTP para os endpoints dos clientes. O isolamento de processos garante que crash ou restart da API não interrompa o despacho, e vice-versa. A latência no pior caso (2 segundos) atende com folga o requisito de 10 segundos.

A ordenação de eventos por pedido é garantida implicitamente enquanto houver um único worker. Essa é uma limitação conhecida e aceita para o escopo atual.

Decisão detalhada em [ADR-005](adrs/ADR-005-worker-polling-separado.md).

### 3. Autenticação HMAC-SHA256 com secret por endpoint

Cada requisição de webhook é assinada com HMAC-SHA256 usando uma secret única por endpoint. O cliente valida autenticidade e integridade do payload do lado dele. Secrets são rotacionáveis com grace period de 24 horas para migração. URLs de webhook devem obrigatoriamente usar HTTPS.

Decisão detalhada em [ADR-003](adrs/ADR-003-autenticacao-hmac-sha256.md).

### 4. Garantia at-least-once com retry e DLQ

O sistema oferece garantia at-least-once: um evento pode ser entregue mais de uma vez, e o cliente deduplica pelo `event_id` (UUID único por evento). Em caso de falha de entrega, aplica-se backoff exponencial com 5 tentativas (cobertura de aproximadamente 15 horas). Após esgotar as tentativas, o evento é movido para uma Dead Letter Queue persistida em tabela separada, permitindo reprocessamento manual via endpoint administrativo restrito à role ADMIN.

Decisões detalhadas em [ADR-004](adrs/ADR-004-garantia-at-least-once.md) e [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md).

### Integração com a arquitetura existente

O módulo de webhooks seguirá os mesmos padrões já estabelecidos no projeto: estrutura de módulos em `src/modules/`, classes de erro estendendo `AppError` com prefixo `WEBHOOK_`, logger Pino, validação com Zod, autenticação e autorização via middlewares existentes. Nenhuma biblioteca ou framework novo será introduzido. Decisão detalhada em [ADR-006](adrs/ADR-006-reuso-de-padroes-existentes.md).

---

## Alternativas consideradas

### 1. Despacho síncrono dentro do service de pedidos

Disparar a chamada HTTP ao endpoint do cliente diretamente dentro do método `changeStatus`, na mesma requisição.

**Descartada porque:** um endpoint de cliente lento travaria a transação de mudança de status, impactando todos os demais pedidos. Além disso, se o endpoint estivesse fora do ar, seria necessário decidir entre dar rollback na mudança de status (inaceitável para o negócio) ou perder o evento (inconsistente). Essa abordagem acoplaria a disponibilidade do OMS à disponibilidade de sistemas externos.

### 2. Fila externa com Redis Streams ou broker de mensagens

Utilizar Redis Streams, RabbitMQ ou outro broker para intermediar os eventos entre a API e o worker de despacho.

**Descartada porque:** exigiria provisionar e manter infraestrutura adicional (Redis Cluster ou broker), desproporcional para o tamanho do time. O padrão outbox no MySQL existente oferece a mesma garantia de consistência transacional com menor custo operacional e sem adicionar dependências externas ao sistema.

### 3. Trigger do MySQL para notificação reativa

Utilizar triggers do banco de dados para notificar o worker imediatamente quando um novo evento fosse inserido na outbox, eliminando a latência de polling.

**Descartada porque:** MySQL não possui mecanismo nativo de notificação a processos externos (como o `NOTIFY/LISTEN` do PostgreSQL). Triggers MySQL executam apenas SQL; para avisar um processo externo, seria necessário improvisar soluções frágeis como escrever em arquivo ou chamar endpoints, introduzindo acoplamento e pontos de falha.

---

## Questões em aberto

1. **Rate limiting de envio para clientes:** se um cliente tiver muitos pedidos mudando de status em curto intervalo, o sistema pode bombardeá-lo com dezenas de chamadas simultâneas. Foi discutido na reunião ([09:38-09:39]) e decidiu-se observar o comportamento em produção antes de implementar qualquer mecanismo de throttling. Permanece como ponto a ser avaliado após o lançamento.

2. **Notificação por email em caso de falhas recorrentes:** Marcos levantou a possibilidade de avisar o cliente quando seu webhook falha repetidamente (por exemplo, 3 falhas consecutivas). Foi explicitamente colocado fora do escopo desta fase ([09:37]), ficando para avaliação futura após medição de impacto.

3. **Estratégia de limpeza da tabela outbox:** Diego mencionou arquivamento de linhas entregues após 30 dias ([09:08]), mas o tema foi marcado como fora do escopo desta feature. A estratégia de retenção e purge precisa ser definida antes que a tabela cresça significativamente em produção.

4. **Escalabilidade para múltiplos workers:** a garantia de ordenação de eventos por pedido depende de haver um único worker. Caso haja necessidade futura de escalar horizontalmente, será necessário implementar particionamento por `order_id` ou lock pessimista ([09:13]). Não há plano definido para quando ou como isso seria feito.

---

## Impacto e riscos

### Impacto no sistema existente

- **Ponto de integração crítico:** o método `changeStatus` do `OrderService` será modificado para inserir eventos na outbox dentro da transação existente. Essa é a única alteração em código de domínio já existente; o restante é aditivo (novo módulo, novo processo).
- **Carga adicional no banco:** cada mudança de status com webhooks configurados gera uma escrita extra na mesma transação. Em cenários de pico, isso aumenta marginalmente a contenção no MySQL.
- **Novo processo em produção:** o worker de despacho é um processo Node.js adicional que precisa ser gerenciado (deploy, monitoramento, restart).

### Riscos identificados

| Risco | Probabilidade | Impacto | Mitigação |
|-------|--------------|---------|-----------|
| Acúmulo de eventos na outbox por falha do worker | Baixa | Alto | Monitoramento do tamanho da fila; alertas quando eventos pendentes ultrapassarem threshold |
| Cliente com endpoint permanentemente indisponível gerando carga de retry | Média | Baixo | Política de 5 tentativas com DLQ; evento sai da fila ativa após ~15h |
| Vazamento de secret de webhook | Baixa | Médio | Secret por endpoint (isolamento de blast radius); rotação via API com grace period |
| Transação de mudança de status mais lenta devido à escrita na outbox | Baixa | Baixo | Escrita simples de uma linha; índice na tabela outbox otimizado para leitura do worker |

---

## Decisões relacionadas

Este RFC é respaldado pelas seguintes decisões arquiteturais, que detalham o racional e as alternativas de cada escolha:

- [ADR-001: Padrão Outbox no MySQL existente](adrs/ADR-001-outbox-no-mysql.md) -- consistência transacional entre mudança de status e registro de evento
- [ADR-002: Política de retry com backoff exponencial e DLQ](adrs/ADR-002-retry-com-backoff-e-dlq.md) -- estratégia de resiliência para falhas de entrega
- [ADR-003: Autenticação HMAC-SHA256 com secret por endpoint](adrs/ADR-003-autenticacao-hmac-sha256.md) -- segurança e autenticidade das notificações
- [ADR-004: Garantia at-least-once com X-Event-Id](adrs/ADR-004-garantia-at-least-once.md) -- semântica de entrega e mecanismo de deduplicação
- [ADR-005: Worker em processo separado com polling de 2 segundos](adrs/ADR-005-worker-polling-separado.md) -- modelo de execução do despacho assíncrono
- [ADR-006: Reuso dos padrões existentes do projeto](adrs/ADR-006-reuso-de-padroes-existentes.md) -- consistência arquitetural com a codebase atual
