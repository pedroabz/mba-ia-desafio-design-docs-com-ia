---
description: Gerar um documento RFC (Request for Comments) a partir da análise do repositório e de um arquivo de transcrição
tags: [project, rfc]
---

Utilize o agente `design-docs-generator` para gerar um RFC (Request for Comments).

Analise o repositório, o arquivo de transcrição e os ADRs já gerados em `docs/adrs/` para gerar um RFC contendo apenas aquilo que é relevante sobre as decisões tomadas na reunião. Referencie os ADRs existentes com links quando relevante.

O documento deve se ater ao escopo de um RFC: descrever como pretendemos resolver o problema, quais alternativas foram consideradas, quais decisões arquiteturais estão sendo propostas e quais pontos permanecem em aberto.

O RFC deve operar em nível de arquitetura. Explique a proposta, seus principais componentes e a forma como ela se integra à arquitetura existente, sem descer ao detalhamento de implementação reservado ao FDD.

**OUTPUT**

Arquivo `.md` na pasta `docs`.

Este arquivo deve conter a proposta técnica da solução, no formato de um documento submetido à equipe para revisão.

O RFC opera em nível de arquitetura: apresenta a abordagem escolhida, as alternativas que foram colocadas na mesa e as questões deixadas em aberto.

Este documento deve ser conciso (entre 2 e 4 páginas).

O documento deve incluir as seguintes seções:

### Seções obrigatórias

- Metadados (autor, status, data, revisores); utilize os participantes da reunião como revisores
- Resumo executivo (TL;DR) da proposta
- Contexto e problema
- Proposta técnica (visão geral da solução, sem descer ao detalhe de implementação do FDD)
- Alternativas consideradas (pelo menos 2 alternativas reais discutidas e descartadas na reunião, cada uma com o trade-off que levou ao descarte)
- Questões em aberto (pelo menos 2 pontos levantados na reunião e não decididos ou adiados)
- Impacto e riscos
- Decisões relacionadas (links para os ADRs correspondentes)

### Regras

- Não inclua detalhamento de implementação. Isso deve ficar a cargo do FDD.
- O RFC não deve duplicar o detalhamento do FDD. Ele responde **"o que propomos e por quê"**; o **"como construir"** em detalhe fica no FDD.
- Caso alguma alternativa ou questão em aberto exigida pela estrutura do documento não esteja presente na transcrição, não invente conteúdo. Registre explicitamente que aquela informação não foi identificada durante a análise da reunião.