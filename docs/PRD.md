# PRD -- Sistema de Webhooks de Notificação de Pedidos

| Campo                | Valor                                                                 |
|----------------------|-----------------------------------------------------------------------|
| **Autor**            | Consolidado a partir da reunião técnica                               |
| **Participantes**    | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno), Diego (Eng. Senior), Sofia (Eng. Segurança) |
| **Status**           | Aprovado em reunião                                                   |
| **Data**             | 2026-07-05                                                            |

---

## 1. Resumo e contexto da feature

O Order Management System (OMS) é uma aplicação Node.js/TypeScript que gerencia o ciclo de vida de pedidos para clientes B2B. O sistema opera com uma máquina de estados controlada (PENDING, PAID, PROCESSING, SHIPPED, DELIVERED, CANCELLED), transições validadas, controle transacional de estoque e auditoria de mudanças de status via histórico.

Atualmente, a plataforma não possui nenhum mecanismo de notificação externa. Clientes que precisam saber quando o status de seus pedidos muda dependem exclusivamente de polling periódico na API REST. Esta feature introduz um **sistema de webhooks outbound** que notifica clientes B2B de forma proativa quando o status de seus pedidos muda, eliminando a necessidade de polling e atendendo a uma demanda formal de clientes estratégicos.

---

## 2. Problema e motivação

### Problema atual

Clientes B2B integrados ao OMS precisam monitorar mudanças de status dos seus pedidos. A única forma disponível hoje é **polling** no endpoint de listagem de pedidos, consultando repetidamente para detectar alterações. Isso gera:

- **Latência de integração:** o cliente só descobre a mudança na próxima iteração de polling, que pode levar minutos ou mais.
- **Custo operacional elevado:** chamadas repetitivas sobrecarregam tanto a infraestrutura do cliente quanto a da plataforma.
- **Experiência degradada:** integrações lentas e manuais prejudicam a operação dos parceiros.

### Demanda formal

Três clientes B2B -- **Atlas Comercial**, **MaxDistribuição** e **Nova Cargo** -- solicitaram formalmente notificações em tempo real na semana anterior à reunião. A Atlas Comercial sinalizou que poderá migrar para um concorrente caso a funcionalidade não seja entregue até o fim do trimestre.

**Fonte:** [09:00] Marcos: "a gente recebeu na semana passada um pedido formal de três clientes B2B [...] A Atlas chegou a sugerir que se a gente não entregar isso até fim do trimestre, eles podem migrar pro nosso concorrente."

### Por que resolver agora

- Risco concreto de churn de cliente B2B de alto valor (Atlas Comercial mencionou explicitamente migração para concorrente).
- O requisito de "tempo real" dos clientes é modesto: qualquer coisa abaixo de 10 segundos já é aceitável ([09:02] Marcos).
- A arquitetura atual do sistema de pedidos já opera com transações SQL atômicas para mudança de status, o que viabiliza a adoção de um mecanismo de notificação confiável sem refatoração estrutural significativa.

---

## 3. Público-alvo e cenários de uso

### Público-alvo

| Persona | Descrição |
|---|---|
| **Cliente B2B (integrador)** | Empresas que consomem a API do OMS e precisam reagir automaticamente a mudanças de status de pedidos (ex.: atualizar ERP, disparar logística, notificar clientes finais). |
| **Operador interno** | Usuário autenticado que configura webhooks em nome de um cliente via API. |
| **Administrador** | Usuário com role ADMIN responsável por monitorar entregas de webhook e reprocessar eventos que falharam permanentemente. |

### Cenários de uso

1. **Configuração de webhook:** Operador cadastra uma URL de destino para o cliente Atlas Comercial, selecionando os status SHIPPED e DELIVERED como eventos de interesse. O sistema gera e fornece uma credencial de autenticação exclusiva para aquele webhook.

2. **Notificação automática:** Quando um pedido do cliente Atlas muda de PROCESSING para SHIPPED, o cliente recebe uma notificação no endpoint cadastrado com os dados da transição em menos de 10 segundos, sem necessidade de consultar a API.

3. **Falha temporária do cliente:** O endpoint do MaxDistribuição fica indisponível por manutenção (2 horas). O sistema reenvia as notificações automaticamente de forma espaçada e as entrega quando o serviço do cliente voltar ao ar, sem perda de eventos.

