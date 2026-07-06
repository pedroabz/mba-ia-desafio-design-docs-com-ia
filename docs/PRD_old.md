# PRD -- Sistema de Webhooks de Notificação de Pedidos

**Autor:** Consolidado a partir da reunião técnica (Larissa, Marcos, Bruno, Diego, Sofia)
**Status:** Aprovado em reunião
**Última atualização:** 2026-07-04

---

## 1. Resumo e contexto da feature

O Order Management System (OMS) é uma aplicação Node.js/TypeScript que gerencia o ciclo de vida de pedidos para clientes B2B. O ciclo de vida do pedido segue uma máquina de estados controlada (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED), com transições validadas, controle transacional de estoque e auditoria de mudanças de status.

Atualmente, a plataforma não possui nenhum mecanismo de notificação externa. Clientes que precisam saber quando o status de seus pedidos muda dependem exclusivamente de polling periódico na API REST. Esta feature introduz um **sistema de webhooks outbound** que notifica clientes B2B de forma proativa quando o status de seus pedidos muda, eliminando a necessidade de polling.

---

## 2. Problema e motivação

### Problema atual

Clientes B2B integrados ao OMS precisam monitorar mudanças de status dos seus pedidos. A única forma disponível hoje é **polling** no endpoint de listagem de pedidos, consultando repetidamente para detectar alterações. Isso gera:

- **Latência de integração:** o cliente só descobre a mudança na próxima iteração de polling, que pode levar minutos ou mais.
- **Custo operacional elevado:** chamadas repetitivas sobrecarregam tanto a infraestrutura do cliente quanto a da plataforma.
- **Experiência degradada:** integrações lentas e manuais prejudicam a operação dos parceiros.

### Demanda concreta

Três clientes B2B -- **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** -- solicitaram formalmente notificações em tempo real. A Atlas Comercial sinalizou que poderá migrar para um concorrente caso a funcionalidade não seja entregue até o fim do trimestre.

**Fonte:** [09:00] Marcos.

### Por que resolver agora

- Risco de churn de clientes B2B de alto valor (Atlas Comercial mencionou explicitamente migração para concorrente).
- O requisito de "tempo real" dos clientes é modesto: qualquer coisa abaixo de 10 segundos já é aceitável para eles.
- A arquitetura atual do sistema de pedidos já opera dentro de transações SQL atômicas para mudança de status, o que viabiliza a adoção de um mecanismo de notificação confiável sem refatoração estrutural significativa.

---

## 3. Público-alvo e cenários de uso

### Público-alvo

| Persona | Descrição |
|---|---|
| **Cliente B2B (integrador)** | Empresas que consomem a API do OMS e precisam reagir automaticamente a mudanças de status de pedidos (ex.: atualizar ERP, disparar logística, notificar clientes finais). |
| **Operador interno** | Usuário autenticado que configura webhooks em nome de um cliente via API. |
| **Administrador** | Usuário com role ADMIN responsável por monitorar entregas de webhook e reprocessar eventos que falharam permanentemente. |

### Cenários de uso

1. **Configuração de webhook:** Operador cadastra uma URL de destino para o cliente Atlas Comercial, selecionando os status SHIPPED e DELIVERED como eventos de interesse. O sistema fornece uma credencial de autenticação exclusiva para aquele webhook.

2. **Notificação automática:** Quando um pedido do cliente Atlas muda de PROCESSING para SHIPPED, o cliente recebe uma notificação no endpoint cadastrado com os dados da transição em menos de 10 segundos, sem necessidade de consultar a API.

3. **Falha temporária do cliente:** O endpoint do MaxDistribuição fica indisponível por manutenção (2 horas). O sistema reenvia as notificações automaticamente de forma espaçada e as entrega quando o serviço do cliente voltar ao ar, sem perda de eventos.

4. **Falha permanente e reprocessamento:** Quando as tentativas automáticas de reenvio se esgotam sem sucesso, o evento fica disponível para consulta pelo administrador. Após o cliente corrigir o problema, o administrador pode disparar o reenvio manualmente.

5. **Rotação de credencial:** O cliente Nova Cargo identifica que sua credencial foi comprometida. Solicita a troca via API. A transição ocorre sem interrupção do serviço, com tempo suficiente para atualizar os sistemas do lado do cliente.

