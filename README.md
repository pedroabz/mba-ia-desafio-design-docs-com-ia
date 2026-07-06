# Da Reunião ao Documento: Design Docs Gerados por IA

## Sobre o desafio

Este repositório faz parte de um desafio do MBA full cycle de Engenharia de Software com IA, na disciplina de Design Docs.
O objetivo era utilizar a transcrição de uma reunião técnica a respeito de uma nova feature em um sistema de OMS (Order Management System) para gerar Design Docs acerca da mesma.
Para tal, era obrigatória a utilização de ferramentas de inteligência artificial a fim de gerar os documentos de forma não manual.

## Ferramentas de IA utilizadas

| Ferramenta | Papel |
|---|---|
| **Claude Code** | Ferramenta principal. Utilizado com dois agentes customizados (um gerador e um revisor) e seis comandos (um orquestrador + cinco geradores por tipo de documento) para automatizar a produção e garantir consistência entre os documentos. |
| **ChatGPT** | Utilizado como segunda opinião na revisão dos próprios comandos e agentes com objetivo de corrigir erros de português, formatação e dar clareza às instruções. |

## Workflow adotado

Ao invés de um fluxo manual de geração de cada arquivo individualmente, foi criado um workflow automatizado composto por **comandos** (prompts especializados por tipo de documento) e **agentes** (personas com diretrizes transversais).

### Comando Orquestrador — `generate-docs-for-transcription`

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

### Agente gerador de documentos — `design-docs-generator`

Agente responsável por gerar os documentos seguindo diretrizes específicas:
- Obrigatoriedade das seções definidas em cada comando.
- Respeito às fronteiras entre documentos (ex.: garantir que o PRD não contivesse conteúdo pertencente ao FDD).
- Utilização da língua portuguesa com caracteres especiais.

### Agente revisor de documentos — `doc-reviewer`

Observou-se que, mesmo com as instruções apresentadas nos comandos e no agente gerador, alguns erros ainda eram cometidos. Para mitigar isso, criou-se um agente dedicado a revisar e corrigir os documentos gerados. As garantias verificadas por esse agente eram:

- Cada documento respeita seu nível de abstração.
- Não há duplicação indevida entre PRD, RFC, FDD, ADRs e Tracker.
- Não há requisitos, decisões ou restrições inventadas.
- Toda informação relevante é rastreável à transcrição ou ao código.
- Os documentos estão consistentes entre si.
- A entrega atende aos critérios obrigatórios do desafio.

### Comandos individuais por documento

Cada tipo de documento possui um comando dedicado (`prd-generator`, `rfc-generator`, `fdd-generator`, `adr-generator`, `tracker-generator`) que define as seções obrigatórias, os inputs esperados e as regras de escopo específicas daquele documento.

## Prompts customizados

### Prompt 1 — Orquestração do fluxo de geração (comando `generate-docs-for-transcription`)

Este prompt define o papel de coordenador, a ordem de execução das 11 etapas e como cada agente é invocado. Cada etapa especifica comando, input, agente e output esperado, garantindo que as dependências entre documentos sejam respeitadas (ex.: ADRs antes do RFC, RFC antes do FDD).

```markdown
Você é um **Senior Technical Architect** responsável por coordenar a geração
completa de documentação técnica e de produto para uma nova feature.

## Regras de Coordenação
* Toda a comunicação passa por você, o coordenador.
* Os agentes NÃO se comunicam diretamente entre si.
* Analise as dependências entre as tarefas ANTES de iniciar os agentes.
* Execute os agentes sequencialmente quando houver dependências.
* Use os outputs de cada etapa como input das etapas seguintes.

## Etapa 1: Gerar ADRs
**Comando:** `.claude/commands/adr-generator.md`
**Input:** arquivo de transcrição e o repositório como contexto de validação
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** arquivos em `docs/adrs/` no formato `ADR-NNN-titulo-em-kebab-case.md`

## Etapa 3: Gerar RFC
**Comando:** `.claude/commands/rfc-generator.md`
**Input:** arquivo de transcrição e os ADRs gerados na etapa 1
**Agente:** `.claude/agents/design-docs-generator.md`
**Output esperado:** `docs/RFC.md`

(... demais etapas seguem o mesmo padrão, totalizando 11 etapas sequenciais)
```

### Prompt 2 — Few-shot prompting para linguagem de produto no PRD (comando `prd-generator`)

O PRD tendia a sair com linguagem técnica de implementação. Para corrigir, foi adicionado um bloco de exemplos e contra-exemplos diretamente no prompt, ensinando ao modelo a traduzir decisões técnicas para linguagem de negócio:

