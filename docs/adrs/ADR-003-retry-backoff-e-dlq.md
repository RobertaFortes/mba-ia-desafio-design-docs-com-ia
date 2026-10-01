# ADR-003: Retry com backoff exponencial e DLQ em tabela separada

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisões em [09:17], [09:18] e [09:36])
- **Decisores:** Larissa, Diego, Bruno, Sofia
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-005](ADR-005-at-least-once-com-x-event-id.md)

## Contexto

O endpoint do cliente pode estar indisponível ou lento. A mudança de status do pedido não pode ser desfeita por isso ([09:04] Bruno). Já houve cliente com duas horas de indisponibilidade em manutenção planejada ([09:16] Diego). O worker trata como falha qualquer chamada que não responda em 10 segundos ([09:42] Diego).

## Decisão

- **Backoff exponencial com 5 tentativas**, nos intervalos **1 min, 5 min, 30 min, 2 h, 12 h** (≈15 h entre a primeira falha e a última tentativa) ([09:17] Larissa).
- Esgotadas as tentativas, a falha é permanente e o evento vai para a **DLQ persistida na tabela separada `webhook_dead_letter`**, com payload, motivo da falha e timestamp ([09:18] Diego).
- **Reprocessamento manual** via `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
- O replay exige **role `ADMIN`** (reaproveitando `requireRole`) e **registra em log quem executou**, para auditoria ([09:36] Sofia, Larissa).

## Alternativas Consideradas

1. **3 tentativas.** Descartada: cobre pouco; uma indisponibilidade matinal ou manutenção de 2h esgotaria tudo em ~30 min ([09:16] Diego).
2. **Retry indefinido com backoff.** Descartada: evento fica pendurado para sempre se o cliente sumiu ([09:15] Diego).
3. **Marcar como `falhou` na própria outbox em vez de tabela separada.** Descartada: tabela separada deixa a leitura da outbox mais limpa e serve de evidência para debug e reprocessamento ([09:18] Diego).
4. **Notificar o cliente por e-mail quando falhar repetidamente.** Fora de escopo desta fase ([09:37] Larissa).

## Consequências

**Positivas**
- Cobre indisponibilidades longas sem acumular retry infinito.
- A outbox principal fica enxuta; falhas permanentes têm histórico próprio e reprocessável.
- Replay protegido por papel e auditado.

**Negativas / trade-offs**
- Cliente pode demorar até ~15 h para receber um evento após falha prolongada; aceito pelo PM ([09:17] Marcos).
- Replay é manual: não há recuperação automática da DLQ nem aviso proativo ao cliente (e-mail adiado).
- Mais uma tabela para modelar, migrar e eventualmente expurgar (política de retenção não definida, ver RFC).
- A interpretação exata de "5 tentativas" (5 reenvios após a falha inicial, como a soma dos intervalos indica) está registrada como questão em aberto no RFC.