6. **Validação de autenticidade:** O cliente recebe uma notificação e consegue verificar, usando sua credencial, que o conteúdo não foi adulterado e que a origem é legítima da plataforma.

7. **Consulta de histórico:** O cliente Atlas consulta o histórico de notificações enviadas para verificar os últimos envios, incluindo se foram entregues com sucesso ou falharam.

---

## 4. Objetivos e métricas de sucesso

### Objetivos

1. Permitir que clientes B2B recebam notificações de mudança de status de pedidos em tempo real (< 10 segundos).
2. Garantir confiabilidade na entrega com semântica at-least-once e mecanismo de retry robusto.
3. Assegurar integridade e autenticidade das notificações via assinatura criptográfica (HMAC-SHA256).
4. Manter consistência transacional: se o status do pedido mudou, o evento de notificação foi registrado; se houve rollback, o evento não existe.
5. Reter os 3 clientes B2B que demandaram a funcionalidade (Atlas Comercial, MaxDistribuição, Nova Cargo).

### Métricas de sucesso

| Métrica | Meta | Justificativa |
|---|---|---|
| Latência de entrega (p95) | < 10 segundos entre mudança de status e recebimento pelo cliente | Requisito explícito dos clientes ([09:02] Marcos) |
| Taxa de entrega bem-sucedida (1a tentativa) | > 95% | Baseline razoável para endpoints saudáveis |
| Taxa de entrega bem-sucedida (com retries) | > 99% | O mecanismo de retry deve cobrir indisponibilidades temporárias |
| Redução de chamadas de polling | > 80% dos clientes integrados deixam de fazer polling | Objetivo principal da feature |
| Inconsistência transacional | 0 eventos perdidos | Garantia arquitetural do padrão outbox |
| Adoção na primeira fase | 3 clientes B2B integrados | Demanda inicial que motivou a feature |

**Nota:** As metas de taxa de entrega (95% e 99%) não foram discutidas na reunião e representam estimativas razoáveis de produto. Devem ser validadas com dados reais após o lançamento.

---

## 5. Escopo

### Incluso (Fase 1)

- Registro automático de evento de notificação dentro da transação de mudança de status do pedido (padrão Transactional Outbox).
- Worker separado para despacho das notificações HTTP.
- Retry com backoff exponencial (5 tentativas: 1m, 5m, 30m, 2h, 12h) e DLQ para eventos que falharam permanentemente.
- Assinatura HMAC-SHA256 por endpoint com secret única gerada pelo sistema.
- Rotação de secret com grace period de 24 horas.
- CRUD de configuração de webhook (criar, listar, editar, remover).
- Filtro de eventos por status: o cliente escolhe quais transições de status deseja receber.
- Consulta de histórico de entregas de um webhook.
- Endpoint administrativo para replay de eventos da DLQ (restrito a role ADMIN).
- Validação de URL HTTPS obrigatória no cadastro.
- Payload renderizado como snapshot no momento da transição (não lazy).
- Timeout de 10 segundos para chamadas HTTP ao endpoint do cliente.
- Limite de 64KB no payload (erro caso exceda).

### Fora de escopo (Fase 1)

1. **Notificação por email quando webhook falha repetidamente.** Mencionado por Marcos ([09:37]) e explicitamente adiado por Larissa para próxima fase, após medir impacto.

2. **Rate limiting de saída** (controle de volume de chamadas enviadas ao cliente). Levantado por Diego ([09:38]) e deliberadamente deixado como ponto de observação: será implementado se/quando virar problema.

3. **Dashboard visual** para clientes acompanharem webhooks. Mencionado por Marcos ([09:39]) e descartado como projeto separado do time de frontend.

4. **Garantia de exactly-once delivery.** Semântica é at-least-once; deduplicação é responsabilidade do cliente via identificador único do evento.

5. **Ordering global** de eventos entre pedidos diferentes. Garantia de ordenação existe apenas por pedido individual com worker único.

6. **Múltiplos workers** em paralelo (escalabilidade horizontal). Decisão deliberada de manter single-worker nesta fase.

