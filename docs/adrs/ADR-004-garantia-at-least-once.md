# ADR-004: Garantia at-least-once com X-Event-Id

**Status:** Aceita
**Data:** 2026-07-05
**Decisores:** Diego (Engenheiro Sênior), Sofia (Engenheira de Segurança), Marcos (Product Manager)

---

## Contexto

O sistema de webhooks precisa definir qual garantia de entrega oferece aos clientes: o evento pode ser entregue zero vezes (best-effort), exatamente uma vez (exactly-once) ou pelo menos uma vez (at-least-once). Essa definição impacta diretamente a complexidade da implementação e as responsabilidades do cliente na integração.

Com o padrão outbox e o mecanismo de retry (ADR-001 e ADR-002), existe a possibilidade real de que o mesmo evento seja entregue mais de uma vez. Por exemplo: o worker envia o evento, o endpoint do cliente processa com sucesso mas a resposta HTTP se perde por timeout de rede. O worker, sem receber confirmação, retenta o envio.

## Decisão

Adotar **garantia at-least-once** com mecanismo de deduplicação baseado em **X-Event-Id**.

### Mecanismo

- Cada evento inserido na tabela outbox receberá um **UUID único** (`event_id`), gerado no momento da inserção.
- Esse UUID será enviado no header **`X-Event-Id`** de cada requisição HTTP ao endpoint do cliente.
- Em caso de retry, o mesmo `event_id` será reenviado, permitindo que o cliente identifique duplicatas.
- A **responsabilidade de deduplicação é do cliente**: ele deve armazenar os `event_id` recebidos e ignorar eventos já processados.

### Headers enviados

Além do `X-Event-Id`, cada requisição de webhook incluirá:
- `X-Signature` (HMAC-SHA256, conforme ADR-003)
- `X-Timestamp` (timestamp do envio, para detecção de replay attack)
- `X-Webhook-Id` (identificador do endpoint de webhook configurado)
- `Content-Type: application/json`

## Alternativas Consideradas

### 1. Garantia exactly-once

Garantir que cada evento seja entregue exatamente uma vez, sem duplicatas.

**Rejeitada porque:**
- Exigiria coordenação bidirecional entre o nosso sistema e o sistema do cliente (por exemplo, protocolo de confirmação em duas fases ou idempotency keys com verificação pré-envio).
- A complexidade de implementação e operacional é desproporcionalmente alta para o benefício ([09:25] Diego).
- Nenhum sistema de webhooks de referência no mercado oferece exactly-once: Stripe, GitHub e outros adotam at-least-once com event_id ([09:25] Diego).

### 2. Sem mecanismo de deduplicação

Oferecer at-least-once sem enviar identificador único, deixando o cliente sem meio de detectar duplicatas.

**Rejeitada porque:**
- O cliente não teria como diferenciar uma entrega original de um retry, podendo processar o mesmo evento múltiplas vezes com efeitos colaterais indesejados (por exemplo, enviar email duplicado ao consumidor final).
- Transferir responsabilidade de deduplicação sem fornecer ferramenta para tal é uma integração frágil.

## Consequências

### Positivas

- **Simplicidade de implementação:** o servidor apenas garante persistência e retry, sem necessidade de coordenação complexa com o cliente.
- **Padrão reconhecido:** clientes que já integram com Stripe, GitHub ou similares já possuem lógica de deduplicação por event_id, reduzindo a curva de integração.
- **Resiliência:** nenhum evento é perdido silenciosamente; na pior das hipóteses, o cliente recebe duplicatas que pode identificar e ignorar.

### Negativas

- **Responsabilidade transferida ao cliente:** o cliente precisa implementar deduplicação do lado dele. Se não o fizer, pode processar eventos duplicados. Marcos se comprometeu a documentar isso de forma destacada no portal de desenvolvedor ([09:26]).
- **Possibilidade de duplicatas:** em cenários de falha de rede ou timeout, o cliente pode receber o mesmo evento mais de uma vez, o que exige que seus sistemas sejam preparados para isso.

### Trade-off explícito

Aceita-se transferir a responsabilidade de deduplicação para o cliente em troca de simplicidade de implementação no servidor e aderência ao padrão consolidado de mercado para sistemas de webhooks.
