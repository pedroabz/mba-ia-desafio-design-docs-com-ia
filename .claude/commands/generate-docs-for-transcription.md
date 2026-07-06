---

description: Gerar pacote completo de design docs a partir da análise do repositório e de um arquivo de transcrição
tags: [project, docs, orchestration]
------------------------------------

Você é um **Senior Technical Architect** responsável por coordenar a geração completa de documentação técnica e de produto para uma nova feature.

**THINKING MODE:** Use **ultrathink** para esta tarefa. Não se preocupe com uso de tokens durante a análise; uma análise completa leva a decisões arquiteturais melhores.

# INPUT

Arquivo de transcrição referente à reunião de tomada de decisão da nova feature a ser documentada.

# Objetivo

Gerar um pacote completo de documentação técnica e de produto a partir da análise do repositório e do arquivo de transcrição.

Cada documento responde a perguntas diferentes.

PRD:

* Por que existe?
* Qual problema resolve?

RFC:

* Como pretendemos resolver?
* Quais alternativas existem?

ADR:

* Por que escolhemos esta decisão?

FDD:

* Como implementar exatamente?

Tracker:

* De onde veio cada requisito, decisão, restrição ou trade-off?

# ORQUESTRAÇÃO DE AGENTES

Você orquestra agentes especializados. Cada agente possui diretrizes detalhadas em `.claude/agents/<nome-do-agente>.md`.

## Idioma

Todos os documentos devem ser escritos em **português brasileiro correto**, com todos os acentos e caracteres especiais (ç, ã, õ, é, ê, á, à, ó, ô, í, ú, ü). Nunca omita diacríticos — palavras como "notificação", "transação", "índice", "conteúdo", "módulo", "código" devem estar grafadas corretamente. Esta regra se aplica a **todas as etapas** e deve ser reforçada em cada chamada de agente.

## Regras de Coordenação

* Toda a comunicação passa por você, o coordenador.
* Os agentes NÃO se comunicam diretamente entre si.
* Analise as dependências entre as tarefas ANTES de iniciar os agentes.
* Execute os agentes sequencialmente quando houver dependências.
* Use os outputs de cada etapa como input das etapas seguintes.
* Não altere código da aplicação (`src/`, `prisma/`, `tests/`, configurações). A entrega é puramente documental.
* Se qualquer etapa falhar ou gerar documento incompleto, corrija antes de avançar para a etapa seguinte.

# Fluxo obrigatório de execução

**Esse fluxo NÃO é opcional. Você deve segui-lo obrigatoriamente.**

## Etapa 1: Gerar ADRs

**Nome:** Gerar ADRs
**Descrição:** Identificar e registrar as decisões arquiteturais principais antes dos demais documentos. As decisões formam o esqueleto do "como implementar" e serão referenciadas pelos documentos seguintes.
**Comando:** `.claude/commands/adr-generator.md`
**Input:** arquivo de transcrição recebido neste comando pelo agente coordenador e o repositório como contexto de validação
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** arquivos em `docs/adrs/` no formato `ADR-NNN-titulo-em-kebab-case.md`

## Etapa 2: Tracker (parcial — ADRs)

**Nome:** Gerar Tracker parcial
**Descrição:** Gerar a primeira versão do Tracker cobrindo os itens registrados nas ADRs.
**Comando:** `.claude/commands/tracker-generator.md`
**Input:** arquivo de transcrição, conjunto de ADRs em `docs/adrs/` e repositório
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/TRACKER.md`

## Etapa 3: Gerar RFC

**Nome:** Gerar RFC
**Descrição:** Consolidar a proposta técnica da solução em cima das decisões registradas nas ADRs, incluindo abordagem escolhida, alternativas consideradas, trade-offs e questões em aberto. Referenciar os ADRs já escritos.
**Comando:** `.claude/commands/rfc-generator.md`
**Input:** arquivo de transcrição recebido neste comando pelo agente coordenador e os ADRs gerados na etapa 1
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/RFC.md`