7. **Archival/cleanup** de eventos entregues. Mencionado por Diego ([09:08]) como "30 dias", mas explicitamente fora do escopo desta feature.

---

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RF01 | O sistema deve registrar um evento de notificação de forma atômica, dentro da mesma transação que altera o status do pedido. Se a transação falhar, o evento não deve existir. | [09:06] Diego, [09:40] Bruno |
| RF02 | O payload do evento deve ser um snapshot dos dados do pedido no momento da transição de status. Não deve ser renderizado sob demanda no momento do envio. | [09:51-09:52] Larissa, Diego, Bruno |
| RF03 | O sistema deve operar um worker separado que processa e despacha os eventos pendentes via chamadas HTTP aos endpoints cadastrados. | [09:08-09:11] Diego |
| RF04 | Quando o endpoint do cliente falha (erro HTTP ou timeout), o sistema deve retentar com backoff exponencial: 5 tentativas nos intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. | [09:15-09:17] Diego, Larissa |
| RF05 | Após esgotar todas as tentativas de retry, o evento deve ser movido para uma Dead Letter Queue (DLQ) persistida, contendo payload, motivo da falha e timestamp. | [09:17-09:18] Diego |
| RF06 | Um administrador (role ADMIN) deve poder reprocessar eventos da DLQ, reinserindo-os na fila de envio. A ação deve ser registrada para auditoria. | [09:18-09:19] Diego, [09:36] Sofia |
| RF07 | O sistema deve oferecer CRUD completo de configuração de webhook: criação (com URL e lista de status de interesse), listagem por cliente, edição e remoção. | [09:31-09:33] Marcos, Bruno |
| RF08 | Na criação de um webhook, a secret deve ser gerada pelo sistema e devolvida na resposta. O cliente não define sua própria secret. | [09:31] Marcos |
| RF09 | O cliente deve poder solicitar rotação de secret. A secret anterior permanece válida por 24 horas em paralelo com a nova (grace period). | [09:21] Sofia |
| RF10 | O sistema deve permitir que o cliente escolha quais transições de status deseja receber. Eventos para status não selecionados não devem ser gerados. | [09:33-09:34] Marcos, Bruno |
| RF11 | O sistema deve oferecer endpoint para consulta de histórico de entregas de um webhook, incluindo status (sucesso/falha), payload, resposta e tempo de resposta. | [09:34] Marcos |
| RF12 | Cada evento deve possuir um identificador único (UUID) que permita ao cliente deduplicar entregas repetidas (semântica at-least-once). | [09:24-09:25] Diego |
| RF13 | O payload do evento deve ser assinado com HMAC-SHA256 usando a secret do endpoint. A assinatura deve ser enviada em header dedicado para que o cliente valide autenticidade. | [09:19-09:20] Sofia |
| RF14 | URLs de webhook devem obrigatoriamente usar HTTPS. Cadastro com HTTP deve ser rejeitado. | [09:23] Sofia |
| RF15 | O endpoint de replay da DLQ deve exigir role ADMIN e registrar em log quem executou a ação. | [09:35-09:36] Larissa, Sofia |

---

## 7. Requisitos não funcionais