4. **Falha permanente e reprocessamento:** Quando as tentativas automáticas de reenvio se esgotam sem sucesso, o evento fica disponível para consulta pelo administrador. Após o cliente corrigir o problema, o administrador pode disparar o reenvio manualmente.

5. **Rotação de credencial:** O cliente Nova Cargo identifica que sua credencial foi comprometida. Solicita a troca via API. A transição ocorre sem interrupção do serviço, com período de graça suficiente para atualizar os sistemas do lado do cliente.

6. **Validação de autenticidade:** O cliente recebe uma notificação e consegue verificar, usando sua credencial, que o conteúdo não foi adulterado e que a origem é legítima da plataforma.

7. **Consulta de histórico:** O cliente Atlas consulta o histórico de notificações enviadas para verificar os últimos envios, incluindo se foram entregues com sucesso ou falharam, e o tempo de resposta de cada entrega.

---

## 4. Objetivos e métricas de sucesso

### Objetivos

1. Permitir que clientes B2B recebam notificações de mudança de status de pedidos em tempo real (< 10 segundos).
2. Garantir confiabilidade na entrega com semântica at-least-once e mecanismo de retry robusto.
3. Assegurar integridade e autenticidade das notificações via assinatura criptográfica.
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

**Nota:** As metas de taxa de entrega (95% na 1a tentativa e 99% com retries) e a meta de redução de polling não foram discutidas na reunião e representam estimativas razoáveis de produto. Devem ser validadas com dados reais após o lançamento.

---

## 5. Escopo

### Incluído (Fase 1)

- Registro automático de evento de notificação, atômico com a transação de mudança de status do pedido.
- Worker separado para despacho das notificações HTTP.
- Retry automático com backoff exponencial (5 tentativas: 1m, 5m, 30m, 2h, 12h) e Dead Letter Queue para eventos que falharam permanentemente.
- Assinatura criptográfica do payload por endpoint com credencial única gerada pelo sistema.
- Rotação de credencial com período de graça de 24 horas.
- CRUD de configuração de webhook (criar, listar, editar, remover).
- Filtro de eventos por status: o cliente escolhe quais transições de status deseja receber.
- Consulta de histórico de entregas de um webhook.
- Endpoint administrativo para replay de eventos da DLQ (restrito a role ADMIN com auditoria).
- Validação de URL HTTPS obrigatória no cadastro.
- Payload renderizado como snapshot no momento da transição (não sob demanda no envio).
- Timeout de 10 segundos para chamadas HTTP ao endpoint do cliente.
- Limite de 64KB no payload (erro caso exceda).

### Fora de escopo (Fase 1)

| Item | Decisão | Fonte |
|---|---|---|
| **Notificação por email quando webhook falha repetidamente** | Explicitamente adiado para próxima fase. | [09:37] Larissa: "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto." |
| **Dashboard visual** para clientes acompanharem webhooks | Descartado como projeto separado do time de frontend. | [09:39-40] Marcos/Larissa: "Não, agora não. Só endpoints. Painel é projeto separado do time de frontend." |
| **Rate limiting de saída** (controle de volume de chamadas enviadas ao cliente) | Adiado para observação em produção. | [09:38-39] Diego/Larissa: "Eu acho que não. A gente observa e implementa se virar problema." |
| **Garantia de exactly-once delivery** | Semântica é at-least-once; deduplicação é responsabilidade do cliente via identificador único do evento. | [09:24-09:25] Diego |
| **Ordering global** de eventos entre pedidos diferentes | Garantia de ordenação existe apenas por pedido individual com worker único. | [09:12-09:13] Diego, Larissa |
| **Múltiplos workers** em paralelo (escalabilidade horizontal) | Decisão deliberada de manter worker único nesta fase. | [09:13] Diego: "isso é problema do futuro, não agora." |
| **Archival/cleanup** de eventos entregues | Mencionado como "30 dias", mas explicitamente fora do escopo desta feature. | [09:08] Diego: "fora do escopo dessa feature." |

---

## 6. Requisitos funcionais

