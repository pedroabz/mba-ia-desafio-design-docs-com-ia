# ADR-001: Usar Transactional Outbox no MySQL para entrega de eventos de webhook

## Status

Aceita

## Contexto

O método `changeStatus` em `src/modules/orders/order.service.ts` já executa uma transação SQL pesada via `this.prisma.$transaction()`, que atualiza pedidos, insere registros em `order_status_history` e decrementa estoque. A funcionalidade de webhooks exige notificar sistemas externos quando o status de um pedido muda.

Realizar chamadas HTTP de forma síncrona dentro dessa transação traria dois problemas graves:

1. **Bloqueio da transação** — se o cliente consumidor do webhook estiver lento, a mudança de status ficaria travada aguardando resposta HTTP.
2. **Impossibilidade de rollback consistente** — se o cliente estiver fora do ar após o commit da transação, não há como reverter a mudança de status; e se a chamada HTTP falhar antes do commit, a transação inteira seria revertida sem necessidade.

A equipe é pequena e a infraestrutura atual já conta com MySQL gerenciado via Prisma (`prisma/schema.prisma`). Adicionar componentes extras de infraestrutura aumentaria a complexidade operacional de forma desproporcional.

## Decisão

Adotar o padrão **Transactional Outbox** utilizando o MySQL existente.

Uma tabela `webhook_outbox` será criada com as seguintes características:

- Chave primária UUID (`@default(uuid()) @db.Char(36)`), seguindo o padrão do projeto definido em `prisma/schema.prisma`.
- Coluna `status` com valores possíveis: `pending`, `processing`, `failed`, `delivered`.
- Índice composto em `(status, created_at)` para leitura eficiente pelo worker.
- A inserção na `webhook_outbox` ocorre **dentro da mesma transação** que atualiza `orders` e insere em `order_status_history` no método `changeStatus` (`src/modules/orders/order.service.ts`).

Um worker separado lê registros com status `pending` em pequenos lotes, realiza o dispatch HTTP e atualiza o status do registro para `delivered` ou `failed`.

Garantia de consistência: se a transação do `changeStatus` fizer commit, o evento está gravado; se fizer rollback, o evento desaparece junto. Não há janela de inconsistência.

## Alternativas Consideradas

### 1. Dispatch síncrono no order service

Chamada HTTP direta dentro de `changeStatus`. Descartada porque bloquearia a transação e impossibilitaria rollback coerente em caso de falha do consumidor.

### 2. Redis Streams

Publicar evento em um stream Redis após o commit. Oferece baixa latência e backpressure nativo, porém:

- Exige infraestrutura adicional (Redis Cluster para alta disponibilidade).
- Introduz risco de inconsistência: o commit SQL pode ocorrer e a publicação no Redis falhar (ou vice-versa), já que são dois sistemas de armazenamento distintos sem transação distribuída.
- Considerada overengineering para o tamanho atual da equipe.

## Consequências

### Positivas

- **Consistência garantida** — evento e estado do pedido compartilham a mesma transação ACID.
- **Zero infraestrutura adicional** — reutiliza o MySQL já existente.
- **Simplicidade operacional** — equipe pequena consegue manter e monitorar com ferramentas que já conhece.
- **Resiliência** — falhas de rede ou indisponibilidade do consumidor não afetam a operação de mudança de status.

### Negativas

- **Polling** — o worker precisa consultar a tabela periodicamente; latência de entrega depende do intervalo de polling.
- **Carga no banco** — em cenários de altíssimo volume, a tabela `webhook_outbox` pode crescer e exigir estratégia de expurgo (delete de registros `delivered` antigos).
- **Acoplamento ao MySQL** — se no futuro o time migrar para outro banco, o mecanismo de outbox precisará ser adaptado.
