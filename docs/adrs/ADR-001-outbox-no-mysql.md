# ADR-001: Padrão Outbox no MySQL com snapshot do payload

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:08] Larissa)
- **Decisores:** Larissa, Diego, Bruno
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-007](ADR-007-publicacao-transacional-em-change-status.md)

## Contexto

Três clientes B2B (Atlas Comercial, MaxDistribuição, Nova Cargo) querem ser notificados quando o status de um pedido muda, hoje fazem polling em `GET /orders` ([09:00] Marcos). A mudança de status acontece em `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), dentro de um `prisma.$transaction` que já atualiza `orders`, insere em `order_status_history` e movimenta `products.stockQuantity`. Disparar HTTP dentro dessa transação travaria mudanças de status de outros pedidos quando um cliente fosse lento, e um cliente fora do ar não pode causar rollback ([09:04] Bruno). O time é pequeno e a única infraestrutura de dados do projeto é o MySQL (`docker-compose.yml`, `prisma/schema.prisma`).

## Decisão

Adotar o **padrão Outbox no MySQL existente**: na mesma transação que atualiza o pedido, inserir uma linha na tabela `webhook_outbox` por webhook interessado. Um worker (ver ADR-002) lê a tabela e faz as chamadas HTTP.

Detalhes fechados na reunião:

- Status da linha: `pendente`, `processando`, `falhou`, `entregue`; índices em status e `created_at` ([09:08] Diego).
- Chave primária **UUID**, como o restante do schema ([09:51] Larissa).
- O payload é guardado **já renderizado (snapshot)** no momento da inserção, não apenas o `order_id` ([09:52] Larissa, Diego, Bruno).
- O filtro de eventos por webhook acontece **na inserção**: se nenhum webhook do customer quer aquele status, nenhuma linha é criada ([09:34] Bruno).
- Arquivamento de linhas entregues (~30 dias) fica **fora desta feature** ([09:08] Diego).

## Alternativas Consideradas

1. **Chamada HTTP síncrona dentro de `changeStatus`.** Descartada: acopla a latência/disponibilidade do cliente à transação de estoque e histórico; rollback por cliente offline é inaceitável ([09:04] Bruno, [09:06] Diego).
2. **Redis Streams (ou fila similar).** Descartada: exige subir nova infra num time pequeno; "overengineering" ([09:07] Larissa, Diego).
3. **Guardar só `order_id` e renderizar o payload no envio.** Descartada: o evento deixaria de refletir o estado do momento da mudança se o pedido mudasse depois ([09:52] Larissa).
4. **Trigger de banco para notificar o worker.** Descartada: trigger só executa SQL e não notifica processo externo ([09:09] Diego). Ver ADR-002.

## Consequências

**Positivas**
- Atomicidade: se a transação commitou, o evento existe; se deu rollback, o evento some junto ([09:06] Diego).
- Nenhuma infra nova; reaproveita MySQL, Prisma e migrations existentes.
- Payload imutável facilita debug e replay.

**Negativas / trade-offs**
- A tabela cresce e disputa o mesmo MySQL das operações de pedido; mitigado por índices e, depois, por arquivamento (fora de escopo).
- Snapshot ocupa mais espaço que guardar apenas uma referência.
- Escrita adicional dentro da transação de `changeStatus` aumenta levemente sua duração.
- Latência de entrega depende de polling (ADR-002), não é push.