| ID | Requisito | Fonte |
|---|---|---|
| RF-01 | O sistema deve permitir o cadastro de um endpoint de webhook para um cliente, com URL de destino e lista de status de interesse. A credencial de autenticação é gerada pelo sistema e devolvida na criação. | [09:31] Marcos |
| RF-02 | O sistema deve permitir a edição de um webhook existente (URL, status de interesse, estado ativo/inativo). | [09:33] Bruno |
| RF-03 | O sistema deve permitir a remoção de um webhook. | [09:33] Bruno |
| RF-04 | O sistema deve permitir a listagem dos webhooks configurados para um determinado cliente. | [09:33] Bruno |
| RF-05 | O sistema deve registrar automaticamente um evento de notificação quando o status de um pedido muda, de forma atômica com a transação de mudança de status. Se a transação falhar, o evento não deve existir. | [09:06] Diego, [09:40] Bruno |
| RF-06 | O sistema deve filtrar eventos de acordo com os status de interesse configurados em cada webhook. Eventos para status não selecionados não devem ser gerados. | [09:33-09:34] Marcos, Bruno |
| RF-07 | O sistema deve oferecer consulta de histórico de entregas de um webhook, incluindo status de sucesso/falha, tempo de resposta e informações de erro. | [09:34] Marcos |
| RF-08 | O sistema deve retentar automaticamente o envio de notificações que falharam, com intervalos crescentes (backoff exponencial): 5 tentativas nos intervalos de 1 minuto, 5 minutos, 30 minutos, 2 horas e 12 horas. | [09:15-09:17] Diego, Larissa |
| RF-09 | Após esgotar todas as tentativas de retry, o evento deve ser movido para uma Dead Letter Queue persistida, contendo o payload original e o motivo da falha. | [09:17-09:18] Diego |
| RF-10 | Um administrador (role ADMIN) deve poder reprocessar eventos da DLQ, reinserindo-os na fila de envio. A ação deve ser registrada para auditoria (quem executou). | [09:18-09:19] Diego, [09:36] Sofia |
| RF-11 | O cliente deve poder solicitar rotação de credencial. A credencial anterior permanece válida por 24 horas em paralelo com a nova (grace period) para permitir migração gradual. | [09:21-09:22] Sofia |

---

## 7. Requisitos não funcionais

| ID | Requisito | Especificação | Fonte |
|---|---|---|---|
| RNF-01 | Latência de entrega | < 10 segundos entre commit da transação de status e recebimento pelo cliente (p95). | [09:02] Marcos, [09:10] Larissa |
| RNF-02 | Consistência transacional | Evento de notificação existe se e somente se a transação de mudança de status commitou com sucesso. Zero inconsistência. | [09:06] Diego |
| RNF-03 | Isolamento de processos | Worker roda como processo separado da API. Restart da API não afeta o worker e vice-versa. | [09:11] Diego |
| RNF-04 | Autenticidade do payload | Payload assinado criptograficamente com credencial única por endpoint. Cliente valida a assinatura para confirmar origem e integridade. | [09:20] Sofia |
| RNF-05 | Segurança de transporte | TLS obrigatório. URLs HTTP rejeitadas na validação de cadastro. | [09:23] Sofia |
| RNF-06 | Rotação de credenciais | Credenciais rotacionáveis com grace period de 24 horas para migração gradual. | [09:21] Sofia |
| RNF-07 | Proteção contra replay | Envio de timestamp no header para que o cliente possa detectar replay attacks. | [09:44] Diego |
| RNF-08 | Timeout de entrega | 10 segundos para resposta do endpoint do cliente. Timeout é tratado como falha e entra na política de retry. | [09:42] Diego |
| RNF-09 | Limite de payload | 64KB. Erro (não truncamento) caso exceda. | [09:24] Diego, Larissa |
| RNF-10 | Semântica de entrega | At-least-once. Duplicatas possíveis; deduplicação via identificador único é responsabilidade do cliente. | [09:24-09:25] Diego |
| RNF-11 | Ordenação | Garantida por pedido individual com worker único (processamento em ordem cronológica). Sem garantia de ordering global entre pedidos distintos. Limitação conhecida e aceita. | [09:12-09:13] Diego, Larissa |
| RNF-12 | Observabilidade | Logs estruturados de despacho, falha e retry. Reutilização do logger já existente no projeto. | [09:29] Bruno |
| RNF-13 | Aderência a padrões existentes | O módulo deve seguir os padrões arquiteturais já estabelecidos no projeto: estrutura modular, hierarquia de erros, validação de schemas, middleware de autorização. | [09:27-09:30] Bruno, Larissa |