## Etapa 4: Tracker (atualizar — ADRs + RFC)

**Nome:** Atualizar Tracker
**Descrição:** Atualizar o Tracker incorporando os itens do RFC.
**Comando:** `.claude/commands/tracker-generator.md`
**Input:** arquivo de transcrição, `docs/RFC.md`, conjunto de ADRs em `docs/adrs/` e repositório
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/TRACKER.md` atualizado

## Etapa 5: Gerar FDD

**Nome:** Gerar FDD
**Descrição:** Com as decisões formalizadas nas ADRs e a proposta consolidada no RFC, gerar o documento detalhado de implementação, incluindo fluxos, contratos públicos, erros, resiliência, observabilidade e integração com o sistema existente.
**Comando:** `.claude/commands/fdd-generator.md`
**Input:** arquivo de transcrição, `docs/RFC.md` e os ADRs gerados nas etapas anteriores
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/FDD.md`

## Etapa 6: Tracker (atualizar — ADRs + RFC + FDD)

**Nome:** Atualizar Tracker
**Descrição:** Atualizar o Tracker incorporando os itens do FDD.
**Comando:** `.claude/commands/tracker-generator.md`
**Input:** arquivo de transcrição, `docs/RFC.md`, `docs/FDD.md`, conjunto de ADRs em `docs/adrs/` e repositório
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/TRACKER.md` atualizado

## Etapa 7: Gerar PRD

**Nome:** Gerar PRD
**Descrição:** Com RFC, FDD e ADRs em mãos, produzir o PRD como consolidação de alto nível focada em problema, público-alvo, objetivos, métricas, escopo, riscos e critérios de aceitação.
**Comando:** `.claude/commands/prd-generator.md`
**Input:** arquivo de transcrição recebido neste comando pelo agente coordenador, `docs/RFC.md`, `docs/FDD.md` e os ADRs como contexto
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/PRD.md`

## Etapa 8: Tracker (completo)

**Nome:** Atualizar Tracker
**Descrição:** Atualizar o Tracker incorporando os itens do PRD, completando a cobertura.
**Comando:** `.claude/commands/tracker-generator.md`
**Input:** arquivo de transcrição, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, conjunto de ADRs em `docs/adrs/` e repositório
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/TRACKER.md` completo

O Tracker deve seguir o formato obrigatório:

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| -- | --------- | ---- | ----------------- | ----- | ----------- |

Cobertura mínima: pelo menos 80% dos itens identificáveis nos documentos devem possuir linha correspondente no Tracker.

## Etapa 9: Revisar PRD

**Nome:** Revisar PRD
**Descrição:** Revisão dedicada do PRD para garantir que está em linguagem de produto, sem detalhes técnicos de implementação. O PRD é o documento mais propenso a sair com linguagem low-level (nomes de arquivos, classes, endpoints, payloads, estrutura de tabelas). Tudo isso deve ser reescrito em linguagem de alto nível ou removido, pois pertence ao RFC ou ao FDD.
**Input:** `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`
**Agente:** `.claude/agents/doc-reviewer.md`
**Output esperado:** PRD revisado em linguagem de produto, sem detalhes técnicos.

**Regras de revisão do PRD:**
* Remover ou reescrever qualquer menção a nomes de arquivos, classes, funções, endpoints ou payloads.
* Decisões técnicas devem ser traduzidas para linguagem de negócio. O leitor do PRD é alguém de produto — não deve precisar saber o que é uma transação, uma outbox, um registro atômico ou um backoff exponencial. Exemplos:
  - "Inserção na tabela webhook_outbox dentro da transação do changeStatus" → "O sistema garante que a notificação é registrada junto com a mudança de status, sem risco de perda"
  - "Worker em polling de 2 segundos" → "Um processo dedicado envia as notificações automaticamente, com latência máxima de poucos segundos"
  - "HMAC-SHA256 sobre o body com secret por endpoint" → "Cada notificação é assinada com uma credencial exclusiva do cliente, permitindo validar autenticidade e integridade"
  - "Endpoint POST /webhooks com body { url, events[] }" → "API para cadastro de webhook com URL de destino e seleção dos eventos de interesse"
  - "PrismaClient separado no worker, mesma DATABASE_URL" → Não mencionar no PRD — detalhe de implementação sem relevância para produto
* Verificar que requisitos funcionais descrevem **o que** o sistema faz, não **como** faz.
* Se algum conteúdo for removido do PRD, verificar que ele já existe no RFC ou FDD. Se não existir, movê-lo para o documento adequado.

## Etapa 10: Revisão final do pacote

**Nome:** Revisão final
**Descrição:** Revisar o pacote completo de documentação contra os critérios do desafio e corrigir inconsistências finais.
**Input:** `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, arquivos em `docs/adrs/` e transcrição original
**Agente:** `.claude/agents/doc-reviewer.md`
**Output esperado:** pacote final consistente, rastreável e pronto para entrega.

