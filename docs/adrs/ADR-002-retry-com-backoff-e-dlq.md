# ADR-002: Política de retry com backoff exponencial e DLQ

**Status:** Aceita
**Data:** 2026-07-05
**Decisores:** Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno), Larissa (Tech Lead)

---

## Contexto

O sistema de webhooks precisa lidar com falhas de entrega: endpoints de clientes podem estar temporariamente indisponíveis por manutenção planejada, problemas de rede ou instabilidades. A reunião identificou que já houve clientes com indisponibilidade de até duas horas em manutenção planejada ([09:16] Diego). É necessário definir quantas vezes o sistema retenta, com quais intervalos, e o que acontece quando todas as tentativas se esgotam.

## Decisão

Adotar **5 tentativas com backoff exponencial** e **DLQ (Dead Letter Queue) em tabela separada** no MySQL.

### Progressão do backoff

| Tentativa | Intervalo após falha |
|-----------|---------------------|
| 1a        | 1 minuto            |
| 2a        | 5 minutos           |
| 3a        | 30 minutos          |
| 4a        | 2 horas             |
| 5a        | 12 horas            |

Tempo total entre a primeira falha e a última tentativa: aproximadamente 15 horas.

### Dead Letter Queue

Após a 5a tentativa falhar, o evento será movido para uma tabela separada (`webhook_dead_letter`) contendo:
- Payload original do evento
- Motivo da falha
- Timestamp

A separação em tabela própria mantém a tabela outbox principal limpa para leitura do worker e serve como evidência para debug e reprocessamento ([09:18] Diego).

### Reprocessamento

Eventos na DLQ poderão ser reprocessados manualmente via endpoint administrativo (`POST /admin/webhooks/dead-letter/:id/replay`), que recolocará o evento na outbox como pendente. Esse endpoint exigirá role `ADMIN` e registrará auditoria de quem executou o replay ([09:36] Sofia).

## Alternativas Consideradas

### 1. Três tentativas (proposta do Bruno)

Bruno sugeriu 3 retries como abordagem mais agressiva ([09:16]).

**Rejeitada porque:**
- Com 3 tentativas, a janela total de cobertura seria de aproximadamente 30 minutos, insuficiente para cobrir indisponibilidades de manutenção planejada que já foram observadas em clientes (2 horas).
- O evento seria descartado prematuramente, gerando perda de notificações para o cliente.

### 2. Retry indefinido com backoff

Retentar infinitamente com intervalos crescentes até que o endpoint responda.

**Rejeitada porque:**
- Eventos ficariam pendurados indefinidamente na outbox caso o cliente sumisse ou desativasse o endpoint ([09:15] Diego).
- A tabela outbox cresceria sem limite, degradando a performance do worker.
- Não há mecanismo para distinguir entre indisponibilidade temporária e abandono permanente do endpoint.

### 3. DLQ na própria tabela outbox (marcação de status "failed")

Marcar eventos como "failed" na própria tabela outbox em vez de mover para tabela separada.

**Rejeitada porque:**
- Polui a tabela principal com registros que não serão mais processados pelo worker, complicando queries de leitura de eventos pendentes.
- Mistura responsabilidades: a outbox deve conter apenas eventos a serem processados.

## Consequências

### Positivas

- **Cobertura de janela adequada:** 15 horas de cobertura atendem cenários reais de manutenção planejada e indisponibilidades temporárias.
- **Tabela outbox limpa:** eventos finalizados (entregues ou movidos para DLQ) não poluem a fila de processamento do worker.
- **Rastreabilidade:** a DLQ serve como registro histórico de falhas permanentes, facilitando debug e auditoria.
- **Recuperação manual:** o endpoint de replay permite reprocessar eventos sem intervenção direta no banco de dados.

### Negativas

- **Complexidade adicional:** o worker precisa gerenciar contagem de tentativas, cálculo de próximo retry e lógica de movimentação para DLQ.
- **Latência de recuperação:** no pior caso, um evento pode levar até 15 horas para ser considerado falha permanente, e o cliente só saberá do problema se consultar o histórico de entregas.

### Trade-off explícito

Aceita-se a complexidade de gerenciamento de retries e DLQ em troca de maximizar a chance de entrega bem-sucedida dentro de uma janela temporal realista, sem manter eventos pendurados indefinidamente.