| ID | Requisito | Especificação | Fonte |
|---|---|---|---|
| RNF01 | Latência de entrega | < 10 segundos entre commit da transação de status e recebimento pelo cliente. | [09:02] Marcos, [09:10] Larissa |
| RNF02 | Consistência transacional | Evento de notificação existe se e somente se a transação de mudança de status commitou com sucesso. Zero inconsistência. | [09:06] Diego |
| RNF03 | Isolamento de processos | Worker roda como processo separado da API. Restart da API não afeta o worker e vice-versa. | [09:11] Diego |
| RNF04 | Segurança -- Autenticidade | Assinatura HMAC-SHA256 sobre o corpo da requisição, com secret única por endpoint. | [09:20] Sofia |
| RNF05 | Segurança -- Transporte | TLS obrigatório. URLs HTTP rejeitadas na validação. | [09:23] Sofia |
| RNF06 | Segurança -- Rotação | Secrets rotacionáveis com grace period de 24h para migração gradual. | [09:21] Sofia |
| RNF07 | Segurança -- Replay protection | Envio de timestamp no header para que o cliente possa detectar replay attacks. | [09:44] Diego |
| RNF08 | Timeout | 10 segundos para resposta do endpoint do cliente. Falha é registrada e entra na política de retry. | [09:42] Diego |
| RNF09 | Limite de payload | 64KB. Erro (não truncamento) caso exceda. | [09:24] Diego, Larissa |
| RNF10 | Semântica de entrega | At-least-once. Duplicatas possíveis; deduplicação via identificador único é responsabilidade do cliente. | [09:24-09:25] Diego |
| RNF11 | Ordenação | Garantida por pedido individual com single-worker (processamento em ordem cronológica). Sem garantia de ordering global entre pedidos distintos. | [09:12-09:13] Diego, Larissa |
| RNF12 | Observabilidade | Logs de despacho, falha e retry. Reutilização do logger já existente no projeto (Pino). | [09:29] Bruno |
| RNF13 | Aderência a padrões existentes | O módulo deve seguir os padrões arquiteturais já estabelecidos no projeto: estrutura modular, hierarquia de erros, validação de schemas, middleware de autorização. | [09:27-09:30] Bruno, Larissa |

---

## 8. Decisões e trade-offs principais

Esta seção lista as decisões de nível de produto tomadas na reunião. O detalhamento técnico de cada decisão (alternativas, consequências, justificativa completa) será registrado nos ADRs correspondentes.

### D1 -- Mecanismo de notificação assíncrono com outbox transacional

O despacho de webhooks será assíncrono, não síncrono dentro da transação de mudança de status. A motivação é que a transação de status já é pesada e uma chamada HTTP síncrona travaria o fluxo caso o cliente estivesse lento ou offline. O registro do evento será atômico com a transação de status usando o padrão Transactional Outbox no MySQL existente, sem necessidade de infraestrutura adicional (Redis, RabbitMQ, etc.).

**Trade-off aceito:** o time abriu mão de uma solução baseada em fila externa (mais escalável) em favor de simplicidade operacional, dado que o time é pequeno e o volume atual não justifica infraestrutura adicional.

**Fonte:** [09:03-09:07] Bruno, Diego, Larissa.

### D2 -- Worker com polling de 2 segundos

O worker lê eventos pendentes via polling a cada 2 segundos, em vez de usar mecanismo reativo. MySQL não possui NOTIFY/LISTEN nativo. A latência mínima de 2 segundos é aceita dado o requisito de < 10 segundos.

**Trade-off aceito:** latência mínima de 2 segundos no melhor caso vs. complexidade de trigger/notificação.

**Fonte:** [09:09-09:10] Diego, Marcos, Larissa.

### D3 -- Semântica at-least-once (não exactly-once)

O sistema garante que o evento será entregue pelo menos uma vez, mas pode haver duplicatas. O cliente é responsável por deduplicar usando o identificador único do evento. Exactly-once exigiria coordenação bidirecional de complexidade desproporcional. É o padrão adotado por plataformas de referência (Stripe, GitHub).

**Trade-off aceito:** responsabilidade de deduplicação transferida para o cliente, em troca de simplicidade arquitetural significativa.

**Fonte:** [09:24-09:26] Diego, Sofia, Marcos.

### D4 -- 5 tentativas de retry com backoff exponencial, depois DLQ

Retry infinito deixaria eventos pendurados indefinidamente. 5 tentativas cobrem ~15 horas, suficiente para manutenções planejadas (histórico de indisponibilidade de até 2 horas em clientes). Eventos que falharam permanentemente vão para DLQ com replay manual administrativo.

**Trade-off aceito:** 3 tentativas seriam mais agressivas, mas insuficientes para cobrir janelas reais de indisponibilidade.

**Fonte:** [09:15-09:18] Diego, Bruno, Larissa.

### D5 -- Secret por endpoint com rotação e grace period

Cada endpoint de webhook tem sua própria secret (não uma secret global da plataforma). Rotação com 24 horas de validade dupla para migração gradual. Motivação: se uma secret vaza, o impacto é isolado a um único endpoint.

**Fonte:** [09:21-09:22] Sofia, Diego.

### D6 -- Payload como snapshot, não renderização sob demanda

