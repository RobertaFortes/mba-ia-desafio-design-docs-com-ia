# FDD — Sistema de Webhooks de Notificação de Pedidos

| | |
|---|---|
| **Status** | Rascunho para revisão (Larissa, Bruno, Diego) |
| **Data** | 2026-10-01 |
| **Documentos** | [PRD](PRD.md) · [RFC](RFC.md) · [Tracker](TRACKER.md) · [ADRs](adrs/) |

> **Convenção de origem.** Itens sem marcação vêm da reunião ou do código (ver [Tracker](TRACKER.md)). Itens marcados **[Proposta FDD]** são detalhes de implementação necessários que a reunião não definiu; devem ser confirmados na revisão e estão sinalizados também no RFC (seção 5, Q5–Q9). Nomes de colunas e variáveis são sugestões de implementação.
> Nomenclatura: a API existente usa camelCase (`customerId`, `totalCents`); o **payload entregue ao cliente** usa snake_case, como descrito na reunião ([09:43] Diego).

---

## 1. Contexto e motivação técnica

Clientes B2B consultam `GET /orders` em loop para detectar mudança de status. O sistema não possui eventos, filas ou webhooks. `OrderService.changeStatus` (`src/modules/orders/order.service.ts`) já executa numa única `prisma.$transaction` a validação da transição, o débito/reposição de estoque, o `order.update` e o `orderStatusHistory.create`. Qualquer chamada HTTP nessa transação acoplaria a disponibilidade do cliente à operação de pedidos ([ADR-001](adrs/ADR-001-outbox-no-mysql.md)).

## 2. Objetivos técnicos

| # | Objetivo | Meta |
|---|---|---|
| OT-1 | Registrar o evento atomicamente com a mudança de status | 0 casos de status alterado sem evento (quando há webhook assinante) |
| OT-2 | Latência da mudança de status até o primeiro envio | < 10 s; polling de 2 s |
| OT-3 | Isolar o cliente lento/offline da API de pedidos | Timeout de 10 s por chamada; worker em processo separado |
| OT-4 | Autenticidade e integridade do payload | HMAC-SHA256, secret por endpoint, HTTPS obrigatório |
| OT-5 | Não perder eventos por indisponibilidade do cliente | 5 tentativas em ≈15 h; depois DLQ reprocessável |
| OT-6 | Aderir às convenções do repositório | Módulo `webhooks` em `src/modules`, `AppError`, Pino, `WEBHOOK_*` |

## 3. Escopo e exclusões

**Dentro:** tabelas de configuração, outbox, entregas e DLQ; `publishWebhookEvent`; worker (a nova entry-point do worker); CRUD de webhooks; rotação de secret; consulta de entregas; replay de DLQ (ADMIN).

**Fora (explicitamente):** e-mail ao cliente em caso de falha ([09:37] Larissa); rate limiting de saída ([09:39] Larissa/Diego); dashboard visual ([09:40] Larissa); arquivamento de eventos entregues ([09:08] Diego); múltiplos workers/ordenação global ([09:13] Larissa); webhooks **inbound** ([09:02] Marcos). Nenhuma alteração em `src/`, `prisma/` ou `tests/` faz parte **desta entrega documental**; este FDD descreve o que o time implementará.

---

## 4. Modelo de dados (Prisma, sugerido)

Todas as PKs são UUID `@db.Char(36)`, como `prisma/schema.prisma` ([09:51] Larissa). Nomes de tabela em snake_case via `@@map`.

