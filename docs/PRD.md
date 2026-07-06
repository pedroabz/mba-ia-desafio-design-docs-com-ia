# PRD: Notificações em Tempo Real de Status de Pedidos via Webhooks

## Metadados

| Campo       | Valor                                                          |
|-------------|----------------------------------------------------------------|
| Autor       | Marcos (Product Manager)                                       |
| Status      | Em revisão                                                     |
| Data        | 2026-07-05                                                     |
| Revisores   | Larissa (Tech Lead), Marcos (PM), Bruno, Diego, Sofia          |

---

## 1. Resumo e contexto da feature

O sistema de gestão de pedidos (OMS) atende clientes B2B que dependem de informações atualizadas sobre o andamento dos seus pedidos. Hoje, esses clientes precisam consultar a plataforma repetidamente para verificar se houve alguma mudança de status -- um processo manual, lento e custoso para todos os envolvidos.

Esta feature introduz um sistema de notificações automáticas (webhooks outbound) que avisa os clientes proativamente sempre que o status de um pedido muda. Em vez de o cliente ir até a plataforma buscar a informação, a plataforma envia a informação até o cliente, em tempo quase real.

---

## 2. Problema e motivação

### O problema atual

Os clientes B2B da plataforma não possuem nenhum mecanismo para serem avisados quando o status dos seus pedidos muda. A única alternativa disponível é consultar a API de pedidos periodicamente, verificando um a um se houve alteração. Essa abordagem traz três problemas concretos:

- **Integração lenta:** o cliente só descobre que um pedido avançou na próxima vez que fizer a consulta, o que pode levar minutos ou horas dependendo da frequência configurada.
- **Custo operacional elevado:** cada consulta consome recursos tanto do lado do cliente quanto do lado da plataforma, mesmo quando não há nenhuma novidade.
- **Experiência inferior à concorrência:** clientes que operam com outros fornecedores já recebem notificações automáticas e esperam o mesmo nível de serviço.

### Quem está pedindo

Três clientes B2B formalizaram o pedido na semana anterior à reunião de definição:

- **Atlas Comercial:** sinalizou que pode migrar para um concorrente caso a funcionalidade não seja entregue até o fim do trimestre. É o caso mais urgente.
- **MaxDistribuição:** necessita de integração em tempo real para alimentar seus sistemas logísticos internos.
- **Nova Cargo:** busca automatizar o acompanhamento de entregas, eliminando verificações manuais.

### O que "tempo real" significa para esses clientes

Foi validado diretamente com os clientes que qualquer notificação entregue em menos de 10 segundos após a mudança de status já atende à expectativa de "tempo real". O ponto crítico não é a latência exata, mas sim a eliminação da necessidade de consultas manuais e periódicas.

---

## 3. Público-alvo e cenários de uso

### Público-alvo

Clientes B2B da plataforma que possuem sistemas próprios capazes de receber notificações automáticas. São empresas com equipes técnicas que mantêm integrações automatizadas com fornecedores.

### Cenários de uso

1. **Atualização logística automatizada:** um cliente configura sua URL de recebimento e seleciona os status "enviado" e "entregue". Sempre que um pedido transita para um desses status, o sistema do cliente é notificado e atualiza automaticamente seu painel de rastreamento, sem intervenção humana.

2. **Integração financeira:** um cliente deseja ser avisado quando um pedido é marcado como "pago". Ao receber a notificação, seu sistema financeiro interno concilia automaticamente o pagamento, eliminando a verificação manual diária.

3. **Monitoramento de cancelamentos:** um cliente quer ser alertado imediatamente quando qualquer pedido é cancelado, para acionar processos internos de reversão de estoque e reembolso.

4. **Múltiplos destinos por cliente:** um mesmo cliente configura dois destinos diferentes -- um para o sistema logístico (ouvindo status de envio) e outro para o financeiro (ouvindo status de pagamento), cada um recebendo apenas os eventos relevantes.

5. **Diagnóstico de problemas de integração:** o responsável técnico do cliente consulta o histórico de entregas para verificar quais notificações foram enviadas, se houve falhas e qual foi o tempo de resposta, facilitando a depuração sem precisar contatar o suporte.

6. **Reprocessamento administrativo:** um administrador da plataforma identifica que notificações falharam porque o cliente estava em manutenção planejada. Após o cliente voltar ao ar, o administrador reprocessa manualmente as notificações que não foram entregues.

---

## 4. Objetivos e métricas de sucesso

### Objetivos

