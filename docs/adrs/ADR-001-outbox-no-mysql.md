# ADR-001: Padrão Outbox no MySQL existente

**Status:** Aceita
**Data:** 2026-07-05
**Decisores:** Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno), Larissa (Tech Lead)

---

## Contexto

O sistema de gestão de pedidos precisa notificar clientes B2B via webhooks quando o status de um pedido muda. A transação de mudança de status (`changeStatus` em `src/modules/orders/order.service.ts`) já é pesada: atualiza a tabela `orders`, insere um registro em `order_status_history` e, dependendo da transição, decrementa ou incrementa `stock_quantity` dos produtos via `debitStock`/`replenishStock`. Tudo isso ocorre dentro de uma única transação Prisma (`this.prisma.$transaction`).

O problema central é: como registrar o evento de webhook de forma que seja atomicamente consistente com a mudança de status, sem degradar a latência nem acoplar a disponibilidade do sistema à disponibilidade dos endpoints dos clientes?

## Decisão

Adotar o **padrão Transactional Outbox** utilizando o MySQL existente do projeto. Dentro da mesma transação SQL que já atualiza `orders`, insere em `order_status_history` e manipula estoque, será inserida uma linha em uma nova tabela `webhook_outbox` contendo o evento a ser despachado.

O payload do evento será renderizado como snapshot no momento da inserção (não será montado posteriormente a partir do `order_id`), garantindo que o evento reflita o estado exato do pedido no instante da transição de status.

Os identificadores da tabela outbox seguirão o padrão UUID já estabelecido no projeto, conforme confirmado na reunião ([09:51]).

A filtragem de eventos ocorrerá no momento da inserção na outbox: se nenhum webhook configurado para o `customer_id` em questão deseja receber notificação para aquele status específico, a linha não será inserida na outbox ([09:34]).

### Ponto de integração no código existente

O método `changeStatus` em `src/modules/orders/order.service.ts` (linhas 126-179) receberá uma chamada adicional dentro do bloco `this.prisma.$transaction(async (tx) => { ... })`, após a inserção no histórico e antes do retorno. Essa chamada inserirá o evento na outbox utilizando o mesmo `tx` (transaction client) da transação corrente, garantindo atomicidade.

## Alternativas Consideradas

### 1. Despacho síncrono dentro do service de orders

Disparar a chamada HTTP ao endpoint do cliente diretamente dentro do método `changeStatus`, na mesma requisição.

**Rejeitada porque:**
- Um cliente com endpoint lento travaria a transação de mudança de status, impactando todos os demais pedidos ([09:04] Bruno).
- Se o endpoint do cliente estivesse fora do ar, seria necessário decidir entre dar rollback na mudança de status (inaceitável) ou perder o evento (inconsistente) ([09:04] Bruno).
- Acoplaria a disponibilidade do sistema à disponibilidade de sistemas externos.

### 2. Fila externa (Redis Streams ou similar)

Utilizar Redis Streams, RabbitMQ ou outro broker de mensagens para intermediar os eventos.

**Rejeitada porque:**
- Exigiria subir e manter infraestrutura adicional (Redis Cluster ou broker), o que é desproporcional para o tamanho do time ([09:07] Diego).
- Introduziria complexidade operacional sem benefício proporcional, dado que o MySQL existente já atende ao requisito.
- O padrão outbox no banco relacional existente oferece a mesma garantia de consistência transacional com menor custo operacional.

## Consequências

### Positivas

- **Consistência transacional garantida:** se a mudança de status sofrer rollback, o evento da outbox é descartado junto; se commitou, o evento está persistido. Não há janela de inconsistência.
- **Nenhuma infraestrutura adicional:** reutiliza o MySQL e o Prisma Client já presentes no projeto (`prisma/schema.prisma`, `src/config/database.ts`).
- **Desacoplamento temporal:** o despacho HTTP acontece de forma assíncrona pelo worker, sem impactar a latência da API.

### Negativas

- **Carga adicional no MySQL:** cada mudança de status que tenha webhooks configurados gera uma escrita extra na tabela outbox dentro da mesma transação. Em cenários de pico, isso aumenta a contenção no banco.
- **Necessidade de manutenção da tabela outbox:** linhas processadas acumulam na tabela e precisam de uma estratégia de arquivamento ou limpeza periódica (mencionado como fora de escopo desta feature, com sugestão de 30 dias - [09:08] Diego).

### Trade-off explícito

Aceita-se uma carga adicional no banco de dados relacional em troca de eliminar a necessidade de infraestrutura externa e garantir consistência transacional forte entre a mudança de status e o registro do evento.
