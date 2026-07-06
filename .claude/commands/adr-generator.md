---
description: Gerar um documento listando as ADRs (Architecture Decision Record) a partir da análise do repositório e de outras documentações geradas (PRD, RFC e FDD)
tags: [project, adr]
---

Utilize o agente `design-docs-generator` para gerar documentos de ADRs.

**INPUT**

- Arquivo de Transcrição
- Repositório (como contexto de validação)

**ESCOPO**

Considere a transcrição como a fonte de verdade para as decisões arquiteturais tomadas na reunião.

Considere o repositório como contexto para validar integrações e referenciar componentes existentes.

As ADRs são geradas primeiro na cadeia de documentos. Elas formam o esqueleto das decisões que serão referenciadas pelo RFC e pelo FDD.

Analise a transcrição e o repositório para gerar um conjunto de ADRs que registre as decisões arquiteturais principais discutidas na reunião.

Produza as ADRs em arquivos separados dentro de docs/adrs/, nomeados no formato ADR-NNN-titulo-em-kebab-case.md (ex: ADR-001-outbox-no-mysql.md).

Cada ADR deve seguir o formato MADR (ou variante padrão) com no mínimo as seções: Status, Contexto, Decisão, Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível), Consequências (positivas e negativas, com trade-off explícito).

Quando relevante, a ADR deve referenciar explicitamente arquivos, módulos ou padrões do código existente.

Cada ADR deve representar uma decisão arquitetural relevante.

Não crie ADRs para detalhes de implementação, como payloads, headers, códigos de erro, timeouts, endpoints ou outros aspectos pertencentes ao FDD.

**OUTPUT**

Arquivos separados dentro de docs/adrs/, nomeados no formato ADR-NNN-titulo-em-kebab-case.md (ex: ADR-001-outbox-no-mysql.md).

### Seções obrigatórias

- Status
- Contexto
- Decisão
- Alternativas Consideradas (pelo menos 1 alternativa real discutida ou plausível) -> Somente quando nenhuma alternativa estiver documentada, utilize uma alternativa plausível para contextualizar a decisão, deixando explícito que se trata de uma alternativa inferida.
- Consequências (positivas e negativas, com trade-off explícito).

### Regras

- Após finalizar o documento, revise-o e garanta que todas as seções obrigatórias existem e cumprem exatamente o objetivo especificado.
- Cada ADR deve registrar exatamente uma decisão.
- Não agrupe decisões independentes no mesmo ADR.