- Eliminar a dependência dos clientes B2B em consultas periódicas para acompanhar mudanças de status de pedidos.
- Entregar notificações em tempo quase real (abaixo de 10 segundos) após cada transição de status.
- Oferecer um mecanismo confiável, seguro e compatível com padrões de mercado que os clientes já conhecem.
- Reter a Atlas Comercial, que sinalizou possível migração para concorrente.

### Métricas de sucesso

| Métrica | Meta | Justificativa |
|---------|------|---------------|
| Latência de notificação | Inferior a 10 segundos entre a mudança de status e a chegada ao cliente | Requisito validado diretamente com os clientes |
| Taxa de entrega bem-sucedida na primeira tentativa | Acima de 95% | Indica que o mecanismo de envio funciona de forma confiável na maioria dos casos |
| Taxa de entrega final (incluindo reenvios) | Acima de 99,5% | O sistema cobre falhas temporárias com reenvios automáticos por até ~15 horas |
| Adoção pelos clientes solicitantes | 3 clientes (Atlas, MaxDistribuição, Nova Cargo) configurados até o fim do trimestre | Validação de que a feature atende à demanda que a originou |
| Redução de consultas periódicas | Redução mensurável no volume de consultas à API de pedidos pelos clientes que adotarem notificações automáticas | Confirma que a feature elimina a necessidade de consultas repetitivas |

**Nota:** as metas quantitativas de taxa de entrega (95% e 99,5%) não foram definidas explicitamente na reunião. São sugestões baseadas no comportamento esperado do sistema conforme descrito. Devem ser validadas com o time antes de serem consideradas compromissos formais.

---

## 5. Escopo

### Incluso nesta fase

- Cadastro, edição, consulta e remoção de configurações de webhook por cliente, via API.
- Seleção dos eventos de interesse por configuração (o cliente escolhe quais mudanças de status deseja receber).
- Envio automático de notificações quando o status de um pedido muda, com latência inferior a 10 segundos.
- Assinatura de segurança em cada notificação, com credencial exclusiva por destino, permitindo ao cliente validar autenticidade e integridade.
- Rotação de credenciais de segurança via API, com período de transição de 24 horas para migração sem interrupção.
- Obrigatoriedade de conexão segura (HTTPS) nos destinos de notificação.
- Reenvio automático em caso de falha, com intervalos crescentes, cobrindo um período de até aproximadamente 15 horas.
- Registro de notificações que falharam permanentemente em área separada, com possibilidade de reprocessamento manual por administradores.
- Histórico de entregas consultável via API (sucesso, falha, tempo de resposta).
- Identificador único por notificação, permitindo ao cliente tratar eventuais duplicatas.
- Garantia de que toda notificação é entregue pelo menos uma vez.

### Fora do escopo (descartados na reunião)

- **Notificação por email em caso de falhas recorrentes:** Marcos levantou a possibilidade de avisar o cliente por email quando o webhook dele falha repetidamente. Larissa descartou para esta fase, com reavaliação futura após medir o impacto em produção.
- **Dashboard visual para clientes:** Marcos sugeriu um painel gráfico para o cliente acompanhar seus webhooks. Larissa definiu que é projeto separado do time de frontend, fora do escopo desta entrega.
- **Controle de volume de envio por cliente:** Diego levantou a questão de clientes que poderiam receber muitas notificações em pouco tempo. O time decidiu observar o comportamento em produção antes de implementar qualquer mecanismo de limitação.
- **Estratégia de limpeza de dados históricos:** mencionada a possibilidade de arquivar registros antigos (sugestão de 30 dias), mas explicitamente colocada fora do escopo desta feature.

---

## 6. Requisitos funcionais

