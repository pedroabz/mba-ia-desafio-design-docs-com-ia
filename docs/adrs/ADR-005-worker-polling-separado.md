# ADR-005: Worker em processo separado com polling de 2 segundos

**Status:** Aceita
**Data:** 2026-07-05
**Decisores:** Diego (Engenheiro Sênior), Bruno (Engenheiro Pleno), Larissa (Tech Lead)

---

## Contexto

O padrão outbox (ADR-001) exige um componente que leia a tabela de eventos pendentes e dispare as chamadas HTTP para os endpoints dos clientes. É necessário definir como esse componente será executado (dentro da API ou como processo separado), como ele detectará novos eventos (polling ou mecanismo reativo) e qual será a latência aceitável.

O requisito de negócio é que a notificação chegue em menos de 10 segundos após a mudança de status ([09:02] Marcos).

O projeto atualmente possui um único entry-point em `src/server.ts`, que inicializa a aplicação Express, conecta ao banco via Prisma Client (`src/config/database.ts`) e configura graceful shutdown para `SIGINT`/`SIGTERM`.

## Decisão

O worker de despacho de webhooks será executado como **processo Node.js separado**, utilizando **polling a cada 2 segundos** na tabela outbox.

### Processo separado

- O worker terá seu próprio entry-point (analogamente ao `src/server.ts` existente para a API).
- Utilizará sua própria instância de `PrismaClient`, conectando ao mesmo banco de dados (`DATABASE_URL`), pois `PrismaClient` é por processo ([09:30] Bruno).
- Será iniciado por um script dedicado (por exemplo, `npm run worker`).
- O ciclo de vida do worker será independente da API: se a API reiniciar, o worker continua operando, e vice-versa ([09:11] Diego).

### Polling de 2 segundos

- A cada 2 segundos, o worker consultará a tabela outbox buscando eventos com status pendente, ordenados por `created_at`.
- Processará os eventos em batch pequeno, marcando cada um como processado ou agendando retry conforme ADR-002.
- A latência no pior caso será de 2 segundos (evento inserido imediatamente após o último ciclo de polling), bem dentro do requisito de 10 segundos.

### Ordenação

- Com um único worker, a ordenação de eventos é garantida implicitamente por `order_id` e `created_at`: eventos do mesmo pedido são processados na sequência em que foram inseridos.
- Esta garantia é válida **apenas enquanto houver um único worker**. Caso haja necessidade futura de escalar para múltiplos workers, será necessário implementar particionamento por `order_id` ou lock pessimista. Isso está documentado como limitação conhecida ([09:13] Diego, Larissa).

## Alternativas Consideradas

### 1. Trigger do MySQL

Utilizar triggers do banco de dados para notificar o worker quando um novo evento fosse inserido na outbox.

**Rejeitada porque:**
- MySQL não possui mecanismo nativo de notificação a processos externos (como o `NOTIFY/LISTEN` do PostgreSQL) ([09:09] Diego).
- Triggers MySQL executam apenas SQL; para avisar um processo externo, seria necessário improvisar soluções frágeis como escrever em arquivo ou chamar endpoints, o que introduz acoplamento e pontos de falha ([09:09] Diego).

### 2. Worker dentro do processo da API

Executar o loop de polling como uma goroutine/setInterval dentro do mesmo processo Node.js que serve a API.

**Rejeitada porque:**
- Se a API reiniciar (deploy, crash, scaling), o worker para junto, causando atraso no despacho de webhooks ([09:11] Diego).
- Mistura responsabilidades: o processo da API deve focar em servir requisições HTTP, enquanto o worker tem um ciclo de vida e características de consumo de recursos diferentes.
- Dificulta o monitoramento e scaling independente de cada componente.

## Consequências

### Positivas

- **Isolamento de falhas:** crash ou restart da API não afeta o despacho de webhooks, e vice-versa.
- **Latência aceitável:** polling de 2 segundos atende com folga o requisito de notificação em menos de 10 segundos.
- **Reuso de infraestrutura:** mesmo banco de dados, mesma stack (Node.js, TypeScript, Prisma), sem necessidade de tecnologias adicionais.
- **Simplicidade operacional:** um `setInterval` ou loop com `setTimeout` é significativamente mais simples de implementar e depurar do que integração com triggers ou mecanismos reativos.

### Negativas

- **Custo de polling:** o worker executa queries no banco a cada 2 segundos mesmo quando não há eventos pendentes, consumindo (minimamente) recursos de conexão e CPU do MySQL.
- **Latência não-zero:** há até 2 segundos de atraso entre a inserção do evento na outbox e sua detecção pelo worker. Para o caso de uso atual isso é aceitável, mas não é adequado para cenários que exijam sub-segundo.
- **Limitação de escala:** a garantia de ordenação depende de single-worker. Escalar horizontalmente exigirá trabalho adicional (particionamento ou locking).

### Trade-off explícito

Aceita-se a latência de até 2 segundos e o custo de polling constante em troca de simplicidade de implementação, isolamento de processos e reutilização da infraestrutura existente sem necessidade de mecanismos reativos ou tecnologias adicionais.