---

## 8. Decisões e trade-offs principais

Esta seção lista as decisões de nível de produto tomadas na reunião. O detalhamento técnico de cada decisão (alternativas, consequências, justificativa completa) está registrado nos ADRs correspondentes.

### D1 -- Notificação assíncrona com registro atômico

O despacho de webhooks será assíncrono, não síncrono dentro da transação de mudança de status. A motivação é que a transação de status já é pesada e uma chamada HTTP síncrona travaria o fluxo caso o cliente estivesse lento ou offline. O registro do evento será atômico com a transação de status, utilizando o banco de dados existente, sem necessidade de infraestrutura adicional.

**Trade-off aceito:** o time abriu mão de uma solução baseada em fila de mensageria externa (mais escalável a longo prazo) em favor de simplicidade operacional, dado que o time é pequeno e o volume atual não justifica infraestrutura adicional.

**Fonte:** [09:03-09:07] Bruno, Diego, Larissa. **ADR:** [ADR-001](adrs/ADR-001-outbox-no-mysql.md).

### D2 -- Detecção de eventos por verificação periódica

O worker detecta novos eventos por verificação periódica a cada 2 segundos, em vez de mecanismo reativo de notificação. A latência mínima de 2 segundos é aceita dado o requisito de < 10 segundos.

**Trade-off aceito:** latência mínima de 2 segundos no pior caso vs. complexidade de mecanismo reativo (que o banco de dados utilizado não suporta nativamente).

**Fonte:** [09:09-09:10] Diego, Marcos, Larissa. **ADR:** [ADR-005](adrs/ADR-005-worker-polling-separado.md).

### D3 -- Entrega pelo menos uma vez (não exatamente uma vez)

O sistema garante que o evento será entregue pelo menos uma vez, mas pode haver duplicatas. O cliente é responsável por deduplicar usando o identificador único do evento. Garantia de entrega exatamente uma vez exigiria coordenação bidirecional de complexidade desproporcional. É o padrão adotado por plataformas de referência (Stripe, GitHub).

**Trade-off aceito:** responsabilidade de deduplicação transferida para o cliente, em troca de simplicidade arquitetural significativa.

**Fonte:** [09:24-09:26] Diego, Sofia, Marcos. **ADR:** [ADR-004](adrs/ADR-004-garantia-at-least-once.md).

### D4 -- 5 tentativas com intervalos crescentes, depois DLQ

Reenvio indefinido deixaria eventos pendurados caso o cliente sumisse. 5 tentativas cobrem aproximadamente 15 horas, suficiente para cobrir manutenções planejadas (histórico de indisponibilidade de até 2 horas em clientes). Eventos que falharam permanentemente vão para fila de eventos mortos com replay manual administrativo.

**Trade-off aceito:** 3 tentativas (proposta por Bruno) seriam mais agressivas, mas insuficientes para cobrir janelas reais de indisponibilidade observadas pela equipe.

**Fonte:** [09:15-09:18] Diego, Bruno, Larissa. **ADR:** [ADR-002](adrs/ADR-002-retry-com-backoff-e-dlq.md).

### D5 -- Credencial única por endpoint com rotação e grace period

Cada endpoint de webhook tem sua própria credencial (não uma credencial global da plataforma). Rotação com 24 horas de validade dupla para migração gradual. Motivação: se uma credencial vaza, o impacto é isolado a um único endpoint. A equipe já teve caso de cliente que vazou credencial em log de aplicação.

**Fonte:** [09:21-09:22] Sofia, Diego. **ADR:** [ADR-003](adrs/ADR-003-autenticacao-hmac-sha256.md).

### D6 -- Payload como snapshot no momento da transição

O conteúdo do evento é gerado e armazenado no momento da transição de status, não no momento do envio. Isso garante que o evento reflete o estado exato do pedido quando a transição ocorreu, mesmo que o pedido mude novamente depois.

**Fonte:** [09:51-09:52] Larissa, Diego, Bruno.