O payload do evento é gerado e armazenado no momento da transição de status, não no momento do envio. Isso garante que o evento reflete o estado exato do pedido quando a transição ocorreu, mesmo que o pedido mude novamente depois.

**Fonte:** [09:51-09:52] Larissa, Diego, Bruno.

---

## 9. Dependências

### Dependências internas

| Componente | Relação com a feature |
|---|---|
| Transação de mudança de status de pedido | Ponto de inserção do evento na outbox. A feature depende de estender a transação existente sem quebrá-la. |
| Sistema de autenticação JWT | Protege os endpoints CRUD de webhook e o endpoint admin de replay. |
| Controle de autorização por role | O endpoint de replay da DLQ exige role ADMIN; demais endpoints autenticados aceitam qualquer role. |
| Hierarquia de erros existente | Novos erros do módulo de webhooks devem seguir o padrão existente (classes de erro, códigos de erro). |
| Middleware de erro centralizado | Já trata erros da hierarquia existente, validação Zod e erros Prisma. Deve funcionar com erros do novo módulo sem alteração. |
| Logger existente (Pino) | Reutilizado para logs do worker e do módulo. |
| Estrutura modular existente | O novo módulo de webhooks segue o padrão de organização de módulos já estabelecido. |

### Dependências externas

- **Nenhuma nova dependência de infraestrutura.** O sistema usa o mesmo banco MySQL e a mesma stack. O worker roda como processo separado no mesmo ambiente.
- **Nenhuma nova dependência de pacote NPM obrigatória.** As funcionalidades criptográficas (HMAC) e de requisição HTTP (fetch) estão disponíveis nativamente no Node.js 20+.

### Dependência de prazo

- Revisão de segurança dedicada (Sofia) deve ocorrer antes do deploy. Reservados 2 dias úteis no cronograma. **Fonte:** [09:46] Sofia.

---

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| R1 | **Acúmulo de eventos não processados na outbox** degrada performance do worker e do banco. | Média | Médio | Índices otimizados na tabela de outbox. Archival de eventos antigos planejado para fase futura. Monitoramento de tamanho da tabela. |
| R2 | **Worker cai e eventos acumulam sem despacho.** | Baixa | Alto | Worker retoma processamento do ponto onde parou ao reiniciar (eventos permanecem no banco). Monitoramento de processo. Nenhum evento é perdido. |
| R3 | **Clientes com endpoints lentos geram backlog no worker.** | Média | Médio | Timeout de 10 segundos por chamada. Worker processa em batch e segue para o próximo em caso de falha. Limitação conhecida do single-worker: backlog pode crescer se muitos clientes estiverem lentos simultaneamente. |
| R4 | **Vazamento de secret do cliente.** | Baixa | Alto | Secret única por endpoint (isolamento de impacto). Rotação com grace period de 24h. Comunicação exclusivamente via HTTPS. |
| R5 | **Duplicação de eventos causa efeitos colaterais no sistema do cliente.** | Média | Baixo | Semântica at-least-once documentada explicitamente. Identificador único do evento enviado para deduplicação. Documentação no portal do desenvolvedor. |
| R6 | **Perda de ordenação ao escalar para múltiplos workers no futuro.** | Baixa (futuro) | Médio | Limitação conhecida e documentada. Solução futura: particionamento por pedido ou lock pessimista. Não é problema com single-worker. |
| R7 | **Prazo apertado (fim do trimestre, Atlas Comercial).** | Média | Alto | Estimativa de 3 sprints aprovada pela tech lead. Revisão de segurança incluída no cronograma. Escopo bem delimitado com itens explicitamente fora de escopo. |

---

## 11. Critérios de aceitação

### Consistência transacional

- [ ] Quando a transação de mudança de status commita com sucesso e existe webhook configurado para aquele status, existe exatamente um evento correspondente registrado para despacho.
- [ ] Quando a transação de mudança de status falha (rollback), nenhum evento de notificação é registrado.
- [ ] O evento registrado contém um snapshot do estado do pedido no momento da transição.

### Entrega de notificações

- [ ] O worker opera como processo independente da API.
- [ ] Eventos pendentes são processados e despachados em menos de 10 segundos (p95) após o commit da transação.
- [ ] A notificação é enviada via POST HTTPS ao endpoint cadastrado pelo cliente.
- [ ] A notificação inclui identificador único do evento, assinatura HMAC, timestamp e identificador do webhook.

