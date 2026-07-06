**Sobre o desafio**
Este repositório faz parte de um desafio do MBA full cycle de Engenharia de Software com IA, na disciplina de Design Docs.
O objetivo era utilizar a transcrição de uma reunião técnica a respeito de uma nova feature em um sistema de OMS (Order Management System) para gerar Design Docs acerca da mesma.
Para tal, era obrigatória a utilização de ferramentas de inteligência artificial a fim de gerar os documentos de forma não manual.

**Ferramentas de IA utilizadas**

| Ferramenta | Papel |
|---|---|
| **Claude Code** | Ferramenta principal. Utilizado com dois agentes customizados (um gerador e um revisor) e seis comandos (um orquestrador + cinco geradores por tipo de documento) para automatizar a produção e garantir consistência entre os documentos. |
| **ChatGPT** | Utilizado como segunda opinião na revisão dos próprios comandos e agentes — erros de português, formatação e clareza das instruções. |

**Workflow Utilizado**

Ao invés de um fluxo manual de geração de cada arquivo individualmente, foi criado um workflow automatizado composto por **comandos** (prompts especializados por tipo de documento) e **agentes** (personas com diretrizes transversais).

### Componentes

#### 1. Comando Orquestrador — `generate-docs-for-transcription`

Responsável por orquestrar o fluxo completo de geração dos documentos. Recebe a transcrição como input e dispara os comandos específicos para cada tipo de documento, na ordem correta de dependências, utilizando o agente relevante para cada etapa.

O fluxo segue 11 etapas sequenciais:

| Etapa | O que faz | Output |
|-------|-----------|--------|
| 1 | **Gerar ADRs** — Identificar e registrar as decisões arquiteturais principais a partir da transcrição. As decisões formam o esqueleto que os documentos seguintes referenciam. | `docs/adrs/ADR-001` a `ADR-006` |
| 2 | **Tracker (parcial)** — Gerar a primeira versão do Tracker com os itens das ADRs. | `docs/TRACKER.md` |
| 3 | **Gerar RFC** — Consolidar a proposta técnica em cima das decisões das ADRs, incluindo alternativas descartadas e questões em aberto. | `docs/RFC.md` |
| 4 | **Atualizar Tracker** — Incorporar os itens do RFC ao Tracker existente. | `docs/TRACKER.md` atualizado |
| 5 | **Gerar FDD** — Com decisões e proposta consolidadas, gerar o documento de implementação detalhado (fluxos, contratos, erros, integração com o código). | `docs/FDD.md` |
| 6 | **Atualizar Tracker** — Incorporar os itens do FDD. | `docs/TRACKER.md` atualizado |
| 7 | **Gerar PRD** — Com RFC, FDD e ADRs em mãos, produzir o PRD como consolidação de alto nível (problema, público, escopo, métricas, riscos). | `docs/PRD.md` |
| 8 | **Atualizar Tracker** — Incorporar os itens do PRD, completando a cobertura. | `docs/TRACKER.md` completo |
| 9 | **Revisar PRD** — Revisão dedicada para garantir que o PRD está em linguagem de produto, sem detalhes técnicos de implementação (nomes de arquivos, classes, endpoints, payloads). | PRD revisado |
| 10 | **Revisão final** — Verificar o pacote completo contra os critérios de aceite do desafio e corrigir inconsistências finais. | Pacote revisado |
| 11 | **Atualizar Tracker (final)** — Refletir no Tracker todas as correções feitas nas etapas 9 e 10, garantindo consistência final. | `docs/TRACKER.md` final |

#### 2. Agente gerador de documentos — `design-docs-generator`

Agente responsável por gerar os documentos seguindo diretrizes específicas:
- Obrigatoriedade das seções definidas em cada comando.
- Respeito às fronteiras entre documentos (ex.: garantir que o PRD não contivesse conteúdo pertencente ao FDD).
- Utilização da língua portuguesa com caracteres especiais.

#### 3. Agente revisor de documentos — `doc-reviewer`

Observou-se que, mesmo com as instruções apresentadas nos comandos e no agente gerador, alguns erros ainda eram cometidos. Para mitigar isso, criou-se um agente dedicado a revisar e corrigir os documentos gerados. As garantias verificadas por esse agente eram:

- Cada documento respeita seu nível de abstração.
- Não há duplicação indevida entre PRD, RFC, FDD, ADRs e Tracker.
- Não há requisitos, decisões ou restrições inventadas.
- Toda informação relevante é rastreável à transcrição ou ao código.
- Os documentos estão consistentes entre si.
- A entrega atende aos critérios obrigatórios do desafio.

#### 4. Comandos individuais por documento

Cada tipo de documento possui um comando dedicado (`prd-generator`, `rfc-generator`, `fdd-generator`, `adr-generator`, `tracker-generator`) que define as seções obrigatórias, os inputs esperados e as regras de escopo específicas daquele documento.

- **Prompts customizados**: 
Alguns dos blocos importantes de Prompt:
1 - A orquestração do fluxo e como cada etapa é chamada (qual input, output, agente e função)

2 - Prompt para exemplificar outputs esperados e não esperados de um PRD, a fim de resolver o problema do PRD tendendo a ficar técnico demais

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



### Desafios encontrados

| Desafio | Problema observado | Solução adotada |
|---|---|---|
| **Caracteres especiais** | Documentos gerados sem acentos do português | Instrução explícita adicionada tanto no agente gerador quanto no comando orquestrador como regra transversal |
| **Fronteira entre documentos** | PRD gerado com conteúdo demasiadamente técnico (pertencente ao FDD) | Instrução explícita de escopo em cada comando, além de uma etapa dedicada no orquestrador (Etapa 9 — Garantir Escopo) para verificar e reescrever em linguagem de produto qualquer ponto que violasse a fronteira |
| **Duplicidade de escopo** | Mesma decisão escrita de forma muito similar em dois documentos (ex.: PRD duplicando conteúdo do FDD) | Etapa de revisão e deduplicação pelo agente revisor, com regra de preservar o conteúdo no documento mais adequado e remover do outro |