| ID   | Requisito |
|------|-----------|
| RF01 | O sistema deve permitir cadastrar uma configuração de webhook para um cliente, informando a URL de destino e os eventos de interesse (status que deseja receber). A credencial de segurança é gerada automaticamente pelo sistema e devolvida na criação. |
| RF02 | O sistema deve permitir editar uma configuração de webhook existente, alterando URL, eventos de interesse e estado ativo/inativo. |
| RF03 | O sistema deve permitir remover uma configuração de webhook. |
| RF04 | O sistema deve permitir listar as configurações de webhook de um cliente. A credencial de segurança não deve ser exibida na listagem. |
| RF05 | Quando o status de um pedido mudar, o sistema deve enviar automaticamente uma notificação para todos os webhooks ativos do cliente cujos eventos de interesse incluam o novo status. Se nenhum webhook estiver configurado para aquele status, nenhuma notificação é gerada. |
| RF06 | Se o status do pedido mudou, a notificação deve ter sido registrada. Se houve falha na mudança de status, nenhuma notificação é gerada. Não pode haver cenário em que o status muda sem que a notificação correspondente seja registrada, nem cenário em que uma notificação é registrada sem que o status tenha efetivamente mudado. |
| RF07 | A notificação enviada deve conter: identificador único do evento, tipo do evento, data e hora, identificador do pedido, número do pedido, status anterior, status novo, identificador do cliente e valor total. Os itens do pedido não são incluídos na notificação; o cliente consulta a API de pedidos caso precise de detalhes. |
| RF08 | Cada notificação deve ser assinada com uma credencial de segurança exclusiva do destino. O cliente deve conseguir validar que a notificação veio realmente da plataforma e que o conteúdo não foi alterado. |
| RF09 | O sistema deve permitir rotacionar a credencial de segurança de um webhook via API. Ao rotacionar, a credencial anterior permanece válida por 24 horas, dando tempo ao cliente para migrar seus sistemas. |
| RF10 | A URL de destino do webhook deve obrigatoriamente utilizar conexão segura (HTTPS). Tentativas de cadastro com URL insegura devem ser recusadas. |
| RF11 | Em caso de falha na entrega de uma notificação, o sistema deve reenviar automaticamente com intervalos crescentes, realizando até 5 tentativas ao longo de aproximadamente 15 horas. Se todas as tentativas falharem, a notificação é movida para uma área de falhas permanentes. |
| RF12 | Um administrador deve conseguir reprocessar manualmente uma notificação que falhou permanentemente, reinserindo-a na fila de envio. Essa ação deve ficar registrada para fins de auditoria. |
| RF13 | O sistema deve disponibilizar um histórico de entregas por webhook, contendo todas as tentativas realizadas (sucesso e falha), com informações como código de resposta, tempo de resposta e conteúdo enviado. |
| RF14 | Cada notificação deve carregar um identificador único que permanece o mesmo em todas as tentativas de entrega, permitindo ao cliente identificar e descartar duplicatas. |

---

## 7. Requisitos não funcionais

| ID    | Requisito |
|-------|-----------|
| RNF01 | A latência entre a mudança de status do pedido e o envio da notificação deve ser inferior a 10 segundos no pior caso. |
| RNF02 | Notificações cujo conteúdo exceda 64 KB devem ser rejeitadas na origem, sem envio. |
| RNF03 | Chamadas ao destino do cliente devem ser interrompidas após 10 segundos sem resposta, sendo tratadas como falha. |
| RNF04 | Credenciais de segurança dos webhooks não devem aparecer em nenhum registro de log da aplicação. |
| RNF05 | O processo que envia as notificações deve operar de forma independente da API principal. Uma falha ou reinício da API não deve interromper o envio de notificações, e vice-versa. |
| RNF06 | A feature deve seguir os mesmos padrões e convenções já adotados pelo restante da plataforma, sem introdução de dependências externas adicionais. |
| RNF07 | O sistema deve garantir que toda notificação é entregue pelo menos uma vez. Em caso de incerteza (por exemplo, falha de comunicação após envio), a plataforma reenvia e o cliente é responsável por tratar duplicatas com base no identificador único. |

---

## 8. Decisões e trade-offs principais

As decisões abaixo foram tomadas durante a reunião de definição e refletem trade-offs conscientes entre simplicidade, custo e robustez.

1. **Notificação registrada junto com a mudança de status, sem risco de perda.**
   O registro da notificação ocorre de forma indissociável da mudança de status do pedido. Se o status mudou, a notificação foi registrada. Se houve falha, nenhuma notificação é gerada. Essa garantia foi priorizada em detrimento de alternativas mais simples que poderiam perder notificações em cenários de falha.

2. **Envio desacoplado da operação de mudança de status.**
   O envio propriamente dito é feito por um processo dedicado, não no momento em que o status muda. Isso evita que um destino lento ou indisponível impacte a operação de mudança de status para todos os demais pedidos. O trade-off é uma latência de até alguns segundos entre a mudança e o recebimento pelo cliente, considerada aceitável.

3. **Reutilização da infraestrutura existente, sem componentes adicionais.**
   O time optou por utilizar apenas os componentes já presentes na plataforma (mesmo banco de dados, mesma linguagem, mesmas bibliotecas), evitando introduzir sistemas de mensageria ou filas externas. Isso reduz o custo operacional e a complexidade, ao custo de limitações de escala que serão reavaliadas se necessário.