| Tabela | Campos principais | Observações |
|---|---|---|
| `webhook_endpoints` | `id`, `customerId` (FK `customers`), `url`, `secret`, `previousSecret?`, `previousSecretExpiresAt?`, `events` (Json: lista de `OrderStatus`), `active`, `createdAt`, `updatedAt` | url + secret + customer + ativo ([09:21]); lista de status ([09:33]); campos de rotação por [09:21] |
| `webhook_outbox` | `id` (= `X-Event-Id`), `webhookId` (FK), `orderId`, `eventType`, `payload` (Json, snapshot), `status` (`PENDING`,`PROCESSING`,`FAILED`,`DELIVERED`), `attempts`, `nextAttemptAt`, `lastError?`, `createdAt`, `updatedAt`, `deliveredAt?` | Índices em `status` e `createdAt` ([09:08]); `nextAttemptAt` **[Proposta FDD]** para agendar o backoff |
| `webhook_deliveries` | `id`, `outboxId`, `webhookId`, `attempt`, `success`, `statusCode?`, `responseBody?` (truncado), `durationMs`, `error?`, `createdAt` | Alimenta `GET /webhooks/:id/deliveries` ([09:34]: sucesso/falha, payload, response, tempo). Tabela **[Proposta FDD]**: a reunião define o endpoint, não o armazenamento. O payload vem do join com a outbox |
| `webhook_dead_letter` | `id`, `outboxId`, `webhookId`, `payload`, `reason`, `failedAt`, `replayedAt?`, `replayedById?` | Payload, motivo e timestamp ([09:18]); campos de replay **[Proposta FDD]** para auditoria ([09:36]) |

`OrderStatus` reaproveita o enum existente. Mapeamento de status da outbox (`PENDING/PROCESSING/FAILED/DELIVERED`) corresponde a "pendente, processando, falhou, entregue" ([09:08]).

---

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro de `changeStatus`)

1. `OrderController.changeStatus` → `OrderService.changeStatus(id, input, userId)` abre `prisma.$transaction`.
2. Executa o fluxo atual: busca o pedido, valida `from !== to` e `canTransition(from, to)`, debita/repõe estoque, `tx.order.update`, `tx.orderStatusHistory.create`.
3. **Novo:** `await publishWebhookEvent(tx, order, from, to)` (função em novo arquivo `publisher` do módulo webhooks, **[Proposta FDD]** de arquivo; nome da função e assinatura vêm de [09:41] Bruno).
4. Dentro da função:
   1. `tx.webhookEndpoint.findMany({ where: { customerId: order.customerId, active: true } })`.
   2. Filtra os que contêm `to` em `events` (**filtro na inserção**, [09:34] Bruno). Se nenhum, retorna sem inserir.
   3. Para cada webhook: gera `eventId = uuid`, monta o snapshot (seção 6.1) com `timestamp` = momento da mudança, e `tx.webhookOutbox.create({ id: eventId, status: PENDING, attempts: 0, nextAttemptAt: now, payload })`.
5. Qualquer erro na função propaga e dá **rollback de toda a transação** ([09:40] Bruno). Ele é lançado como `WebhookOutboxPublishError` (`WEBHOOK_OUTBOX_PUBLISH_FAILED`, 500).
6. Commit. O resto de `changeStatus` (o `findUnique` final) permanece igual.

### 5.2 Processamento pelo worker

Entry-point do worker; lógica em novo arquivo `processor` do módulo webhooks ([09:28] Bruno).

```
loop a cada 2 s (setTimeout recursivo, sem sobreposição de ciclos):
  1. SELECT eventos WHERE status = PENDING AND nextAttemptAt <= now
     ORDER BY createdAt ASC LIMIT <batch>          # batch pequeno
  2. para cada evento, em sequência (single-worker, preserva ordem):
     a. UPDATE status = PROCESSING   (claim)
     b. carrega webhook; se inativo/inexistente → vai para DLQ (reason = WEBHOOK_NOT_FOUND)  [Proposta FDD]
     c. serializa payload; se > 64 KB → DLQ direto (WEBHOOK_PAYLOAD_TOO_LARGE), sem retry
     d. assina (6.2), monta headers (6.3), POST com timeout de 10 s
     e. grava webhook_deliveries (sucesso/falha, status, tempo, resposta)
     f. sucesso → status = DELIVERED, deliveredAt = now
        falha   → fluxo de retry (5.3)
```