A revisão final deve verificar obrigatoriamente:

* `docs/PRD.md` contém todas as seções obrigatórias.
* `docs/RFC.md` contém todas as seções obrigatórias.
* `docs/FDD.md` contém todas as seções obrigatórias, incluindo "Integração com o sistema existente".
* `docs/adrs/` contém entre 5 e 8 ADRs.
* Cada ADR possui Status, Contexto, Decisão, Alternativas Consideradas e Consequências.
* Nenhum requisito, decisão ou restrição contradiz a transcrição ou o código.
* Nenhum arquivo de código mencionado é inexistente.
* O PRD não contém detalhamento que pertença ao FDD.
* O RFC não duplica o detalhamento do FDD.
* O FDD está acionável o suficiente para iniciar implementação.

## Etapa 11: Atualizar Tracker (final)

**Nome:** Atualizar Tracker após revisões
**Descrição:** Atualizar o Tracker para refletir todas as correções feitas nas etapas 9 e 10. Esta etapa garante que o Tracker está consistente com o estado final dos documentos.
**Comando:** `.claude/commands/tracker-generator.md`
**Input:** arquivo de transcrição, `docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/TRACKER.md`, conjunto de ADRs em `docs/adrs/` e repositório
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/TRACKER.md` final e consistente.

**Regras:**
* Remover linhas cujo conteúdo deixou de existir nos documentos após as revisões.
* Atualizar resumos de itens que foram reescritos.
* Preservar IDs e linhas não afetadas.
* Verificar que o Tracker cobre pelo menos 80% dos itens identificáveis.
* Verificar que pelo menos 70% das linhas têm Fonte = `TRANSCRICAO` com timestamp válido `[hh:mm] Nome`.
* Verificar que pelo menos 5 linhas têm Fonte = `CODIGO` com caminho de arquivo real.
* `docs/TRACKER.md` segue o formato obrigatório.

# Manifesto de execução

Durante a execução, mantenha um arquivo `docs/manifest.md` com o progresso das etapas.

Formato obrigatório:

| Etapa | Documento | Status | Input usado | Output gerado | Observações |
| ----- | --------- | ------ | ----------- | ------------- | ----------- |

Atualize o manifesto ao final de cada etapa.

# Resultado final esperado

Ao concluir todas as etapas, devem existir os seguintes arquivos:

```text
docs/
├── PRD.md
├── RFC.md
├── FDD.md
├── TRACKER.md
├── manifest.md
└── adrs/
    ├── ADR-001-*.md
    ├── ADR-002-*.md
    ├── ADR-003-*.md
    ├── ADR-004-*.md
    ├── ADR-005-*.md
    └── ... até no máximo ADR-008-*.md
```

Ao final, apresente um resumo curto contendo:

* arquivos gerados;
* principais correções feitas na revisão;
* pendências, caso existam.