```markdown
Não inclua detalhes implementacionais que pertençam a RFC ou FDD, como:
- nomes de arquivos, classes ou funções
- endpoints detalhados, payloads completos
- estrutura de tabelas ou fluxos internos de implementação

Caso uma decisão técnica apareça na reunião, traduza-a para linguagem de
negócio e somente se ela for relevante para escopo, risco, dependência
ou trade-off do produto.

### Exemplos de linguagem adequada vs. inadequada para o PRD

| Inadequado (técnico demais)                                  | Adequado (linguagem de produto)                              |
|--------------------------------------------------------------|--------------------------------------------------------------|
| "Inserção na tabela webhook_outbox dentro da transação       | "O sistema garante que a notificação é registrada junto com  |
|  do changeStatus"                                            |  a mudança de status, sem risco de perda"                    |
| "Worker em polling de 2 segundos lê a outbox e despacha      | "Um processo dedicado envia as notificações automaticamente,  |
|  via HTTP"                                                   |  com latência máxima de poucos segundos"                     |
| "HMAC-SHA256 sobre o body com secret por endpoint"           | "Cada notificação é assinada com uma credencial exclusiva    |
|                                                              |  do cliente, permitindo validar autenticidade e integridade" |
| "PrismaClient separado no worker, mesma DATABASE_URL"        | Não mencionar no PRD — detalhe de implementação              |
```

## Iterações e ajustes

Ao longo do processo, seguem 5 pontos que merecem maior destaque:

1. **De comandos avulsos para workflow orquestrado:** A primeira abordagem foi criar comandos autossuficientes — cada um gerava um documento de forma independente. Rapidamente isso se mostrou inviável: qualquer ajuste (ex.: uma regra de escopo ou formatação) precisava ser replicado manualmente em todos os comandos; gerar os documentos um a um era tedioso; e manter a separação de responsabilidades entre documentos acabava sendo uma tarefa artesanal. Essa foi a razão principal de migrar para o workflow com entrada única — um comando orquestrador que coordena agentes e comandos especializados, com dependências explícitas entre etapas.

2. **Caracteres especiais:** Os documentos gerados saíam sem acentos nem caracteres especiais do português (ç, ã, é, etc.). Adicionei a instrução de idioma no agente gerador, mas não foi suficiente — a IA continuava omitindo diacríticos. Só foi resolvido quando a instrução foi replicada nos **dois agentes** (gerador e revisor) **e** no comando orquestrador simultaneamente. Lição: instruções críticas precisam estar em todos os pontos da cadeia, não apenas em um.

3. **PRD altamente técnico:** O PRD gerado continha linguagem de implementação (nomes de tabelas, endpoints, classes) que o tornava incompreensível para alguém de produto. A solução envolveu duas ações: (a) adicionar o bloco de few-shot prompting com exemplos e contra-exemplos no comando `prd-generator` (Prompt 2 acima), e (b) criar uma etapa dedicada no workflow (Etapa 9) exclusivamente para revisão do PRD pelo agente revisor, focada em rebaixar linguagem técnica para linguagem de negócio.

4. **Duplicidade de escopo entre documentos:** A mesma decisão aparecia com nível de detalhe muito similar em dois documentos (ex.: PRD duplicando conteúdo que pertencia ao FDD, RFC repetindo detalhes do FDD). A solução foi adicionar regras explícitas no agente revisor para identificar duplicações e preservar o conteúdo no documento mais adequado, removendo do outro. Além disso, o agente gerador recebeu a regra de "nunca antecipar conteúdo pertencente a documentos posteriores da cadeia".

5. **Tracker com cobertura insuficiente:** O Tracker gerado numa única passada não atingia os 80% de cobertura exigidos. A solução foi fragmentar a geração do Tracker em múltiplas etapas incrementais (etapas 2, 4, 6, 8 e 11 do workflow), adicionando os itens de cada documento conforme ele era gerado, ao invés de tentar mapear tudo no final.

## Como navegar a entrega

Os arquivos entregues estão organizados na seguinte estrutura:

```
docs/
├── PRD.md                              # Product Requirements Document
├── RFC.md                              # Request for Comments
├── FDD.md                              # Feature Design Document
├── TRACKER.md                          # Tracker de Rastreabilidade
├── manifest.md                         # Log de execução das etapas
└── adrs/
    ├── ADR-001-outbox-no-mysql.md
    ├── ADR-002-retry-com-backoff-e-dlq.md
    ├── ADR-003-autenticacao-hmac-sha256.md
    ├── ADR-004-garantia-at-least-once.md
    ├── ADR-005-worker-polling-separado.md
    └── ADR-006-reuso-de-padroes-existentes.md
```

**Ordem sugerida de leitura:** PRD → RFC → ADRs → FDD → TRACKER

**Para reproduzir a geração dos documentos**, basta executar o comando orquestrador no Claude Code, passando o caminho do arquivo de transcrição:

```bash
/generate-docs-for-transcription TRANSCRICAO.md
```

Este comando dispara as 11 etapas sequenciais descritas no workflow, utilizando os agentes gerador e revisor para produzir e validar todos os documentos automaticamente.
