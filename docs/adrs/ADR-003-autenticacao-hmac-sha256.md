# ADR-003: Autenticação de entregas de webhook com HMAC-SHA256 e secret por endpoint

## Status

Aceita

## Contexto

O sistema expõe eventos contendo dados de pedidos para endpoints externos fora da nossa infraestrutura. O cliente consumidor precisa garantir duas propriedades ao receber uma requisição:

1. **Autenticidade** — a requisição realmente partiu da nossa plataforma e não de um terceiro mal-intencionado.
2. **Integridade** — o payload não foi adulterado em trânsito.

Além disso, houve incidentes anteriores em que um cliente vazou a secret em logs de aplicação. Isso evidencia a necessidade de isolar o impacto de um vazamento ao menor escopo possível, evitando que uma única secret comprometida exponha todos os endpoints da plataforma.

## Decisão

Adotar **HMAC-SHA256** como mecanismo de autenticação para todas as entregas de webhook, com as seguintes regras:

1. **Assinatura sobre o body** — o worker de dispatch calcula `HMAC-SHA256(secret, request_body)` e envia o resultado no header `X-Signature`. O cliente recalcula o HMAC no seu lado e compara com o valor recebido.

2. **Secret única por endpoint** — cada registro de webhook na tabela de configuração armazena `url`, `secret`, `customer_id` e `active`. Não existe secret global da plataforma. Se uma secret vazar, apenas aquele endpoint específico fica comprometido.

3. **Rotação com grace period de 24 horas** — o cliente pode solicitar uma nova secret via API. Ao rotacionar, a secret anterior permanece válida em paralelo por 24 horas, permitindo que o cliente migre sem perder entregas. Após esse período, a secret antiga é invalidada automaticamente.

4. **TLS obrigatório** — a URL do webhook deve utilizar HTTPS. Tentativas de cadastrar URLs com esquema HTTP são rejeitadas com erro de validação no momento do registro.

## Alternativas Consideradas

### 1. Secret global única para toda a plataforma

Uma única secret compartilhada entre todos os endpoints cadastrados. Descartada porque o vazamento de uma única chave comprometeria **todos** os clientes simultaneamente, tornando o raio de impacto inaceitável. O incidente anterior de vazamento em log de aplicação reforça esse risco.

### 2. JWT assinado por request

Gerar um token JWT para cada entrega, assinado com chave privada da plataforma, permitindo que o cliente valide com a chave pública. Descartada porque:

- Adiciona complexidade desnecessária (gerenciamento de par de chaves, distribuição de chave pública, parsing de claims).
- Não oferece vantagem prática para o cenário de webhook outbound, onde o cliente já possui uma secret compartilhada e precisa apenas validar autenticidade e integridade.
- HMAC-SHA256 é o padrão de mercado para webhooks (adotado por Stripe, GitHub, Shopify) e toda linguagem relevante possui bibliotecas maduras para verificação.

## Consequências

### Positivas

- **Isolamento de impacto** — vazamento de uma secret compromete apenas um endpoint, não a plataforma inteira.
- **Padrão de mercado** — clientes já possuem familiaridade e bibliotecas prontas para verificar HMAC-SHA256, reduzindo atrito de integração.
- **Rotação sem downtime** — o grace period de 24 horas garante que o cliente pode migrar para a nova secret sem perder entregas durante a transição.
- **Proteção contra man-in-the-middle** — TLS obrigatório protege o payload em trânsito; HMAC garante que mesmo em cenário de TLS comprometido, adulteração é detectável.

### Negativas

- **Complexidade de rotação** — o sistema precisa manter até duas secrets válidas simultaneamente por endpoint durante o grace period, exigindo lógica adicional no worker de dispatch (tentar validar com ambas as secrets) e um job para expirar secrets antigas após 24 horas.
- **Armazenamento seguro de secrets** — cada endpoint armazena uma secret sensível no banco de dados. É necessário garantir criptografia at-rest e controle de acesso rigoroso à tabela de configuração de webhooks.
- **Responsabilidade do cliente** — a segurança depende de o cliente implementar corretamente a verificação do HMAC e proteger a secret no seu ambiente. Não há como a plataforma impedir mau uso no lado do consumidor.
