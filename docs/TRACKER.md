# Tracker de Rastreabilidade

Mapeia cada item dos documentos à origem na `TRANSCRICAO.md` (formato `[hh:mm] Falante`) ou no código (caminho de arquivo). Itens marcados **Proposta** são detalhes de implementação que a reunião não definiu: apontam para a fala ou o arquivo que os motivou e estão sinalizados como proposta nos documentos.

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
|---|---|---|---|---|---|
| PRD-CTX-01 | docs/PRD.md | Contexto | Três clientes B2B (Atlas, MaxDistribuição, Nova Cargo) pedem notificação de mudança de status | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Contexto | Clientes fazem polling em GET /orders, integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Feature é somente outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | Aplicação não possui mecanismo de notificação, eventos ou filas | CODIGO | src/app.ts |
| PRD-CTX-05 | docs/PRD.md | Restrição | Risco de a Atlas migrar para concorrente se não entregar no trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-06 | docs/PRD.md | Restrição | Prazo desejado pela Atlas: fim de novembro | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-01 | docs/PRD.md | Objetivo | Latência < 10 s ("tempo real" para o cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo | Cobrir indisponibilidade de até ≈15 h via 5 tentativas | TRANSCRICAO | [09:17] Larissa |
| PRD-OBJ-03 | docs/PRD.md | Objetivo | Zero mudanças de status sem evento | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-04 | docs/PRD.md | Objetivo | Entregar aos 3 clientes em 3 sprints | TRANSCRICAO | [09:47] Larissa |
| PRD-OBJ-05 | docs/PRD.md | Objetivo | Reduzir polling a GET /orders (meta numérica não definida) | TRANSCRICAO | [09:00] Marcos |
| PRD-PER-01 | docs/PRD.md | Contexto | Usuários da nossa API representam o cliente; JWT do sistema | TRANSCRICAO | [09:32] Marcos |
| PRD-PER-02 | docs/PRD.md | Contexto | Marcos documenta integração no portal de desenvolvedor | TRANSCRICAO | [09:40] Marcos |
| PRD-OOS-01 | docs/PRD.md | Fora de escopo | E-mail ao cliente por falhas, próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-OOS-02 | docs/PRD.md | Fora de escopo | Rate limiting de saída: observar e decidir depois | TRANSCRICAO | [09:39] Larissa |
| PRD-OOS-03 | docs/PRD.md | Fora de escopo | Dashboard visual fora de escopo | TRANSCRICAO | [09:40] Larissa |
| PRD-OOS-04 | docs/PRD.md | Fora de escopo | Arquivamento de eventos entregues (~30 dias) fora da feature | TRANSCRICAO | [09:08] Diego |
| PRD-OOS-05 | docs/PRD.md | Fora de escopo | Múltiplos workers / ordenação global adiados | TRANSCRICAO | [09:13] Larissa |
| PRD-OOS-06 | docs/PRD.md | Fora de escopo | Exactly-once descartado | TRANSCRICAO | [09:25] Diego |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | Cadastrar webhook (POST) com secret gerada e devolvida | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | Lista de status de interesse por webhook | TRANSCRICAO | [09:33] Marcos |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | Editar webhook (PATCH) | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | Remover webhook (DELETE) | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Listar webhooks do customer; customer_id no body/path, não no JWT | TRANSCRICAO | [09:32] Larissa |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Notificação assíncrona sem bloquear mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Evento registrado atomicamente com a mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Payload enxuto com campos definidos, sem items | TRANSCRICAO | [09:43] Diego |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Assinatura HMAC e headers X-Signature/X-Event-Id/X-Timestamp/X-Webhook-Id | TRANSCRICAO | [09:44] Diego |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Rotação de secret com antiga válida por 24 h | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Retry 5x com backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | DLQ em tabela separada com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | Replay via endpoint admin, role ADMIN, com log de quem executou | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-14 | docs/PRD.md | Requisito Funcional | Histórico das últimas 100 entregas via GET /webhooks/:id/deliveries | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-15 | docs/PRD.md | Requisito Funcional | Identificador único de evento para dedupe no cliente | TRANSCRICAO | [09:25] Diego |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Polling de 2 s atende < 10 s | TRANSCRICAO | [09:09] Diego |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Entrega at-least-once | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | URL do webhook obrigatoriamente https (Zod) | TRANSCRICAO | [09:23] Sofia |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | Payload máx. 64 KB, erro se ultrapassar | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Timeout de 10 s por chamada | TRANSCRICAO | [09:42] Diego |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-07 | docs/PRD.md | Restrição | Ordenação só por order_id com single-worker | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Reuso de padrões do projeto (AppError, Pino, módulos, WEBHOOK_) | TRANSCRICAO | [09:30] Larissa |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | CRUD por qualquer role autenticada; replay só ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-RNF-10 | docs/PRD.md | Requisito Não Funcional | Revisão de segurança de 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Segundo processo (worker) na operação | TRANSCRICAO | [09:11] Larissa |
| PRD-DEP-02 | docs/PRD.md | Dependência | requireRole e authenticate existentes | CODIGO | src/middlewares/auth.middleware.ts |
| PRD-RISK-01 | docs/PRD.md | Risco | Perder prazo e a Atlas migrar | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Falha na outbox bloqueia mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RISK-03 | docs/PRD.md | Risco | Vazamento de secret pelo cliente | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Cliente não deduplica eventos | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-05 | docs/PRD.md | Risco | Worker único como limite de vazão/falha | TRANSCRICAO | [09:12] Diego |
| PRD-RISK-06 | docs/PRD.md | Risco | Eventos fora de ordem fora do cenário single-worker sem falhas | TRANSCRICAO | [09:13] Larissa |
| PRD-RISK-07 | docs/PRD.md | Risco | Revisão de segurança atrasar o deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-01 | docs/PRD.md | Estratégia | Testes seguem o stack Vitest existente | CODIGO | vitest.config.ts |
| PRD-TEST-02 | docs/PRD.md | Estratégia | Testes de integração seguem tests/orders.test.ts e factories | CODIGO | tests/orders.test.ts |
| RFC-PROP-01 | docs/RFC.md | Decisão | Outbox atômica com snapshot do payload | TRANSCRICAO | [09:06] Diego |
| RFC-PROP-02 | docs/RFC.md | Decisão | Worker separado (nova entry-point), `npm run worker` | TRANSCRICAO | [09:11] Larissa |
| RFC-PROP-03 | docs/RFC.md | Decisão | Resumo das decisões antes do fechamento da reunião | TRANSCRICAO | [09:48] Larissa |
| RFC-CTX-01 | docs/RFC.md | Contexto | Transação de changeStatus atualiza order, histórico e estoque | CODIGO | src/modules/orders/order.service.ts |
| RFC-ALT-01 | docs/RFC.md | Alternativa | Síncrono em changeStatus, descartado | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa | Redis Streams, descartado por infra extra | TRANSCRICAO | [09:07] Larissa |
| RFC-ALT-03 | docs/RFC.md | Alternativa | Trigger de banco para acordar o worker, descartado | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa | Retry indefinido / só 3 tentativas, descartados | TRANSCRICAO | [09:16] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa | Exactly-once, descartado | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-06 | docs/RFC.md | Alternativa | Secret global, descartada | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída | TRANSCRICAO | [09:39] Larissa |
| RFC-OQ-02 | docs/RFC.md | Questão em aberto | Ordenação com múltiplos workers | TRANSCRICAO | [09:13] Diego |
| RFC-OQ-03 | docs/RFC.md | Questão em aberto | Arquivamento de eventos entregues | TRANSCRICAO | [09:08] Diego |
| RFC-OQ-04 | docs/RFC.md | Questão em aberto | E-mail ao cliente em falhas, próxima fase | TRANSCRICAO | [09:37] Larissa |
| RFC-OQ-05 | docs/RFC.md | Lacuna (Proposta) | Contagem de "5 tentativas" vs soma dos intervalos | TRANSCRICAO | [09:17] Diego |
| RFC-OQ-06 | docs/RFC.md | Lacuna (Proposta) | Proteção da secret em repouso | TRANSCRICAO | [09:46] Sofia |
| RFC-OQ-07 | docs/RFC.md | Lacuna (Proposta) | Assinatura durante o grace period de 24 h | TRANSCRICAO | [09:21] Sofia |
| RFC-OQ-08 | docs/RFC.md | Lacuna (Proposta) | Evento na criação do pedido (create grava histórico com fromStatus null) | CODIGO | src/modules/orders/order.service.ts |
| RFC-OQ-09 | docs/RFC.md | Lacuna (Proposta) | Critério HTTP de sucesso não definido | TRANSCRICAO | [09:42] Diego |
| RFC-IMP-01 | docs/RFC.md | Trade-off | Prazo de 3 sprints + 2 dias de revisão de segurança | TRANSCRICAO | [09:47] Larissa |
| ADR-001 | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Padrão Outbox no MySQL | TRANSCRICAO | [09:08] Larissa |
| ADR-001-a | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | PK UUID na outbox | TRANSCRICAO | [09:51] Larissa |
| ADR-001-b | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Snapshot do payload na inserção | TRANSCRICAO | [09:52] Larissa |
| ADR-001-c | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Status e índices da outbox | TRANSCRICAO | [09:08] Diego |
| ADR-001-d | docs/adrs/ADR-001-outbox-no-mysql.md | Decisão | Filtro de eventos na inserção | TRANSCRICAO | [09:34] Bruno |
| ADR-001-e | docs/adrs/ADR-001-outbox-no-mysql.md | Contexto | Schema com UUID Char(36) e MySQL | CODIGO | prisma/schema.prisma |
| ADR-002 | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Worker em polling de 2 s | TRANSCRICAO | [09:10] Larissa |
| ADR-002-a | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Processo separado da API | TRANSCRICAO | [09:11] Diego |
| ADR-002-b | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | Nova entry-point do worker + npm run worker | TRANSCRICAO | [09:11] Larissa |
| ADR-002-c | docs/adrs/ADR-002-worker-separado-em-polling.md | Decisão | PrismaClient próprio no worker | TRANSCRICAO | [09:30] Bruno |
| ADR-002-d | docs/adrs/ADR-002-worker-separado-em-polling.md | Restrição | Single-worker, ordem por order_id | TRANSCRICAO | [09:12] Diego |
| ADR-002-e | docs/adrs/ADR-002-worker-separado-em-polling.md | Contexto | Entry-point atual src/server.ts como modelo | CODIGO | src/server.ts |
| ADR-003 | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| ADR-003-a | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | DLQ em tabela webhook_dead_letter | TRANSCRICAO | [09:18] Diego |
| ADR-003-b | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay manual via endpoint admin | TRANSCRICAO | [09:18] Diego |
| ADR-003-c | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Decisão | Replay exige ADMIN e loga o executor | TRANSCRICAO | [09:36] Larissa |
| ADR-003-d | docs/adrs/ADR-003-retry-backoff-e-dlq.md | Trade-off | 3 tentativas insuficientes (manutenção de 2 h) | TRANSCRICAO | [09:16] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | HMAC-SHA256 do corpo, secret por endpoint, rotação 24 h | TRANSCRICAO | [09:22] Sofia |
| ADR-004-a | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Decisão | Tabela de config: url, secret, customer_id, ativo | TRANSCRICAO | [09:21] Bruno |
| ADR-004-b | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Restrição | URL https validada no schema Zod | TRANSCRICAO | [09:23] Sofia |
| ADR-004-c | docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md | Trade-off | X-Timestamp para detecção de replay, fora da assinatura | TRANSCRICAO | [09:44] Diego |
| ADR-005 | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id | TRANSCRICAO | [09:26] Larissa |
| ADR-005-a | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | UUID gerado ao entrar na outbox | TRANSCRICAO | [09:25] Diego |
| ADR-005-b | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Trade-off | Responsabilidade de dedupe vai para o cliente | TRANSCRICAO | [09:25] Sofia |
| ADR-005-c | docs/adrs/ADR-005-at-least-once-com-x-event-id.md | Decisão | Headers do envio incluindo X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Reuso máximo dos padrões existentes | TRANSCRICAO | [09:30] Larissa |
| ADR-006-a | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Módulo webhooks em src/modules com controller/service/repository/routes/schemas | TRANSCRICAO | [09:27] Bruno |
| ADR-006-b | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Decisão | Erros WEBHOOK_* sobre AppError | TRANSCRICAO | [09:28] Bruno |
| ADR-006-c | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Hierarquia AppError | CODIGO | src/shared/errors/app-error.ts |
| ADR-006-d | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Error middleware trata AppError, Zod, Prisma | CODIGO | src/middlewares/error.middleware.ts |
| ADR-006-e | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Registro de módulos via buildControllers | CODIGO | src/app.ts |
| ADR-006-f | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | requireRole reaproveitado | CODIGO | src/middlewares/auth.middleware.ts |
| ADR-006-g | docs/adrs/ADR-006-reuso-dos-padroes-existentes.md | Contexto | Logger Pino compartilhado | CODIGO | src/shared/logger/index.ts |
| ADR-007 | docs/adrs/ADR-007-publicacao-transacional-em-change-status.md | Decisão | Inserir na outbox dentro da transação de changeStatus | TRANSCRICAO | [09:40] Bruno |
| ADR-007-a | docs/adrs/ADR-007-publicacao-transacional-em-change-status.md | Decisão | Função publishWebhookEvent(tx, order, from, to) | TRANSCRICAO | [09:41] Bruno |
| ADR-007-b | docs/adrs/ADR-007-publicacao-transacional-em-change-status.md | Decisão | Função pura, sem injetar repository | TRANSCRICAO | [09:41] Diego |
| ADR-007-c | docs/adrs/ADR-007-publicacao-transacional-em-change-status.md | Contexto | Transação atual de changeStatus | CODIGO | src/modules/orders/order.service.ts |
| ADR-007-d | docs/adrs/ADR-007-publicacao-transacional-em-change-status.md | Contexto | canTransition define transições válidas | CODIGO | src/modules/orders/order.status.ts |
| FDD-OT-01 | docs/FDD.md | Objetivo | Latência < 10 s | TRANSCRICAO | [09:02] Marcos |
| FDD-OT-02 | docs/FDD.md | Objetivo | Isolar cliente lento da API de pedidos | TRANSCRICAO | [09:04] Bruno |
| FDD-DADOS-01 | docs/FDD.md | Decisão | Tabelas com PK UUID | TRANSCRICAO | [09:51] Larissa |
| FDD-DADOS-02 | docs/FDD.md | Decisão | Tabela de configuração do webhook | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-03 | docs/FDD.md | Decisão | Outbox com status e índices em status/created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-04 | docs/FDD.md | Proposta | Tabela webhook_deliveries para alimentar o histórico | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-05 | docs/FDD.md | Decisão | webhook_dead_letter com payload, motivo, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-06 | docs/FDD.md | Proposta | Campo nextAttemptAt para agendar backoff | TRANSCRICAO | [09:17] Diego |
| FDD-DADOS-07 | docs/FDD.md | Contexto | Convenção de models Prisma (@@map, Char(36)) | CODIGO | prisma/schema.prisma |
| FDD-FLX-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação | TRANSCRICAO | [09:06] Diego |
| FDD-FLX-02 | docs/FDD.md | Fluxo | Filtro por status do webhook antes de inserir | TRANSCRICAO | [09:34] Bruno |
| FDD-FLX-03 | docs/FDD.md | Fluxo | Rollback total se a outbox falhar | TRANSCRICAO | [09:40] Bruno |
| FDD-FLX-04 | docs/FDD.md | Fluxo | Worker faz polling dos pendentes mais antigos em batch pequeno | TRANSCRICAO | [09:09] Diego |
| FDD-FLX-05 | docs/FDD.md | Proposta | WEBHOOK_BATCH_SIZE padrão 10 | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-06 | docs/FDD.md | Proposta | Sucesso = HTTP 2xx | TRANSCRICAO | [09:42] Diego |
| FDD-FLX-07 | docs/FDD.md | Proposta | Recuperação de eventos presos em PROCESSING | TRANSCRICAO | [09:08] Diego |
| FDD-FLX-08 | docs/FDD.md | Fluxo | Worker com PrismaClient próprio via createPrismaClient | CODIGO | src/config/database.ts |
| FDD-FLX-09 | docs/FDD.md | Fluxo | Shutdown com SIGINT/SIGTERM e $disconnect | CODIGO | src/server.ts |
| FDD-FLX-10 | docs/FDD.md | Fluxo | Tabela de retry 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Diego |
| FDD-FLX-11 | docs/FDD.md | Restrição | Retry quebra ordenação por order_id | TRANSCRICAO | [09:12] Diego |
| FDD-FLX-12 | docs/FDD.md | Fluxo | Movimentação para DLQ ao esgotar tentativas | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-13 | docs/FDD.md | Fluxo | Replay recoloca na outbox como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLX-14 | docs/FDD.md | Fluxo | Log de quem fez o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks (url, events, secret na resposta) | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks por customer, paginado | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-02a | docs/FDD.md | Contrato | Formato paginated() | CODIGO | src/shared/http/response.ts |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id (204, como CustomerController.delete) | CODIGO | src/modules/customers/customer.controller.ts |
| FDD-CONTRATO-05 | docs/FDD.md | Proposta | POST /webhooks/:id/rotate-secret (path não definido) | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries, últimos 100 | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers de saída X-Event-Id, X-Signature, X-Timestamp, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Header X-Webhook-Id | TRANSCRICAO | [09:44] Sofia |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Corpo do payload snake_case sem items | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-11 | docs/FDD.md | Contexto | Formato de erro { error: { code, message, details } } | CODIGO | src/middlewares/error.middleware.ts |
| FDD-CONTRATO-12 | docs/FDD.md | Contrato | Prefixo /api/v1 | CODIGO | src/routes/index.ts |
| FDD-CONTRATO-13 | docs/FDD.md | Proposta | Replay responde 202 | TRANSCRICAO | [09:18] Diego |
| FDD-SEG-01 | docs/FDD.md | Restrição | URL https via Zod | TRANSCRICAO | [09:23] Sofia |
| FDD-SEG-02 | docs/FDD.md | Decisão | HMAC-SHA256 sobre o corpo | TRANSCRICAO | [09:20] Sofia |
| FDD-SEG-03 | docs/FDD.md | Proposta | Duas assinaturas em X-Signature durante a rotação | TRANSCRICAO | [09:21] Sofia |
| FDD-SEG-04 | docs/FDD.md | Restrição | Logger não redige campos secret hoje; incluir *.secret | CODIGO | src/shared/logger/index.ts |
| FDD-SEG-05 | docs/FDD.md | Contexto | Cliente já vazou secret em log | TRANSCRICAO | [09:22] Diego |
| FDD-SEG-06 | docs/FDD.md | Restrição | Armazenamento seguro da secret fica para a revisão de segurança | TRANSCRICAO | [09:46] Sofia |
| FDD-ERR-01 | docs/FDD.md | Erro | WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED | TRANSCRICAO | [09:28] Bruno |
| FDD-ERR-02 | docs/FDD.md | Erro | Prefixo WEBHOOK_ para tudo do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERR-03 | docs/FDD.md | Erro | WEBHOOK_PAYLOAD_TOO_LARGE (>64 KB) | TRANSCRICAO | [09:24] Larissa |
| FDD-ERR-04 | docs/FDD.md | Erro | WEBHOOK_DELIVERY_TIMEOUT (10 s) | TRANSCRICAO | [09:42] Diego |
| FDD-ERR-05 | docs/FDD.md | Erro | WEBHOOK_OUTBOX_PUBLISH_FAILED (rollback) | TRANSCRICAO | [09:40] Bruno |
| FDD-ERR-06 | docs/FDD.md | Erro | Classes base reutilizadas (BadRequestError, ConflictError etc.) | CODIGO | src/shared/errors/http-errors.ts |
| FDD-ERR-07 | docs/FDD.md | Proposta | WEBHOOK_DLQ_NOT_FOUND / ALREADY_REPLAYED / DELIVERY_FAILED / RETRIES_EXHAUSTED | TRANSCRICAO | [09:18] Diego |
| FDD-RES-01 | docs/FDD.md | Decisão | Timeout de 10 s | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Decisão | fetch nativo sem nova dependência (Node ≥ 20) | CODIGO | package.json |
| FDD-RES-03 | docs/FDD.md | Decisão | Fallback para DLQ | TRANSCRICAO | [09:18] Diego |
| FDD-RES-04 | docs/FDD.md | Decisão | Processo separado isola a API | TRANSCRICAO | [09:11] Diego |
| FDD-RES-05 | docs/FDD.md | Restrição | Sem endpoint de listagem da DLQ na reunião | TRANSCRICAO | [09:35] Diego |
| FDD-OBS-01 | docs/FDD.md | Decisão | Logs com Pino, nada novo | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Contexto | Padrão de eventos de log existente (server_started, http_request) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-03 | docs/FDD.md | Proposta | Métricas derivadas de SQL/logs (sem stack de métricas definida) | TRANSCRICAO | [09:02] Marcos |
| FDD-OBS-04 | docs/FDD.md | Proposta | Correlação por eventId e requestId (req.id) | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus chama publishWebhookEvent na mesma transação | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Status válidos reutilizados de order.status.ts | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Novas classes de erro sobre AppError | CODIGO | src/shared/errors/app-error.ts |
| FDD-INT-04 | docs/FDD.md | Integração | errorMiddleware sem alteração | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate e requireRole('ADMIN') | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate() com schemas Zod | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | buildControllers registra o módulo | CODIGO | src/app.ts |
| FDD-INT-08 | docs/FDD.md | Integração | buildApiRouter monta /webhooks e /admin/webhooks | CODIGO | src/routes/index.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Entry-point do worker espelha server.ts | CODIGO | src/server.ts |
| FDD-INT-10 | docs/FDD.md | Integração | env.ts recebe novas variáveis | CODIGO | src/config/env.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Novos models e migration | CODIGO | prisma/schema.prisma |
| FDD-INT-12 | docs/FDD.md | Integração | Script "worker" no package.json | CODIGO | package.json |
| FDD-INT-13 | docs/FDD.md | Integração | tests/setup.ts limpa tabelas em ordem de FK | CODIGO | tests/setup.ts |
| FDD-INT-14 | docs/FDD.md | Integração | create() grava histórico com fromStatus null; não emite evento | CODIGO | src/modules/orders/order.service.ts |
| FDD-COMP-01 | docs/FDD.md | Dependência | Revisão da Sofia antes do deploy | TRANSCRICAO | [09:46] Sofia |
| FDD-COMP-02 | docs/FDD.md | Dependência | Marcos documenta contrato no portal | TRANSCRICAO | [09:26] Marcos |
| FDD-RISK-01 | docs/FDD.md | Risco | Backlog/latência do worker único | TRANSCRICAO | [09:13] Diego |
| FDD-RISK-02 | docs/FDD.md | Risco | Crescimento da outbox sem retenção | TRANSCRICAO | [09:08] Diego |
