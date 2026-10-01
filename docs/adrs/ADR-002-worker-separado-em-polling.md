# ADR-002: Worker em processo separado com polling de 2 segundos

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisões em [09:10] e [09:11])
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md)

## Contexto

Com a outbox em MySQL (ADR-001), é preciso um consumidor que leia os eventos pendentes e faça as chamadas HTTP. O requisito de produto é que "abaixo de 10 segundos" já é tempo real para os clientes ([09:02] Marcos). O projeto hoje tem uma única entry-point, `src/server.ts`, que sobe o Express e cria o `prisma` de `src/config/database.ts`.

## Decisão

- O worker faz **polling a cada 2 segundos**, buscando os eventos pendentes mais antigos em **batch pequeno** ([09:08] e [09:09] Diego). Latência mínima de 2s no pior caso, aceita ([09:10] Larissa).
- O worker roda como **processo separado da API**, para que reiniciar a API não perca o worker ([09:11] Diego).
- A nova entry-point do worker com script `npm run worker` ([09:11] Larissa); a lógica de processamento fica em um arquivo `worker` no módulo webhooks ou `webhook.processor.ts` ([09:28] Bruno).
- O worker usa o **mesmo banco** (mesma `DATABASE_URL`) mas instancia **seu próprio `PrismaClient`**, pois o client é por processo ([09:30] Bruno).
- **Single-worker** por enquanto. A ordem de entrega segue `created_at` da outbox, o que dá ordenação implícita por `order_id`; **não há garantia de ordering global** ([09:12] Diego, [09:13] Larissa).

## Alternativas Consideradas

1. **Trigger/notificação do banco (estilo LISTEN/NOTIFY).** Descartada: MySQL não tem listener nativo; trigger só executa SQL e avisar o processo externo exigiria gambiarras ([09:09] Diego).
2. **Worker dentro do processo da API.** Descartada: reinício da API perderia o worker ([09:11] Diego).
3. **Múltiplos workers em paralelo desde já.** Adiado: quebraria a ordenação; exigiria particionar por `order_id` ou lock pessimista, "problema do futuro" ([09:13] Diego).

## Consequências

**Positivas**
- Simplicidade operacional: um loop, sem broker nem listener.
- Isolamento de falhas: API e worker reiniciam de forma independente.
- Reaproveita stack (TypeScript, Prisma, Pino) e o padrão de bootstrap/shutdown de `src/server.ts`.

**Negativas / trade-offs**
- Latência mínima de até 2s e carga constante de consultas no MySQL, mesmo sem eventos.
- Single-worker é ponto único de throughput e de falha (sem HA nesta fase). Escalar exige novo ADR.
- Ordenação só vale por `order_id` e enquanto houver um único worker (limitação conhecida, [09:13] Larissa).
- Um deploy adicional (segundo processo) para operar.