### Retry e DLQ

- [ ] Quando o endpoint do cliente retorna erro ou excede o timeout de 10 segundos, o evento é agendado para retry.
- [ ] O backoff segue a progressão: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas.
- [ ] Após 5 tentativas falhadas, o evento é movido para a DLQ com motivo da falha registrado.
- [ ] Administrador (role ADMIN) consegue reprocessar eventos da DLQ. A ação é registrada para auditoria.

### Segurança

- [ ] Payload é assinado com HMAC-SHA256 usando a secret única do endpoint.
- [ ] Secret é gerada pelo sistema na criação do webhook e devolvida na resposta.
- [ ] Rotação de secret mantém a secret anterior válida por 24 horas.
- [ ] URLs HTTP são rejeitadas na validação de cadastro.
- [ ] Endpoint de replay da DLQ exige role ADMIN.

### Configuração e consulta

- [ ] CRUD completo de configuração de webhook funciona corretamente (criar, listar, editar, remover).
- [ ] Filtro de eventos por status funciona: se o webhook não escuta o status de destino, nenhum evento é gerado.
- [ ] Endpoint de histórico de entregas retorna registros com status, payload, resposta e tempo de resposta.

### Aderência a padrões

- [ ] Módulo segue a estrutura modular existente do projeto.
- [ ] Erros utilizam a hierarquia de erros existente com códigos prefixados WEBHOOK_.
- [ ] Logs utilizam o logger existente (Pino).
- [ ] Validações utilizam Zod.

---

## 12. Estratégia de testes e validação

### Testes unitários

- Verificar que o registro de evento na outbox ocorre dentro da transação de mudança de status e que o payload gerado contém os campos corretos.
- Testar que eventos só são registrados quando existe webhook configurado para aquele status.
- Testar geração e verificação de assinatura HMAC-SHA256 com diferentes payloads e secrets.
- Testar que ambas as secrets (antiga e nova) são válidas durante o grace period de 24h.
- Testar cálculo dos intervalos de backoff exponencial.
- Testar rejeição de URLs HTTP e aceitação de HTTPS.

### Testes de integração

- Criar pedido, mudar status e verificar que evento foi registrado. Forçar falha na transação e verificar que evento não existe.
- Testar CRUD completo de webhook com persistência no banco.
- Inserir evento pendente, executar ciclo do worker e verificar que endpoint mock recebeu a notificação com headers e payload corretos.
- Configurar endpoint mock que falha nas primeiras chamadas e depois retorna sucesso. Verificar que o retry funciona conforme o backoff.
- Configurar endpoint mock que sempre falha. Verificar que após 5 tentativas o evento aparece na DLQ com motivo.
- Inserir evento na DLQ, chamar endpoint de replay com role ADMIN, verificar reinserção na fila. Verificar que role OPERATOR recebe 403.

### Revisão de segurança

- Sofia (Engenheira de Segurança) faz revisão dedicada do código de HMAC e geração de secret antes do deploy. Reservar 2 dias úteis. **Fonte:** [09:46] Sofia.
- Testar que payload adulterado falha na verificação de assinatura.
- Testar que cadastro com URL HTTP é rejeitado.
- Testar que endpoint de replay exige role ADMIN.

### Validação com clientes

- Disponibilizar ambiente de staging para os 3 clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) testarem a integração antes do deploy em produção.
- Marcos (PM) prepara documentação no portal do desenvolvedor explicando formato do payload, headers, verificação de HMAC, semântica at-least-once e deduplicação via identificador único do evento. **Fonte:** [09:26] Marcos.

### Critérios de go/no-go para produção

1. Todos os testes unitários e de integração passam.
2. Revisão de segurança da Sofia aprovada.
3. Pelo menos 1 cliente B2B validou a integração em staging.
4. Worker executou por pelo menos 48 horas em staging sem erros não tratados.

**Nota:** O critério de 48 horas de execução em staging não foi discutido na reunião. É uma recomendação de produto para garantir estabilidade antes do lançamento. Deve ser validado com a tech lead.
