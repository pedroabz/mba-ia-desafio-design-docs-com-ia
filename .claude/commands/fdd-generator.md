---
description: Gerar um documento FDD (Feature Design Document) a partir da análise do repositório e de um arquivo de transcrição
tags: [project, fdd]
---

**INPUT**

- Arquivo de Transcrição
- Arquivo RFC
- ADRs em `docs/adrs/`

**Escopo**

Utilize o agente `design-docs-generator` para gerar um FDD (Feature Design Document).

Considere o RFC como a fonte de verdade para a justificativa arquitetural.

Considere os ADRs como fonte de verdade para as decisões individuais já registradas.

Analise o repositório, o RFC, os ADRs e o arquivo de transcrição para gerar um FDD contendo apenas aquilo que é relevante sobre as decisões tomadas na reunião.

O documento deve se ater ao escopo de um FDD: detalhar **como implementar** a feature.

O FDD é o documento mais técnico do conjunto e deve estar acionável o suficiente para que um desenvolvedor consiga iniciar a implementação da feature apenas com este documento.

**OUTPUT**

Arquivo `.md` na pasta `docs`.

### Seções obrigatórias

- Contexto e motivação técnica
- Objetivos técnicos
- Escopo e exclusões
- Fluxos detalhados (criação do evento na outbox, processamento pelo worker, retry, DLQ)
- Contratos públicos (endpoints HTTP com payloads de exemplo, headers, status codes, semântica)
- Matriz de erros previstos com códigos no padrão `WEBHOOK_*`
- Estratégias de resiliência (timeouts, retries, backoff, fallback)
- Observabilidade (métricas, logs, tracing)
- Dependências e compatibilidade
- Critérios de aceite técnicos
- Riscos e mitigação
- Integração com o sistema existente (nomear pelo menos 4 caminhos de arquivo reais do código base e descrever como o módulo de webhooks vai se integrar com cada um)

### Regras

- Após finalizar o documento, revise-o e garanta que todas as seções obrigatórias existem e cumprem exatamente o objetivo especificado.
- Inclua o detalhamento de implementação necessário para a construção da feature.
- O FDD complementa o RFC. Não repita justificativas arquiteturais ou alternativas já registradas no RFC; concentre-se em especificar a implementação.
- Sempre que possível, reutilize e referencie componentes, padrões e convenções já existentes no repositório, em vez de propor novas estruturas.
- Caso alguma informação exigida pela estrutura do documento não esteja presente na transcrição nem possa ser inferida a partir do repositório, não invente conteúdo. Registre explicitamente que aquela informação não foi identificada durante a análise.