# ADR-003: Autenticação HMAC-SHA256 com secret por endpoint

**Status:** Aceita
**Data:** 2026-07-05
**Decisores:** Sofia (Engenheira de Segurança), Bruno (Engenheiro Pleno), Diego (Engenheiro Sênior)

---

## Contexto

O sistema de webhooks envia dados de pedidos (status, valores, identificadores) para endpoints externos controlados pelos clientes B2B. É necessário que o cliente consiga verificar duas propriedades de cada requisição recebida:

1. **Autenticidade:** a requisição foi realmente originada pelo nosso sistema.
2. **Integridade:** o payload não foi adulterado em trânsito.

Além disso, caso uma secret seja comprometida (situação já observada anteriormente quando um cliente vazou a secret em logs de aplicação - [09:22] Diego), o impacto deve ser limitado e a recuperação deve ser simples.

## Decisão

Adotar **HMAC-SHA256** como mecanismo de assinatura dos payloads de webhook, com as seguintes características:

### Assinatura

- O corpo (body) de cada requisição HTTP será assinado com HMAC-SHA256 usando a secret do endpoint.
- A assinatura será enviada no header `X-Signature`.
- O cliente valida a assinatura do lado dele recalculando o HMAC sobre o body recebido com a secret compartilhada.

### Secret por endpoint

- Cada endpoint de webhook configurado terá uma **secret única**, associada ao registro de configuração (url + secret + customer_id + estado ativo).
- Não haverá secret global da plataforma. Se uma secret vazar, somente aquele endpoint específico é comprometido ([09:21] Sofia).

### Rotação de secrets

- O cliente poderá solicitar a rotação da secret via API.
- Ao rotacionar, a secret anterior permanecerá válida por um **grace period de 24 horas**, permitindo que o cliente migre seus sistemas.
- Após 24 horas, a secret antiga será invalidada automaticamente.
- Durante o grace period, o sistema assinará com a secret nova, mas o cliente poderá validar com qualquer uma das duas.

### TLS obrigatório

- A URL do webhook deverá obrigatoriamente usar HTTPS. URLs com esquema HTTP serão rejeitadas com erro de validação no momento do cadastro ([09:23] Sofia).

## Alternativas Consideradas

### 1. Secret global da plataforma

Uma única secret compartilhada entre todos os endpoints de todos os clientes.

**Rejeitada porque:**
- O vazamento de uma única secret comprometeria todos os clientes simultaneamente ([09:21] Sofia).
- Não permite rotação granular: rotacionar a secret global exigiria que todos os clientes atualizassem seus sistemas ao mesmo tempo.

### 2. JWT assinado por request

Gerar um JWT para cada requisição de webhook, contendo claims com os dados do evento.

**Rejeitada porque:**
- Adiciona complexidade desnecessária: o JWT carrega overhead de encoding (header, payload, signature) quando o único objetivo é validar autenticidade e integridade do body.
- HMAC-SHA256 é o padrão de mercado para webhooks (Stripe, GitHub, Shopify) e todas as bibliotecas de clientes já suportam nativamente ([09:20] Sofia).
- O JWT exigiria que o cliente também validasse claims de expiração e audience, adicionando lógica que não agrega valor neste caso de uso.

## Consequências

### Positivas

- **Padrão de mercado:** HMAC-SHA256 é amplamente adotado em webhooks, facilitando a integração por parte dos clientes que já possuem bibliotecas para essa validação.
- **Isolamento de comprometimento:** o vazamento de uma secret afeta apenas um endpoint específico de um cliente.
- **Rotação sem downtime:** o grace period de 24 horas permite que o cliente migre sem perder eventos durante a transição.
- **Simplicidade:** a implementação é direta tanto no lado do servidor (assinar) quanto no lado do cliente (verificar).

### Negativas

- **Armazenamento de duas secrets durante rotação:** durante o grace period, o sistema precisa manter e gerenciar duas secrets ativas por endpoint, adicionando complexidade ao modelo de dados e à lógica de assinatura.
- **Responsabilidade do cliente:** o cliente é responsável por implementar a verificação do HMAC corretamente. Se ele ignorar a validação, a segurança do lado dele não está garantida.

### Trade-off explícito

Aceita-se a complexidade de gerenciar secrets individuais por endpoint e o mecanismo de grace period em troca de isolamento de segurança granular e aderência ao padrão de mercado.