---

## 9. Dependências

### Dependências internas

| Componente | Relação com a feature |
|---|---|
| Transação de mudança de status de pedido | Ponto de inserção do evento de notificação. A feature depende de estender a transação existente sem quebrá-la. |
| Sistema de autenticação JWT | Protege os endpoints de configuração de webhook e o endpoint admin de replay. |
| Controle de autorização por role | O endpoint de replay da DLQ exige role ADMIN; demais endpoints autenticados aceitam qualquer role ([09:37] Sofia). |
| Hierarquia de erros existente | Novos erros do módulo de webhooks devem seguir o padrão existente. |
| Middleware de erro centralizado | Já trata erros da hierarquia existente. Deve funcionar com erros do novo módulo sem alteração. |
| Logger existente | Reutilizado para logs do worker e do módulo. |
| Estrutura modular existente | O novo módulo de webhooks segue o padrão de organização de módulos já estabelecido. |

### Dependências externas

- **Nenhuma nova dependência de infraestrutura.** O sistema usa o mesmo banco de dados e a mesma stack. O worker roda como processo separado no mesmo ambiente.
- **Nenhuma nova dependência de pacote.** As funcionalidades criptográficas e de requisição HTTP estão disponíveis nativamente no runtime utilizado.

### Prazo e cronograma

- **Estimativa:** 3 sprints, incluindo revisão de segurança. **Fonte:** [09:46] Larissa.
- **Compromisso externo:** Atlas Comercial espera entrega até fim do trimestre. Marcos confirmará prazo com os clientes. **Fonte:** [09:45-09:47] Marcos.
- **Revisão de segurança:** Sofia reserva pelo menos 2 dias úteis para revisar o código de autenticação e geração de credenciais antes do deploy. **Fonte:** [09:46] Sofia.

---

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| R1 | **Prazo apertado (compromisso com Atlas Comercial até fim do trimestre).** | Média | Alto | Estimativa de 3 sprints aprovada pela tech lead. Escopo bem delimitado com 7 itens explicitamente fora de escopo. Revisão de segurança incluída no cronograma. |
| R2 | **Worker fora do ar causa acúmulo de eventos sem despacho.** | Média | Alto | Eventos permanecem persistidos e são processados quando o worker reinicia. Nenhum evento é perdido. Monitoramento do processo é necessário para detectar indisponibilidade. |
| R3 | **Acúmulo de eventos não processados degrada performance do banco.** | Média | Médio | Índices otimizados na tabela de eventos. Archival de eventos antigos planejado para fase futura. |
| R4 | **Clientes com endpoints lentos geram backlog no worker.** | Média | Médio | Timeout de 10 segundos por chamada. Worker processa em lote e segue para o próximo em caso de falha. Limitação conhecida do worker único. |
| R5 | **Vazamento de credencial do cliente.** | Baixa | Alto | Credencial única por endpoint (isolamento de impacto). Rotação com grace period de 24h. Comunicação exclusivamente via HTTPS. Credencial não retornada em listagens. |
| R6 | **Duplicação de eventos causa efeitos colaterais no sistema do cliente.** | Média | Baixo | Semântica at-least-once documentada explicitamente. Identificador único do evento enviado para deduplicação. Documentação no portal do desenvolvedor ([09:26] Marcos). |
| R7 | **Perda de ordenação ao escalar para múltiplos workers no futuro.** | Baixa (futuro) | Médio | Limitação conhecida e documentada. Solução futura: particionamento por pedido. Não é problema com worker único. |

---

## 11. Critérios de aceitação

### Consistência transacional

- [ ] Quando a transação de mudança de status commita com sucesso e existe webhook configurado para aquele status, existe exatamente um evento correspondente registrado para despacho.
- [ ] Quando a transação de mudança de status falha (rollback), nenhum evento de notificação é registrado.
- [ ] O evento registrado contém um snapshot do estado do pedido no momento da transição.

### Entrega de notificações

- [ ] O worker opera como processo independente da API.
- [ ] Eventos pendentes são processados e despachados em menos de 10 segundos (p95) após o commit da transação.
- [ ] A notificação é enviada via HTTPS ao endpoint cadastrado pelo cliente.
- [ ] A notificação inclui identificador único do evento, assinatura criptográfica, timestamp e identificador do webhook.