4. **Entrega garantida pelo menos uma vez, com responsabilidade de deduplicação no cliente.**
   Em vez de tentar garantir entrega exatamente uma vez (o que exigiria coordenação complexa entre os dois lados), o sistema garante que nenhuma notificação é perdida silenciosamente, podendo em raros casos enviar a mesma notificação mais de uma vez. Cada notificação carrega um identificador único para que o cliente possa tratar duplicatas. Essa é a mesma abordagem adotada por plataformas de referência no mercado.

5. **Credencial de segurança isolada por destino.**
   Cada destino de notificação possui sua própria credencial. Se uma credencial for comprometida, apenas aquele destino específico é afetado, sem impacto nos demais clientes. A rotação é autoatendimento, via API, com período de transição de 24 horas.

6. **Ordenação por pedido, não global.**
   Notificações de um mesmo pedido são entregues na ordem em que ocorreram. Não há garantia de ordenação global entre pedidos diferentes. Os clientes nunca solicitaram ordenação global -- o interesse é saber se cada pedido individual avançou.

---

## 9. Dependências

| Dependência | Descrição | Impacto |
|-------------|-----------|---------|
| API de pedidos (módulo existente) | A geração de notificações depende diretamente da operação de mudança de status de pedidos, que é o ponto de integração com o sistema existente. | Qualquer alteração futura nessa operação deve considerar o impacto no registro de notificações. |
| Módulo de autenticação existente | As operações de configuração de webhooks utilizam o sistema de autenticação e autorização já presente na plataforma. | Nenhuma alteração necessária no módulo de autenticação. |
| Módulo de clientes existente | A configuração de webhook é vinculada a um cliente cadastrado na plataforma. | O cliente deve existir previamente no sistema. |
| Revisão de segurança pela Sofia | A engenheira de segurança solicitou pelo menos dois dias úteis para revisar a implementação das credenciais e da assinatura de notificações antes do deploy. | Bloqueia o deploy final; deve ser agendada com antecedência. |
| Documentação para clientes | Marcos se comprometeu a documentar a integração no portal de desenvolvedores, incluindo orientações sobre tratamento de duplicatas e validação de assinatura. | Não bloqueia o desenvolvimento, mas é necessária antes que os clientes comecem a integrar. |

---

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|-------|---------------|---------|-----------|
| 1 | **Acúmulo de notificações pendentes por falha no processo de envio.** Se o processo responsável pelo envio parar de funcionar, as notificações se acumulam sem serem enviadas, causando atraso crescente para todos os clientes. | Baixa | Alto | Monitorar o volume de notificações pendentes e configurar alertas quando ultrapassar um limite aceitável. O processo de envio opera de forma independente da API, reduzindo os cenários de falha simultânea. |
| 2 | **Cliente com destino permanentemente indisponível gerando reenvios desnecessários.** Um cliente que desativa seu destino de recebimento sem remover a configuração do webhook gera tentativas de entrega que consomem recursos sem resultado. | Média | Baixo | Após 5 tentativas distribuídas ao longo de ~15 horas, a notificação é movida para a área de falhas permanentes, cessando os reenvios automáticos. O administrador pode reprocessar manualmente quando o cliente voltar. |
| 3 | **Comprometimento de credencial de segurança.** Se a credencial de assinatura de um webhook for exposta (já houve caso de cliente que vazou credencial em log de aplicação), notificações para aquele destino podem ser falsificadas por terceiros. | Baixa | Médio | Cada destino possui credencial própria, isolando o impacto a um único destino. A rotação via API com período de transição de 24 horas permite substituição rápida sem interrupção do serviço. |
| 4 | **Prazo insuficiente para entrega antes do fim do trimestre.** A estimativa é de três sprints, incluindo revisão de segurança. Qualquer imprevisto pode comprometer a entrega dentro da janela comunicada à Atlas Comercial. | Média | Alto | Marcos confirma prazo com a Atlas. O escopo foi deliberadamente limitado (sem email, sem dashboard, sem controle de volume) para viabilizar a entrega no prazo. Itens descartados podem ser adicionados em fases futuras. |
| 5 | **Crescimento da base de dados de notificações sem estratégia de limpeza.** A estratégia de arquivamento de registros antigos ficou fora do escopo desta fase. Em produção com alto volume, isso pode degradar a performance do envio. | Média | Médio | Registrado como ponto em aberto. Sugestão de limpeza a cada 30 dias foi mencionada na reunião, mas precisa ser definida e implementada antes que o volume se torne problemático. |