- Tamanho do batch: variável de ambiente **[Proposta FDD]** `WEBHOOK_BATCH_SIZE` (padrão sugerido 10); a reunião só diz "batch pequeno" ([09:08] Diego).
- **Critério de sucesso [Proposta FDD, Q9]:** resposta HTTP 2xx. Qualquer outro status, erro de rede ou timeout é falha.
- **Recuperação de crash [Proposta FDD]:** no início do worker e a cada ciclo, eventos `PROCESSING` com `updatedAt` mais antigo que 2 × timeout (20 s) voltam a `PENDING`. Isso pode gerar duplicata, coberta por at-least-once ([ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)).
- O worker instancia o próprio `PrismaClient` via `createPrismaClient()` de `src/config/database.ts` ([09:30] Bruno).
- **Shutdown:** em `SIGINT`/`SIGTERM` termina o evento corrente, para o loop e chama `prisma.$disconnect()` (mesmo padrão de `src/server.ts`).

### 5.3 Retry

Intervalos fixos ([09:17] Larissa): **1 min → 5 min → 30 min → 2 h → 12 h**.

| Falha nº (após 1ª tentativa) | Próxima tentativa em | Soma acumulada |
|---|---|---|
| 1ª falha | +1 min | 1 min |
| 2ª | +5 min | 6 min |
| 3ª | +30 min | 36 min |
| 4ª | +2 h | 2 h 36 min |
| 5ª | +12 h | ≈ 14 h 36 min |
| 6ª falha | — vai para DLQ | |

Em falha: `attempts += 1`, `lastError` preenchido, `status = PENDING`, `nextAttemptAt = now + intervalo[attempts - 1]`. Leitura adotada de "5 tentativas" **(Q5)**: 5 reenvios após a falha inicial, coerente com "≈15 h entre a primeira falha e a última tentativa" ([09:17] Diego). Se a revisão decidir que o total é 5 chamadas, remove-se o último intervalo.

Nota de ordem: um evento em retry não bloqueia os seguintes do mesmo pedido; portanto a ordenação por `order_id` ([09:12] Diego) vale **quando não há falhas**. Isto é consequência do desenho single-worker/polling e deve constar da documentação para clientes.

### 5.4 DLQ

Quando `attempts` excede o limite (ou erro não retentável como payload > 64 KB):

1. Em transação: `INSERT webhook_dead_letter { outboxId, webhookId, payload, reason, failedAt }` e `UPDATE webhook_outbox SET status = FAILED`.
2. Log `webhook_dlq_moved` (nível `warn`) com `eventId`, `webhookId`, `reason`.

### 5.5 Replay manual (ADMIN)

`POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego):

1. `authenticate` + `requireRole('ADMIN')`.
2. Busca o registro da DLQ (`WEBHOOK_DLQ_NOT_FOUND` se inexistente; `WEBHOOK_DLQ_ALREADY_REPLAYED` se já reprocessado **[Proposta FDD]**).
3. Em transação: outbox → `status = PENDING`, `attempts = 0`, `nextAttemptAt = now`; DLQ → `replayedAt`, `replayedById = req.user.id`.
4. Log `webhook_replay_requested` com `userId`, `dlqId`, `eventId` ([09:36] Sofia). O `X-Event-Id` **é preservado**, então o cliente deduplica normalmente.

---

## 6. Contratos públicos

Prefixo `/api/v1` (ver `src/routes/index.ts`). Todos exigem `Authorization: Bearer <JWT>` (`authenticate`). Erros no formato existente `{ "error": { "code", "message", "details?" } }`.

**Autorização.** CRUD, rotação e deliveries: qualquer role autenticada ([09:36]–[09:37] Sofia/Marcos). Replay: `ADMIN`. O `customerId` vem no **body ou na query**, não do JWT ([09:32] Larissa).

### 6.1 `POST /webhooks` — cadastrar webhook

Request:
```json
{
  "customerId": "6f1b7a52-3c1e-4f0a-9d7b-2a6c1d9e8f10",
  "url": "https://api.atlas.example.com/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"]
}
```
Response `201 Created`:
```json
{
  "id": "b2d0f7c4-8a31-4e55-a7b6-0e5c9f3d1a22",
  "customerId": "6f1b7a52-3c1e-4f0a-9d7b-2a6c1d9e8f10",
  "url": "https://api.atlas.example.com/hooks/oms",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_9f8c...e41",
  "createdAt": "2026-10-01T12:00:00.000Z"
}
```
- A `secret` é gerada pelo servidor e **só é devolvida aqui e na rotação** ([09:31] Marcos).
- Erros: `400 WEBHOOK_INVALID_URL` (não https), `400 WEBHOOK_INVALID_EVENTS`, `400 VALIDATION_ERROR`, `404 NOT_FOUND` (customer inexistente, via `NotFoundError('Customer')`), `401 UNAUTHORIZED`.

### 6.2 `GET /webhooks?customerId=...` — listar webhooks do customer

Response `200 OK` (formato `paginated()` de `src/shared/http/response.ts`; `secret` **nunca** é retornada):
```json
{
  "data": [
    {
      "id": "b2d0f7c4-8a31-4e55-a7b6-0e5c9f3d1a22",
      "customerId": "6f1b7a52-3c1e-4f0a-9d7b-2a6c1d9e8f10",
      "url": "https://api.atlas.example.com/hooks/oms",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-10-01T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```
Query: `customerId` (uuid, obrigatório), `page`, `pageSize`. Erros: `400 VALIDATION_ERROR`, `401`.

### 6.3 `PATCH /webhooks/:id` — editar

Request (todos opcionais):
```json
{ "url": "https://api.atlas.example.com/hooks/v2", "events": ["PAID", "SHIPPED"], "active": true }
```
Response `200 OK`: objeto do webhook como em 6.2.
Erros: `404 WEBHOOK_NOT_FOUND`, `400 WEBHOOK_INVALID_URL`, `400 WEBHOOK_INVALID_EVENTS`, `400 VALIDATION_ERROR`. Alterar `events` afeta apenas eventos **futuros**; linhas já na outbox não são refiltradas **[Proposta FDD]**.

### 6.4 `DELETE /webhooks/:id` — remover

Response `204 No Content` (mesmo padrão de `CustomerController.delete`). Erro: `404 WEBHOOK_NOT_FOUND`.
Comportamento para eventos pendentes do webhook removido: o worker os encaminha à DLQ com `reason = WEBHOOK_NOT_FOUND` **[Proposta FDD]**; implica FK sem cascade ou remoção lógica (decisão de implementação).

### 6.5 `POST /webhooks/:id/rotate-secret` — rotacionar secret

Rota **[Proposta FDD]** (a reunião exige "endpoint para pedir nova secret", sem path, [09:21] Sofia). Request sem body.
Response `200 OK`:
```json
{
  "id": "b2d0f7c4-8a31-4e55-a7b6-0e5c9f3d1a22",
  "secret": "whsec_41ab...c07",
  "previousSecretExpiresAt": "2026-10-02T12:00:00.000Z"
}
```
A secret anterior permanece válida por 24 h e depois é descartada ([09:21] Sofia). Erros: `404 WEBHOOK_NOT_FOUND`.

### 6.6 `GET /webhooks/:id/deliveries` — histórico de entregas

Query: `limit` (1–100, padrão 100) — "últimos 100" ([09:34] Marcos). Response `200 OK`:
```json
{
  "data": [
    {
      "eventId": "0a9c1e7e-52b4-4c8d-8f10-33d7a1f2b9aa",
      "attempt": 1,
      "success": true,
      "statusCode": 200,
      "durationMs": 184,
      "responseBody": "ok",
      "payload": { "event_id": "0a9c1e7e-52b4-4c8d-8f10-33d7a1f2b9aa", "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "createdAt": "2026-10-01T12:05:03.000Z"
    },
    {
      "eventId": "7d3e0b44-91c2-4a6f-b2de-6c8a5e1f0d13",
      "attempt": 2,
      "success": false,
      "durationMs": 10001,
      "error": "WEBHOOK_DELIVERY_TIMEOUT",
      "createdAt": "2026-10-01T12:07:10.000Z"
    }
  ]
}
```
Erros: `404 WEBHOOK_NOT_FOUND`, `400 VALIDATION_ERROR`, `401`.

### 6.7 `POST /admin/webhooks/dead-letter/:id/replay` — reprocessar DLQ (ADMIN)

Request sem body. Response `202 Accepted`:
```json
{
  "dlqId": "c41f9a10-7b2e-4d63-9a58-1f0e7c2b6d99",
  "eventId": "7d3e0b44-91c2-4a6f-b2de-6c8a5e1f0d13",
  "status": "PENDING",
  "replayedAt": "2026-10-01T13:00:00.000Z",
  "replayedBy": "a7c3d1e2-0b4f-4c9a-8e11-5d2f6b7a9c30"
}
```
Erros: `401`, `403 FORBIDDEN` (role insuficiente, de `requireRole`), `404 WEBHOOK_DLQ_NOT_FOUND`, `409 WEBHOOK_DLQ_ALREADY_REPLAYED`. Status 202 é escolha **[Proposta FDD]** (replay assíncrono).

### 6.8 Contrato de saída: o request que o cliente recebe

`POST <url do cliente>` com `Content-Type: application/json` e os headers ([09:44]–[09:45] Diego, Sofia):

| Header | Valor |
|---|---|
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (= `webhook_outbox.id`), estável entre retries e replay |
| `X-Signature` | HMAC-SHA256 hexadecimal do **corpo bruto** (formato do valor: ver 7.2) |
| `X-Timestamp` | Momento do envio, ISO 8601, para detecção de replay pelo cliente |
| `X-Webhook-Id` | `id` do `webhook_endpoints` que originou o envio |

Corpo (snapshot imutável gerado na inserção, sem `items`, [09:43] Diego):
```json
{
  "event_id": "7d3e0b44-91c2-4a6f-b2de-6c8a5e1f0d13",
  "event_type": "order.status_changed",
  "timestamp": "2026-10-01T12:05:00.000Z",
  "order_id": "e19b6c3a-42d1-4b7e-a0f5-8c2d9a1b3e47",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "6f1b7a52-3c1e-4f0a-9d7b-2a6c1d9e8f10",
  "total_cents": 1599000
}
```
O cliente consulta `GET /orders/:id` para detalhes. Resposta esperada do cliente: 2xx (Q9). Tamanho máximo do corpo: 64 KB ([09:24] Larissa).

---

## 7. Segurança

### 7.1 Validação de URL
Schema Zod em `webhook.schemas.ts`: `z.string().url()` com refinamento `startsWith('https://')`; falha mapeia para `WebhookInvalidUrlError` (`400 WEBHOOK_INVALID_URL`) ([09:23] Sofia).

### 7.2 Assinatura
`signature = HMAC_SHA256(secret, rawBody)` em hex, calculada com `node:crypto` sobre exatamente os bytes enviados ([09:20] Sofia). Geração de secret com `crypto.randomBytes(32)` **[Proposta FDD]**; a revisão da Sofia cobre geração e uso ([09:46]).

**Durante a rotação (Q7) [Proposta FDD]:** enquanto `previousSecretExpiresAt > now`, `X-Signature` carrega duas assinaturas separadas por vírgula (`v1=<nova>,v1=<antiga>`), e o cliente aceita qualquer uma. Fora da janela, uma só. Alternativa a avaliar com Sofia: assinar só com a nova e deixar que a antiga valha apenas se o cliente ainda não migrou.

### 7.3 Armazenamento da secret (Q6)
O HMAC exige a secret recuperável; a proteção em repouso (ex.: criptografia de coluna) **não foi discutida** e depende da revisão de segurança. O logger Pino hoje **não** redige campos chamados `secret` (`redactPaths` em `src/shared/logger/index.ts` cobre `password`, `passwordHash`, `token`, `accessToken`): é necessário acrescentar `*.secret` e `*.previousSecret` — relevante pois cliente já vazou secret em log ([09:22] Diego).

---

## 8. Matriz de erros

Todos estendem `AppError` e usam o prefixo `WEBHOOK_` ([09:28]–[09:29] Bruno/Larissa). Linhas "Entrega" não têm HTTP de API: são razões gravadas em `lastError`, `webhook_deliveries.error` e `webhook_dead_letter.reason`.

| Código | HTTP | Classe base (existente) | Quando | Retentável |
|---|---|---|---|---|
| `WEBHOOK_NOT_FOUND` | 404 | `AppError` (404) | Webhook inexistente em `PATCH/DELETE/rotate/deliveries` | n/a |
| `WEBHOOK_INVALID_URL` | 400 | `BadRequestError` | URL não https ou malformada | n/a |
| `WEBHOOK_INVALID_EVENTS` | 400 | `BadRequestError` | Lista de status vazia ou com valor fora de `OrderStatus` | n/a |
| `WEBHOOK_SECRET_REQUIRED` | 500 | `AppError` | Endpoint sem secret disponível no momento de assinar (integridade) | Não |
| `WEBHOOK_DLQ_NOT_FOUND` | 404 | `AppError` (404) | Replay de id inexistente na DLQ | n/a |
| `WEBHOOK_DLQ_ALREADY_REPLAYED` | 409 | `ConflictError` | Replay repetido do mesmo registro (**Proposta FDD**) | n/a |
| `WEBHOOK_OUTBOX_PUBLISH_FAILED` | 500 | `AppError` | Falha ao inserir na outbox dentro de `changeStatus`; causa rollback ([09:40]) | n/a |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | — Entrega | `AppError` | Corpo > 64 KB; vai direto à DLQ ([09:23]–[09:24]) | Não |
| `WEBHOOK_DELIVERY_TIMEOUT` | — Entrega | `AppError` | Sem resposta em 10 s ([09:42]) | Sim |
| `WEBHOOK_DELIVERY_FAILED` | — Entrega | `AppError` | Resposta não 2xx ou erro de rede | Sim |
| `WEBHOOK_RETRIES_EXHAUSTED` | — Entrega | `AppError` | Limite de tentativas atingido; motivo gravado na DLQ | Não |

Erros genéricos reaproveitados sem alteração: `VALIDATION_ERROR` (Zod via `validate`), `UNAUTHORIZED`/`FORBIDDEN` (`authenticate`/`requireRole`), `NOT_FOUND` (customer inexistente). Nenhuma mudança em `errorMiddleware`.

---

## 9. Estratégias de resiliência

| Aspecto | Estratégia | Origem |
|---|---|---|
| Timeout | 10 s por chamada, `fetch` nativo do Node ≥20 (`engines.node` do `package.json`) com `AbortSignal.timeout(10_000)`; sem nova dependência | [09:42] Diego |
| Retry | 5 reenvios, 1m/5m/30m/2h/12h | [09:17] Larissa |
| Backoff | Exponencial por tabela fixa de intervalos | [09:15] Diego |
| Fallback | DLQ em `webhook_dead_letter` + replay manual ADMIN | [09:18] Diego |
| Isolamento | Processo separado; falha do cliente nunca atinge a transação de pedidos | [09:11] Diego |
| Atomicidade | Evento na mesma transação do status | [09:40] Bruno |
| Idempotência | At-least-once + `X-Event-Id` | [09:26] Larissa |
| Crash do worker | Reposição de `PROCESSING` expirado a `PENDING` | **Proposta FDD** |
| Shutdown limpo | Finaliza evento corrente e desconecta o Prisma | padrão de `src/server.ts` |

Limitações conhecidas: single-worker sem HA; ordenação apenas por `order_id` e sem falhas (5.3); nenhum rate limiting de saída; retenção da outbox/DLQ indefinida; **não há endpoint de listagem da DLQ** na reunião — o `id` para replay precisa vir de log ou consulta ao banco (candidato a evolução).

---

## 10. Observabilidade

**Logs (Pino, `src/shared/logger/index.ts`, [09:29] Bruno).** Eventos estruturados, no estilo dos existentes (`server_started`, `http_request`):

| Evento | Nível | Campos |
|---|---|---|
| `worker_started` / `worker_stopped` | info | `pollIntervalMs`, `batchSize` |
| `webhook_event_enqueued` | info | `eventId`, `webhookId`, `orderId`, `toStatus` |
| `webhook_delivery_succeeded` | info | `eventId`, `webhookId`, `attempt`, `statusCode`, `durationMs` |
| `webhook_delivery_failed` | warn | `eventId`, `webhookId`, `attempt`, `error`, `nextAttemptAt` |
| `webhook_dlq_moved` | warn | `eventId`, `webhookId`, `reason` |
| `webhook_replay_requested` | info | `userId`, `dlqId`, `eventId` (auditoria, [09:36] Sofia) |

Nunca logar `secret`, `previousSecret` nem o valor de `X-Signature` completo (ver 7.3).

**Métricas [Proposta FDD].** A reunião não escolheu stack de métricas e o projeto não tem uma; para respeitar [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md), as métricas abaixo podem ser derivadas por consulta SQL às tabelas ou emitidas como logs periódicos do worker, até que a plataforma defina um backend:

| Métrica | Tipo | Finalidade |
|---|---|---|
| `webhook_outbox_pending` | gauge | Backlog (linhas `PENDING`) |
| `webhook_outbox_oldest_pending_age_seconds` | gauge | Acompanhar o limite de 10 s ([09:02] Marcos) |
| `webhook_delivery_attempts_total{result}` | counter | Sucesso / falha / timeout |
| `webhook_delivery_duration_ms` | histogram | Latência do cliente (já gravada em `webhook_deliveries.durationMs`) |
| `webhook_dlq_total` | counter | Eventos enviados à DLQ |

**Tracing [Proposta FDD].** O projeto usa `req.id` / `X-Request-Id` (`src/middlewares/request-logger.middleware.ts`). Propõe-se: (a) usar `eventId` como chave de correlação em todos os logs do worker e no header `X-Event-Id`; (b) opcionalmente persistir o `requestId` da requisição que originou a mudança na linha da outbox para ligar API → worker. Tracing distribuído completo (OpenTelemetry) não foi discutido e está fora do escopo.

---

## 11. Integração com o sistema existente

| Caminho | Como o módulo de webhooks se integra |
|---|---|
| `src/modules/orders/order.service.ts` | `changeStatus` passa a chamar `publishWebhookEvent(tx, order, from, to)` logo após `tx.orderStatusHistory.create` e antes do `findUnique` final, dentro da mesma `$transaction` ([ADR-007](adrs/ADR-007-publicacao-transacional-em-change-status.md)). A assinatura pública de `changeStatus` e o construtor de `OrderService` não mudam. `create()` **não** emite evento (Q8). |
| `src/modules/orders/order.status.ts` | Reuso do conjunto de status válidos (`OrderStatus` do Prisma) para validar `events`; nenhuma regra de transição é alterada. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | As novas classes `Webhook*Error` herdam de `AppError`/`BadRequestError`/`ConflictError` com `errorCode` `WEBHOOK_*`, como `InsufficientStockError` e `InvalidStatusTransitionError`. Exportadas por `src/shared/errors/index.ts` ou por arquivo próprio do módulo. |
| `src/middlewares/error.middleware.ts` | **Sem alteração**: já serializa `AppError`, `ZodError` e erros Prisma `P2002`/`P2025` ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | `authenticate` em todas as rotas; `requireRole('ADMIN')` (já usado em `src/modules/users/user.routes.ts`) no replay. |
| `src/middlewares/validate.middleware.ts` | Validação de body/params/query com schemas Zod do módulo. |
| `src/app.ts` | `buildControllers` instancia `WebhookRepository`, `WebhookService`, `WebhookController` e os devolve em `Controllers`. |
| `src/routes/index.ts` | `buildApiRouter` monta `/webhooks` e `/admin/webhooks` (novo `webhook.routes.ts`), e `Controllers` ganha `webhooks`. |
| `src/server.ts` | Modelo para a entry-point do worker (bootstrap, `logger.info`, `SIGINT`/`SIGTERM`, `prisma.$disconnect()`); **não é modificado** — o worker é outro processo. |
| `src/config/database.ts` | `createPrismaClient()` reutilizado pelo worker para instanciar seu próprio client ([09:30] Bruno). |
| `src/config/env.ts` | Novas variáveis (ex.: `WEBHOOK_BATCH_SIZE`) entram no `envSchema` Zod. O worker valida o mesmo `env`. |
| `src/shared/logger/index.ts` | Reuso do `logger`; acrescentar `*.secret` ao `redactPaths` (7.3). |
| `src/shared/http/response.ts` | `paginated()` na listagem de webhooks. |
| `prisma/schema.prisma` | Novos models com `@@map`, UUID `Char(36)` e relações com `Customer`/`Order`; nova migration em `prisma/migrations/`. |
| `package.json` | Novo script `"worker"` (ex.: `tsx watch --env-file=.env <entry-point do worker>`, espelhando `dev`) e correspondente de produção. |
| `tests/setup.ts` | O `beforeEach` limpa tabelas em ordem de FK; as novas tabelas dependentes de `customers`/`orders` precisarão entrar nessa limpeza ao implementar. |

---

## 12. Dependências e compatibilidade

- **Runtime:** Node ≥ 20 (`engines` do `package.json`); `fetch`, `AbortSignal.timeout` e `node:crypto` nativos. Nenhuma biblioteca nova é necessária.
- **Banco:** MySQL via Prisma 5.22; apenas migrations aditivas (novas tabelas). Nenhuma coluna das tabelas existentes é alterada.
- **Compatibilidade da API:** rotas novas; contratos de `/orders` inalterados. Clientes sem webhook cadastrado não geram linhas na outbox (comportamento anterior preservado).
- **Operação:** exige um segundo processo (`npm run worker`) com a mesma `DATABASE_URL`.
- **Pessoas:** revisão de segurança da Sofia (2 dias úteis) antes do deploy ([09:46]); Marcos documenta o contrato no portal de desenvolvedor ([09:26], [09:40]).

## 13. Critérios de aceite técnicos

1. Mudança de status com webhook assinante cria exatamente uma linha de outbox por webhook, na mesma transação; sem assinante, nenhuma linha.
2. Forçar erro na inserção da outbox faz rollback do status, do histórico e do estoque.
3. Evento `PENDING` é entregue em < 10 s em condições normais (polling 2 s).
4. O request entregue contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json`; a assinatura confere com HMAC-SHA256 do corpo e da secret.
5. Resposta não 2xx ou ausência de resposta em 10 s agenda retry conforme 1m/5m/30m/2h/12h; esgotado, o evento vai para a DLQ.
6. Replay por não-ADMIN retorna 403; por ADMIN recoloca o evento como `PENDING` mantendo o `X-Event-Id` e registrando o usuário em log.
7. Cadastro com URL `http://` retorna `400 WEBHOOK_INVALID_URL`.
8. Após rotação, as duas secrets validam por 24 h; depois só a nova.
9. Payload > 64 KB não é enviado e termina na DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`.
10. Reiniciar a API não interrompe o worker; `secret` não aparece em nenhum log.
11. Testes existentes (`tests/auth.test.ts`, `tests/orders.test.ts`) continuam passando.

## 14. Riscos e mitigação

| Risco | Prob. | Impacto | Mitigação |
|---|---|---|---|
| Falha na outbox bloqueia mudança de status | Baixa | Alto | Operação simples (INSERT na mesma conexão); testes de rollback; monitorar erro `WEBHOOK_OUTBOX_PUBLISH_FAILED` |
| Backlog/latência do worker único > 10 s | Média | Médio | Batch ajustável, métrica de idade do evento mais antigo, escalar com particionamento (decisão futura) |
| Vazamento de secret (logs, banco) | Média | Alto | Secret por endpoint, rotação 24 h, redação de logs, revisão da Sofia |
| Duplicatas mal tratadas pelo cliente | Média | Médio | `X-Event-Id` + documentação destacada no portal |
| Ordem trocada após retry | Média | Médio | Documentar limitação; avaliar bloqueio por `order_id` se virar problema |
| Crescimento da outbox/DLQ sem retenção | Média | Médio | Arquivamento planejado como próxima feature ([09:08]) |
| Evento stuck em `PROCESSING` após crash | Baixa | Médio | Recuperação por timeout (5.2) |
