---
description: Gerar um documento TRACKER (Tracker de Rastreabilidade) a partir da análise do repositório, da transcrição e das documentações geradas (PRD, RFC, FDD e ADRs)
tags: [project, tracker]
---

Utilize o agente `design-docs-generator` para gerar o documento TRACKER.

## INPUT

- Arquivo de Transcrição
- Documentos já gerados (qualquer combinação de PRD, RFC, FDD, ADRs disponível no momento)
- `docs/TRACKER.md` existente (se houver)

## ESCOPO

Considere o PRD como a fonte de verdade para requisitos de negócio.

Considere o RFC como a fonte de verdade para a proposta arquitetural.

Considere o FDD como a fonte de verdade para os detalhes de implementação.

Considere as ADRs como a fonte de verdade para cada decisão arquitetural individual.

## Modo de operação

- **Se `docs/TRACKER.md` não existir:** crie-o do zero a partir dos documentos disponíveis.
- **Se `docs/TRACKER.md` já existir:** leia o conteúdo atual e **adicione** as linhas referentes aos novos documentos gerados desde a última atualização. Preserve as linhas existentes e seus IDs — não reordene, não renumere, não remova linhas válidas. Apenas acrescente os novos itens ao final da tabela.

Gere ou atualize o arquivo `TRACKER.md`, contendo uma tabela Markdown que mapeia cada item registrado nos documentos à sua origem na transcrição ou no código.

O Tracker funciona como uma referência cruzada: permite que qualquer leitor entenda de onde veio cada requisito, decisão, restrição ou trade-off, garantindo que toda a documentação permaneça consistente com o que foi efetivamente discutido na reunião e com o estado atual do repositório.

O Tracker é um documento específico deste projeto. Sua principal função é garantir rastreabilidade e evitar alucinações da IA.

## OUTPUT

Arquivo `docs/TRACKER.md`.

O arquivo deve conter uma tabela Markdown no seguinte formato:

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|----|-----------|------|-------------------|--------|-------------|

Onde:

- **ID:** identificador único do item (ex.: `PRD-FR-01`, `RFC-ALT-02`, `FDD-CONTRATO-03`, `ADR-002`)
- **Documento:** arquivo onde o item aparece (`docs/PRD.md`, `docs/RFC.md`, `docs/FDD.md`, `docs/adrs/ADR-002-...md`)
- **Tipo:** Requisito Funcional, Requisito Não Funcional, Decisão, Restrição, Trade-off, entre outros
- **Conteúdo (resumo):** descrição resumida do item
- **Fonte:** `TRANSCRICAO` ou `CODIGO`
- **Localização:**
  - Para `TRANSCRICAO`: timestamp + nome do participante (ex.: `[09:17] Diego`)
  - Para `CODIGO`: caminho do arquivo (ex.: `src/modules/orders/order.service.ts`)

## Regras

- Cobertura mínima: pelo menos 80% dos itens identificáveis nos documentos disponíveis devem possuir uma linha correspondente no Tracker.
- Se o mínimo de 80% não for atingido, modificar este arquivo até que isso seja atingido.
- Todo item registrado deve possuir uma origem claramente identificável na transcrição ou no código.
- Não invente fontes para satisfazer a cobertura mínima. Caso algum item não possua origem identificável, registre explicitamente essa inconsistência.
- Sempre valide se referências ao código correspondem ao estado atual do repositório.
- Após finalizar o documento, revise-o e garanta que a estrutura da tabela está correta e que todas as referências apontam para localizações válidas.