---

## 11. Critérios de aceitação

1. Um cliente pode cadastrar, editar, consultar e remover configurações de webhook pela API, selecionando quais eventos deseja receber.
2. Quando o status de um pedido muda para um status monitorado por um webhook ativo, o cliente recebe a notificação em menos de 10 segundos.
3. Quando o status de um pedido muda para um status que nenhum webhook monitora, nenhuma notificação é gerada.
4. Se a mudança de status falhar por qualquer motivo, nenhuma notificação é registrada.
5. Cada notificação é assinada com a credencial exclusiva do destino, e o cliente pode validar a autenticidade recalculando a assinatura.
6. Após rotação de credencial, a credencial anterior permanece válida por 24 horas.
7. URLs de destino sem HTTPS são rejeitadas no cadastro.
8. Em caso de falha, o sistema retenta automaticamente até 5 vezes com intervalos crescentes ao longo de ~15 horas.
9. Após esgotadas as tentativas, a notificação aparece na área de falhas permanentes e pode ser reprocessada por um administrador.
10. O histórico de entregas de cada webhook é consultável pela API, mostrando sucessos, falhas, tempo de resposta e conteúdo enviado.
11. A mesma notificação reenviada carrega o mesmo identificador único, permitindo ao cliente descartar duplicatas.
12. Uma falha ou reinício da API não interrompe o envio de notificações pendentes.
13. Credenciais de segurança dos webhooks não aparecem em logs nem em respostas de listagem.

---

## 12. Estratégia de testes e validação

### Validação funcional

- Configuração completa de webhook via API: criação, edição de URL e eventos, desativação e remoção. Verificar que a credencial é retornada apenas na criação e na rotação.
- Mudança de status de pedido com webhook ativo configurado: confirmar que a notificação é enviada ao destino dentro do tempo esperado, com o conteúdo correto (dados do pedido e status anterior/novo).
- Mudança de status de pedido sem webhook configurado para aquele status: confirmar que nenhuma notificação é gerada.
- Filtragem de eventos: configurar webhook para ouvir apenas dois status, mudar o pedido para um terceiro status e verificar que nada é enviado.

### Validação de segurança

- Validação de assinatura: receber notificação e confirmar que a assinatura de segurança permite verificar a autenticidade e integridade do conteúdo.
- Rotação de credencial: rotacionar, enviar notificação com a nova credencial e validar que a credencial anterior ainda funciona dentro de 24 horas.
- Rejeição de URL HTTP: tentar cadastrar webhook com URL sem HTTPS e confirmar recusa.
- Ausência de credenciais em logs e listagens: verificar que nenhuma resposta de consulta nem linha de log contém o valor da credencial.

### Validação de resiliência

- Falha de entrega e reenvio: simular destino indisponível, verificar que o sistema retenta nos intervalos esperados.
- Falha permanente: simular 5 falhas consecutivas e confirmar que a notificação aparece na área de falhas permanentes.
- Reprocessamento: executar reprocessamento via API administrativa e confirmar que a notificação volta a ser enviada.
- Isolamento de processos: reiniciar a API durante o envio de notificações e confirmar que o processo de envio continua operando normalmente.

### Validação de integridade

- Consistência entre mudança de status e notificação: forçar falha na mudança de status e confirmar que nenhuma notificação foi registrada.
- Identificador de deduplicação: provocar reenvio da mesma notificação e confirmar que o identificador permanece o mesmo em todas as tentativas.
- Conteúdo da notificação como instantâneo: mudar o pedido novamente após a primeira mudança de status e confirmar que a notificação original contém os dados do momento em que foi registrada, não os dados atuais.

### Prazo estimado

Três sprints, incluindo revisão de segurança por Sofia (mínimo de dois dias úteis reservados antes do deploy). Estimativa definida por Larissa na reunião e confirmada pelo time.

---

## Documentos relacionados

- [RFC: Sistema de Webhooks](RFC.md) -- detalhamento técnico da solução proposta, alternativas consideradas e questões em aberto
- [FDD: Sistema de Webhooks](FDD.md) -- especificação de implementação com contratos de API, fluxos internos e modelos de dados
- [ADR-001 a ADR-006](adrs/) -- registro das decisões arquiteturais e seus racionais
