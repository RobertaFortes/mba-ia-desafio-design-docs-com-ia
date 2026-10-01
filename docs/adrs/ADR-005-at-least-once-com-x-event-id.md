# ADR-005: Garantia at-least-once com `X-Event-Id` para deduplicação no cliente

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:26] Larissa)
- **Decisores:** Larissa, Diego, Marcos
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-003](ADR-003-retry-backoff-e-dlq.md)

## Contexto

Com retries (ADR-003) e um worker que pode falhar entre "enviar" e "marcar como entregue", o mesmo evento pode ser enviado mais de uma vez. É preciso definir a semântica de entrega e como o cliente distingue duplicatas.

## Decisão

- Garantir **entrega at-least-once**; o cliente pode receber o mesmo evento duas vezes e deve estar preparado ([09:24] Diego).
- Cada evento recebe um **UUID gerado quando entra na outbox**, enviado no header `X-Event-Id`. O cliente deduplica por esse valor ([09:25] Diego).
- A responsabilidade de deduplicação é do cliente; Marcos documentará isso em destaque no portal de desenvolvedor ([09:26] Marcos).
- Conjunto de headers de cada envio: `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:44] Diego, [09:44] Sofia).

## Alternativas Consideradas

1. **Exactly-once.** Descartada: exigiria coordenação dos dois lados e complexidade muito maior ([09:25] Diego).
2. **At-most-once (sem retry).** Não discutida explicitamente; incompatível com o objetivo de não perder notificações e com a política de retry do ADR-003. Registrada como alternativa plausível.

## Consequências

**Positivas**
- Modelo simples, igual ao de Stripe e GitHub, citados na reunião ([09:25] Diego).
- Nenhum evento é perdido enquanto a transação principal tiver commitado.

**Negativas / trade-offs**
- Joga responsabilidade de idempotência para o cliente ([09:25] Sofia); depende de boa documentação.
- Duplicatas são esperadas, não exceção: testes e docs do portal precisam deixar isso explícito.
- Sem garantia de ordenação global; vale só por `order_id` com single-worker (ADR-002).