### Retry e DLQ

- [ ] Quando o endpoint do cliente retorna erro ou excede o timeout de 10 segundos, o evento é agendado para retry.
- [ ] O backoff segue a progressão: 1 minuto, 5 minutos, 30 minutos, 2 horas, 12 horas.
- [ ] Após 5 tentativas falhadas, o evento é movido para a DLQ com motivo da falha registrado.
- [ ] Administrador (role ADMIN) consegue reprocessar eventos da DLQ. A ação é registrada para auditoria.

### Segurança

- [ ] Payload é assinado criptograficamente usando a credencial única do endpoint.
- [ ] Credencial é gerada pelo sistema na criação do webhook e devolvida na resposta (única vez visível).
- [ ] Rotação de credencial mantém a credencial anterior válida por 24 horas.
- [ ] URLs HTTP são rejeitadas na validação de cadastro.
- [ ] Endpoint de replay da DLQ exige role ADMIN.

### Configuração e consulta

- [ ] CRUD completo de configuração de webhook funciona corretamente (criar, listar, editar, remover).
- [ ] Filtro de eventos por status funciona: se o webhook não escuta o status de destino, nenhum evento é gerado.
- [ ] Endpoint de histórico de entregas retorna registros com status de sucesso/falha, tempo de resposta e informações de erro.

### Aderência a padrões

- [ ] Módulo segue a estrutura modular existente do projeto.
- [ ] Erros utilizam a hierarquia de erros existente com códigos padronizados e prefixados.
- [ ] Logs utilizam o logger existente.
- [ ] Validações utilizam os schemas de validação do projeto.

---

## 12. Estratégia de testes e validação

### Testes unitários

- Verificar que o registro de evento ocorre de forma atômica com a transação de mudança de status e que o payload gerado contém os campos corretos.
- Testar que eventos só são registrados quando existe webhook configurado para aquele status.
- Testar geração e verificação de assinatura criptográfica com diferentes payloads e credenciais.
- Testar que ambas as credenciais (antiga e nova) são válidas durante o grace period de 24 horas.
- Testar cálculo dos intervalos de backoff.
- Testar rejeição de URLs HTTP e aceitação de HTTPS.

### Testes de integração

- Criar pedido, mudar status e verificar que evento foi registrado. Forçar falha na transação e verificar que evento não existe.
- Testar CRUD completo de webhook com persistência no banco.
- Inserir evento pendente, executar ciclo do worker e verificar que endpoint mock recebeu a notificação com headers e payload corretos.
- Configurar endpoint mock que falha nas primeiras chamadas e depois retorna sucesso. Verificar que o retry funciona conforme o backoff.
- Configurar endpoint mock que sempre falha. Verificar que após 5 tentativas o evento aparece na DLQ com motivo registrado.
- Inserir evento na DLQ, chamar endpoint de replay com role ADMIN, verificar reinserção na fila. Verificar que role sem permissão recebe erro de autorização.

### Revisão de segurança

- Sofia (Engenheira de Segurança) faz revisão dedicada do código de assinatura criptográfica e geração de credenciais antes do deploy. Reservar pelo menos 2 dias úteis. **Fonte:** [09:46] Sofia.
- Testar que payload adulterado falha na verificação de assinatura.
- Testar que cadastro com URL HTTP é rejeitado.
- Testar que endpoint de replay exige role ADMIN.

### Validação com clientes

- Disponibilizar ambiente de staging para os 3 clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) testarem a integração antes do deploy em produção.
- Marcos (PM) prepara documentação no portal do desenvolvedor explicando formato do payload, headers, verificação de assinatura, semântica at-least-once e deduplicação via identificador único do evento. **Fonte:** [09:26] Marcos.

### Critérios de go/no-go para produção

1. Todos os testes unitários e de integração passam.
2. Revisão de segurança da Sofia aprovada.
3. Pelo menos 1 cliente B2B validou a integração em staging.
4. Worker executou por pelo menos 48 horas em staging sem erros não tratados.

**Nota:** O critério de 48 horas de execução em staging não foi discutido na reunião. É uma recomendação de produto para garantir estabilidade antes do lançamento. Deve ser validado com a tech lead.
