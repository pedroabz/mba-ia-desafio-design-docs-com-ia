# ADR-002: Política de retry com backoff exponencial e Dead Letter Queue

## Status

Aceita

## Contexto

O worker de webhooks (definido na ADR-001) realiza o dispatch HTTP para sistemas externos. Quando o cliente consumidor está offline ou retorna erro, o sistema precisa de uma estratégia clara para retentar a entrega sem bloquear a fila de eventos pendentes nem acumular retries infinitos.

Cenários reais observados pela equipe incluem:

- Clientes com janelas de manutenção planejada de até 2 horas.
- Clientes que ficam indisponíveis durante toda uma manhã por problemas de infraestrutura.
- Clientes que desaparecem permanentemente (cancelamento, falha crítica sem recuperação).

Sem uma política definida, eventos falhos ficariam marcados como `pending` ou `processing` indefinidamente, consumindo recursos do worker e dificultando a identificação de falhas reais versus indisponibilidades temporárias.

## Decisão

Adotar **5 tentativas de retry com backoff exponencial fixo** e mover eventos que esgotaram as tentativas para uma **Dead Letter Queue (DLQ)** implementada como tabela separada.

### Progressão de backoff

| Tentativa | Intervalo desde a anterior |
|-----------|---------------------------|
| 1a        | imediata (falha original) |
| 2a        | 1 minuto                  |
| 3a        | 5 minutos                 |
| 4a        | 30 minutos                |
| 5a        | 2 horas                   |
| 6a (DLQ)  | 12 horas                  |

O tempo total entre a primeira falha e a última tentativa é de aproximadamente **15 horas**. Se o cliente permanecer indisponível após esse período, o evento é considerado falha permanente.

### Dead Letter Queue

- Tabela separada `webhook_dead_letter` com colunas: `id`, `outbox_id`, `payload`, `failure_reason`, `last_http_status`, `attempts_count`, `created_at`.
- Eventos movidos para DLQ são removidos da `webhook_outbox` (não ficam marcados como "failed" na outbox), mantendo a tabela principal limpa.
- Replay manual via endpoint administrativo: `POST /admin/webhooks/dead-letter/:id/replay`, que reinsere o evento na `webhook_outbox` com status `pending` para nova tentativa completa.

## Alternativas Consideradas

### 1. Três tentativas com backoff agressivo

Proposta por Bruno durante a discussão. Utilizaria intervalos menores e descartaria mais rápido.

**Motivo da rejeição:** três tentativas em intervalos curtos (~30 minutos no total) não cobrem cenários reais de manutenção planejada de 2 horas. A equipe já observou clientes com indisponibilidades dessa duração, o que resultaria em falsos positivos frequentes na DLQ e necessidade constante de replay manual.

### 2. Retry infinito com backoff exponencial (sem limite de tentativas)

O worker retentaria indefinidamente, aumentando o intervalo progressivamente.

**Motivo da rejeição:** se o cliente desapareceu permanentemente (cancelamento, empresa fechou), o evento ficaria pendurado para sempre na outbox, consumindo espaço e ciclos de polling. Além disso, dificulta a visibilidade operacional — a equipe não conseguiria distinguir entre "ainda tentando" e "nunca vai funcionar" sem inspeção manual.

## Consequências

### Positivas

- **Cobertura de indisponibilidades reais** — a janela de 15 horas cobre manutenções planejadas, falhas de manhã inteira e problemas de DNS/rede transitórios.
- **Fila principal sempre limpa** — eventos com falha permanente saem da outbox para a DLQ, mantendo o worker eficiente e a tabela enxuta.
- **Visibilidade operacional** — a tabela `webhook_dead_letter` funciona como um log claro de falhas permanentes, com motivo e payload preservados para diagnóstico.
- **Recuperação controlada** — o endpoint de replay permite reprocessar eventos individuais após o cliente voltar, sem necessidade de scripts ad-hoc ou acesso direto ao banco.

### Negativas

- **Entrega não garantida em tempo real** — se o cliente ficou fora por 15 horas, só receberá o evento após replay manual. Não há notificação automática de que eventos caíram na DLQ.
- **Custo operacional do replay** — alguém precisa monitorar a DLQ e acionar o replay. Sem alertas configurados, eventos podem ficar esquecidos na dead letter.
- **Rigidez da progressão** — os intervalos são fixos (1m/5m/30m/2h/12h). Se um cliente específico precisar de uma política diferente, não há personalização por destinatário nesta versão.
- **Complexidade adicional** — uma tabela e um endpoint a mais para manter, testar e documentar em relação a uma solução mais simples de apenas marcar como `failed` na outbox.
