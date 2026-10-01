# ADR-007: Publicação do evento dentro da transação de `changeStatus` via `publishWebhookEvent(tx, ...)`

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:41] Bruno, Diego)
- **Decisores:** Bruno, Diego, Larissa
- **Relacionados:** [ADR-001](ADR-001-outbox-no-mysql.md), [ADR-006](ADR-006-reuso-dos-padroes-existentes.md)

## Contexto

`OrderService.changeStatus` (`src/modules/orders/order.service.ts`) executa, em `this.prisma.$transaction(async (tx) => ...)`, a validação da transição (`canTransition` em `src/modules/orders/order.status.ts`), o débito/reposição de estoque, o `tx.order.update` e o `tx.orderStatusHistory.create`. A garantia do outbox só existe se o evento for gravado **nessa mesma transação** ([09:40] Bruno, [09:41] Diego).

## Decisão

- Inserir na `webhook_outbox` **dentro da transação** de `changeStatus`. Se a inserção falhar, a transação inteira dá rollback: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).
- Expor uma função **`publishWebhookEvent(tx, order, fromStatus, toStatus)`** que recebe o `tx` da transação corrente ([09:41] Bruno). É uma função, **não um repositório injetado** no `OrderService` ([09:41] Diego).
- A função consulta os webhooks ativos do customer que assinam o `toStatus`, renderiza o snapshot do payload e insere uma linha por webhook; se nenhum assina, não insere nada ([09:33]/[09:34] Marcos, Bruno).
- O `OrderService` chama a função depois do `orderStatusHistory.create` e antes do `findUnique` final.

## Alternativas Consideradas

1. **Publicar após o commit (fora da transação).** Descartada: perde a garantia do outbox; status pode mudar sem evento ([09:41] Diego, "se ficar fora da transação, perde a garantia toda").
2. **Injetar um `WebhookRepository` no construtor de `OrderService`.** Descartada em favor de função pura que recebe `tx` ([09:41] Diego).
3. **Hook/trigger SQL em `orders`.** Não discutida; plausível, mas esconderia a regra de filtro e snapshot fora do código TypeScript. Registrada como alternativa plausível.

## Consequências

**Positivas**
- Consistência atômica entre status, histórico, estoque e evento.
- Mudança mínima e localizada no `OrderService`; o contrato público de `changeStatus` não muda.

**Negativas / trade-offs**
- `changeStatus` passa a depender do módulo `webhooks` (acoplamento de código, ainda que por função).
- Falha na outbox bloqueia a mudança de status (comportamento desejado, mas é um novo modo de falha para o módulo de pedidos).
- A transação fica um pouco mais longa (consulta de webhooks + inserções).
- A criação de pedido (`OrderService.create`) também grava histórico com `fromStatus: null`; a reunião só tratou `changeStatus`, então emitir evento na criação permanece em aberto (ver RFC).
