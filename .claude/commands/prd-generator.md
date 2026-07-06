---
description: Gerar um PRD a partir da análise do repositório e de um arquivo de transcrição
tags: [project, prd]
---

Utilize o agente `design-docs-generator` para gerar um PRD.

Analise o repositório, o arquivo de transcrição, e utilize `docs/RFC.md`, `docs/FDD.md` e os ADRs em `docs/adrs/` como contexto para gerar um PRD consolidado com aquilo que é relevante sobre as decisões tomadas na reunião.

O documento deve se ater ao escopo de um PRD: problema, motivação, público-alvo, escopo, objetivos, métricas de sucesso, riscos e critérios de aceitação.

Não gere documentação de outro escopo.

Não inclua detalhes implementacionais que pertençam a RFC ou FDD, como:
- nomes de arquivos
- nomes de classes/funções
- endpoints detalhados
- payloads completos
- estrutura de tabelas
- fluxos internos de implementação

Caso uma decisão técnica apareça na reunião, traduza-a para linguagem de negócio e somente se ela for relevante para escopo, risco, dependência ou trade-off do produto. O leitor do PRD é alguém de produto — não deve precisar saber o que é uma transação, uma outbox, um registro atômico ou um backoff exponencial.

### Exemplos de linguagem adequada vs. inadequada para o PRD

| Inadequado (técnico demais) | Adequado (linguagem de produto) |
|---|---|
| "Inserção na tabela webhook_outbox dentro da transação do changeStatus" | "O sistema garante que a notificação é registrada junto com a mudança de status, sem risco de perda" |
| "Registro atômico do evento junto à mudança de status" | "Se o status do pedido mudou, a notificação foi registrada. Se houve falha, nenhuma notificação é gerada" |
| "Worker em polling de 2 segundos lê a outbox e despacha via HTTP" | "Um processo dedicado envia as notificações automaticamente, com latência máxima de poucos segundos" |
| "Retry com backoff exponencial: 1m, 5m, 30m, 2h, 12h" | "Em caso de falha, o sistema reenviar automaticamente com intervalos crescentes, cobrindo um período de até 15 horas" |
| "HMAC-SHA256 sobre o body com secret por endpoint" | "Cada notificação é assinada com uma credencial exclusiva do cliente, permitindo que ele valide a autenticidade e integridade da mensagem" |
| "Garantia at-least-once com X-Event-Id para dedup client-side" | "O sistema garante que toda notificação é entregue pelo menos uma vez. Cada envio inclui um identificador único para que o cliente possa tratar eventuais duplicatas" |
| "PrismaClient separado no worker, mesma DATABASE_URL" | Não mencionar no PRD — detalhe de implementação sem relevância para produto |
| "Endpoint POST /webhooks com body { url, events[] }" | "API para cadastro de webhook com URL de destino e seleção dos eventos de interesse" |

Seções obrigatórias:

- Resumo e contexto da feature
- Problema e motivação
- Público-alvo e cenários de uso
- Objetivos e métricas de sucesso
- Escopo (incluso e fora de escopo)
- Requisitos funcionais
- Requisitos não funcionais
- Decisões e trade-offs principais
- Dependências
- Riscos e mitigação
- Critérios de aceitação
- Estratégia de testes e validação