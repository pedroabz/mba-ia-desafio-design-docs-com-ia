# ADR-004: Garantia de entrega at-least-once com X-Event-Id para deduplicação client-side

## Status

Aceita

## Contexto

O sistema de webhooks precisa definir qual garantia de entrega oferece aos clientes consumidores. As três opções clássicas são: at-most-once (pode perder eventos), at-least-once (pode duplicar eventos) e exactly-once (nunca perde nem duplica).

O mecanismo de retry com backoff exponencial (ADR-002) implica que um mesmo evento pode ser entregue mais de uma vez — por exemplo, quando o servidor do cliente recebe o payload e processa com sucesso, mas a resposta HTTP se perde na rede antes de chegar ao nosso worker. Nesse cenário, o worker interpreta como falha e reenvia.

Para que o cliente consiga identificar duplicatas, é necessário fornecer um identificador único e estável por evento. O projeto já utiliza UUID como padrão de identificação (`@default(uuid()) @db.Char(36)` em `prisma/schema.prisma`), o que torna natural gerar um UUID no momento em que o evento é inserido na tabela `webhook_outbox`.

## Decisão

Adotar garantia de entrega **at-least-once** com header `X-Event-Id` para deduplicação no lado do cliente.

Funcionamento:

1. Quando um evento é inserido na `webhook_outbox` (dentro da transação do `changeStatus`), um UUID é gerado como chave primária do registro. Esse UUID é o **event_id**.
2. Em toda requisição HTTP de dispatch do webhook, o header `X-Event-Id` é incluído com o valor do UUID do evento.
3. Se o mesmo evento for entregue mais de uma vez (por conta de retry), o valor de `X-Event-Id` será idêntico em todas as tentativas.
4. O cliente é responsável por armazenar os `event_id` já processados e ignorar duplicatas.

A responsabilidade de deduplicação fica no cliente. Esse modelo é padrão de mercado — adotado por Stripe, GitHub Webhooks e outros provedores de larga escala.

## Alternativas Consideradas

### 1. Exactly-once delivery — coordenação bidirecional

Garantir entrega exatamente uma vez exigiria um protocolo de two-phase commit ou confirmação explícita entre nosso sistema e o cliente (ex.: o cliente confirma via callback que processou o evento, e só então marcamos como entregue). Descartada porque:

- Aumenta drasticamente a complexidade de implementação em ambos os lados.
- Exige que todos os clientes implementem o protocolo de confirmação, elevando a barreira de integração.
- Falhas parciais no protocolo de coordenação podem gerar estados inconsistentes difíceis de resolver.
- A equipe é pequena e o benefício marginal sobre at-least-once com deduplicação não justifica o custo.

### 2. Sem mecanismo de deduplicação

Entregar at-least-once sem fornecer um identificador estável ao cliente. Mais simples de implementar (nenhum header adicional), porém:

- Clientes não teriam como diferenciar um evento novo de uma reentrega, tornando impossível a deduplicação no lado deles.
- Poderia causar efeitos colaterais indesejados (ex.: cobrar um cliente duas vezes, enviar e-mail duplicado).
- Transfere todo o risco de duplicata para o cliente sem oferecer ferramenta para mitigá-lo.

## Consequências

### Positivas

- **Simplicidade de implementação** — reusa o UUID já gerado como PK da `webhook_outbox`; basta incluir o header no dispatch HTTP.
- **Padrão de mercado** — clientes familiarizados com Stripe/GitHub já conhecem o modelo e sabem implementar deduplicação.
- **Resiliência** — o sistema pode reenviar livremente sem risco de causar side-effects no cliente, desde que o cliente implemente dedup.
- **Baixo acoplamento** — não exige protocolo de coordenação bidirecional; o contrato é unidirecional (nosso sistema envia, cliente deduplica).

### Negativas

- **Responsabilidade no cliente** — clientes que não implementarem deduplicação por `X-Event-Id` podem sofrer processamento duplicado. Essa responsabilidade deve ser documentada de forma proeminente no portal do desenvolvedor.
- **Armazenamento no cliente** — o cliente precisa manter um registro de `event_id` já processados, o que consome armazenamento e exige estratégia de expurgo.
- **Não elimina duplicatas** — o sistema conscientemente aceita que duplicatas ocorrerão; apenas fornece o mecanismo para o cliente tratá-las.
