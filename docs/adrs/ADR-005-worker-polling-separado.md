# ADR-005: Worker em processo separado com polling a cada 2 segundos

## Status

Aceita

## Contexto

A decisão de usar Transactional Outbox no MySQL (ADR-001) requer um worker que leia eventos pendentes da tabela `webhook_outbox` e realize o dispatch HTTP. Duas questões precisam ser respondidas:

1. **Como o worker detecta novos eventos?** — O MySQL não oferece mecanismo nativo de notificação a processos externos (como `NOTIFY/LISTEN` do PostgreSQL). Triggers no MySQL apenas executam SQL dentro do próprio banco; não conseguem notificar um processo Node.js diretamente sem improvisações frágeis (escrita em arquivo, chamada a endpoint).

2. **Onde o worker executa?** — A API roda em `src/server.ts`. Se o worker rodar no mesmo processo, uma reinicialização ou crash da API derruba o worker junto, causando acúmulo de eventos pendentes sem processamento.

O requisito de negócio é que webhooks sejam entregues em menos de 10 segundos após a mudança de status. A equipe é pequena e deseja manter a stack simples (Node.js + Prisma + MySQL).

## Decisão

O worker será implementado como um **processo Node.js separado**, com entry-point próprio em `src/worker.ts`, executado via script `npm run worker`.

O mecanismo de leitura será **polling com intervalo fixo de 2 segundos**:

- A cada ciclo, o worker consulta a tabela `webhook_outbox` buscando registros com `status = 'pending'` ordenados por `created_at ASC`.
- Processa os eventos encontrados (dispatch HTTP + atualização de status).
- Aguarda 2 segundos antes do próximo ciclo.

O worker utiliza a mesma configuração de banco definida em `src/config/database.ts` e instancia seu próprio PrismaClient (necessário por ser um processo Node.js distinto).

Inicialmente haverá um único worker. O ordenamento de processamento é garantido implicitamente pela ordenação `created_at` dentro de cada `order_id`. Não há garantia de ordenamento global caso múltiplos workers sejam adicionados no futuro — isso é aceito como limitação conhecida.

## Alternativas Consideradas

### 1. MySQL trigger para notificação externa

Triggers no MySQL executam apenas SQL dentro do banco. Para notificar um processo externo, seria necessário improvisar — por exemplo, gravar em arquivo ou chamar endpoint via UDF. Essas abordagens são frágeis, difíceis de monitorar e fogem do modelo operacional da equipe. Descartada.

### 2. Worker dentro do processo da API

Rodar o loop de polling como thread/intervalo dentro de `src/server.ts`. Descartada porque:

- Reinicialização da API (deploy, crash, restart) interrompe o worker.
- Acúmulo de eventos pendentes durante indisponibilidade do processo.
- Mistura responsabilidades (servir HTTP e processar background jobs) dificultando debug e scaling independente.

### 3. Redis ou message broker (RabbitMQ, SQS)

Introduzir um broker permitiria push-based com latência menor. Descartada porque:

- Adiciona infraestrutura que a equipe precisa provisionar, monitorar e manter.
- Introduz risco de inconsistência entre MySQL e broker (dois datastores sem transação distribuída).
- Overengineering para o volume e tamanho de equipe atuais.
- O polling de 2 segundos já atende ao requisito de entrega em menos de 10 segundos.

## Consequências

### Positivas

- **Simplicidade** — nenhuma infraestrutura adicional; mesmo banco, mesma stack (Node.js + Prisma).
- **Resiliência da API** — crash ou restart de `src/server.ts` não afeta o worker; eventos continuam sendo processados.
- **Requisito atendido** — latência máxima de 2 segundos (pior caso) entre gravação do evento e início do processamento, bem abaixo do limite de 10 segundos.
- **Facilidade de deploy** — entry-point dedicado (`src/worker.ts`) com script `npm run worker`, sem alterar a API existente.
- **Ordenamento por pedido** — com worker único, eventos de um mesmo `order_id` são processados na ordem de criação.

### Negativas

- **Latência mínima de até 2 segundos** — no pior caso, um evento espera quase 2 segundos para ser lido. Aceito como trade-off pela simplicidade.
- **Polling consome recursos** — a query executa a cada 2 segundos mesmo sem eventos pendentes. Em volume baixo, o custo é desprezível; o índice `(status, created_at)` garante leitura eficiente.
- **Sem garantia de ordenamento global** — se no futuro houver múltiplos workers, eventos de pedidos diferentes podem ser processados fora de ordem. Documentado como limitação conhecida; para o cenário atual (worker único) não há impacto.
- **Processo adicional para gerenciar** — necessita supervisão (systemd, pm2 ou container dedicado) para garantir que o worker reinicie em caso de crash.
