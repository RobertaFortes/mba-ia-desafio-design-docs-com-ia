# PRD — Sistema de Webhooks de Notificação de Pedidos

| | |
|---|---|
| **Status** | Em revisão |
| **Data** | 2026-10-01 |
| **Stakeholders** | Marcos (PM), Larissa (Tech Lead), Bruno, Diego, Sofia |
| **Documentos** | [RFC](RFC.md) · [FDD](FDD.md) · [ADRs](adrs/) · [Tracker](TRACKER.md) |

## 1. Resumo e contexto da feature

O Order Management System (OMS) passará a notificar clientes B2B, por HTTP e em tempo quase real, sempre que o status de um pedido mudar. Hoje o OMS (Node.js + TypeScript + MySQL/Prisma) não tem nenhum mecanismo de notificação externa: os clientes descobrem mudanças consultando `GET /orders` repetidamente. A feature é **somente outbound**: o OMS envia, o cliente recebe ([09:02] Marcos).

## 2. Problema e motivação

Três clientes B2B (Atlas Comercial, MaxDistribuição e Nova Cargo) pediram formalmente para serem notificados quando o status dos seus pedidos muda. O polling atual torna a integração lenta e cara para eles. A Atlas indicou que pode migrar para um concorrente se a entrega não ocorrer até o fim do trimestre; a data desejada é fim de novembro ([09:00] Marcos, [09:45] Marcos). Para os clientes, "tempo real" significa **menos de 10 segundos** ([09:02] Marcos).

## 3. Público-alvo e cenários de uso

| Persona | Necessidade |
|---|---|
| **Cliente B2B (integrador)** — Atlas, MaxDistribuição, Nova Cargo | Receber eventos assinados, validar origem, deduplicar, consultar histórico de entregas |
| **Usuário que representa o cliente na nossa API** | Cadastrar, editar, remover e listar webhooks via API autenticada com JWT do sistema ([09:32] Marcos) |
| **Administrador (role ADMIN)** | Reprocessar eventos que foram para a DLQ, com auditoria |
| **Equipe de produto/suporte** | Documentar a integração no portal de desenvolvedor ([09:26], [09:40] Marcos) |

Cenários:
1. A Atlas cadastra `https://.../hooks`, escolhe receber só `SHIPPED` e `DELIVERED` e guarda a secret devolvida na criação.
2. Um pedido passa a `SHIPPED`; em até 10 s a Atlas recebe o evento assinado e confere o HMAC.
3. O endpoint da Nova Cargo fica fora do ar por 2 h de manutenção; o OMS retenta e entrega depois, sem perda.
4. Após ~15 h sem sucesso, o evento vai para a DLQ e um ADMIN o reprocessa.
5. A MaxDistribuição rotaciona a secret e migra seus sistemas dentro de 24 h.

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
|---|---|---|---|
| O1 | Notificar em tempo real | Tempo entre o commit da mudança de status e o primeiro envio ao endpoint | **< 10 s** ([09:02] Marcos); polling de 2 s no pior caso ([09:10] Larissa) |
| O2 | Não perder notificações por indisponibilidade do cliente | Eventos entregues após falha transitória dentro da janela de retry | Cobertura de indisponibilidades de até ≈15 h (5 tentativas) ([09:17] Larissa) |
| O3 | Consistência entre status e evento | Mudanças de status com assinante sem evento correspondente | **0** ([09:40] Bruno) |
| O4 | Atender os clientes solicitantes no prazo | Clientes com webhook em produção | Os 3 solicitantes, até o fim de novembro, em 3 sprints ([09:45]–[09:47]) |
| O5 | Reduzir o polling dos clientes a `GET /orders` | Volume de chamadas de polling dos 3 clientes | Redução esperada; **linha de base e meta numérica a definir com Marcos** (não discutidas na reunião) |

## 5. Escopo

### 5.1 Incluído
- Configuração de webhooks por customer (CRUD) e filtro por status de interesse.
- Notificação de mudança de status de pedido via outbox e worker separado.
- Assinatura HMAC-SHA256, secret por endpoint e rotação com 24 h de convivência.
- Retry com backoff, DLQ e replay manual por ADMIN.
- Histórico de entregas por webhook.

### 5.2 Fora de escopo (descartado ou adiado na reunião)

| Item | Decisão | Origem |
|---|---|---|
| Aviso por e-mail ao cliente quando o webhook falha repetidamente | Fora da fase; talvez na próxima, após medir impacto | [09:37] Larissa |
| Rate limiting de envio por cliente | Não faz parte; observar e implementar se virar problema | [09:39] Larissa, Diego |
| Dashboard/painel visual para o cliente | Fora agora; projeto separado do time de frontend | [09:40] Larissa |
| Arquivamento de eventos entregues (~30 dias) | Fora desta feature | [09:08] Diego |
| Múltiplos workers e ordenação global | Adiado; só ordem por `order_id` com um worker | [09:13] Larissa |
| Exactly-once | Descartado; at-least-once com dedupe no cliente | [09:25] Diego |
| Webhooks inbound (cliente → OMS) | Não é necessidade | [09:02] Marcos |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
|---|---|---|
| RF-01 | Cadastrar webhook via `POST` com `url`; a secret é gerada pelo sistema e devolvida na criação | [09:31] Marcos |
| RF-02 | O webhook declara a lista de status que deseja receber; só esses geram evento | [09:31], [09:33] Marcos |
| RF-03 | Editar um webhook (`PATCH`) | [09:33] Bruno |
| RF-04 | Remover um webhook (`DELETE`) | [09:33] Bruno |
| RF-05 | Listar os webhooks de um customer (`GET`); o `customerId` vem no body/path, não do JWT | [09:33] Bruno, [09:32] Larissa |
| RF-06 | Enviar notificação a cada mudança de status de pedido que o webhook assina, de forma assíncrona e sem bloquear a mudança de status | [09:04] Bruno, [09:33] Marcos |
| RF-07 | O evento é registrado atomicamente com a mudança de status; falha ao registrar desfaz a mudança | [09:40] Bruno |
| RF-08 | Payload JSON enxuto (event_id, event_type `order.status_changed`, timestamp, order_id, order_number, from_status, to_status, customer_id, total_cents), sem itens | [09:43] Diego |
| RF-09 | Assinar o corpo com HMAC-SHA256 usando a secret do endpoint e enviar nos headers `X-Signature`, `X-Event-Id`, `X-Timestamp`, `X-Webhook-Id` | [09:20] Sofia, [09:44]–[09:45] |
| RF-10 | Rotacionar a secret pela API, mantendo a anterior válida por 24 h | [09:21] Sofia |
| RF-11 | Retentar falhas 5 vezes com backoff 1m/5m/30m/2h/12h | [09:17] Larissa |
| RF-12 | Após esgotar as tentativas, mover o evento para a DLQ (tabela separada) com payload, motivo e timestamp | [09:18] Diego |
| RF-13 | Reprocessar item da DLQ via `POST /admin/webhooks/dead-letter/:id/replay`, exigindo role ADMIN e registrando quem executou | [09:18] Diego, [09:36] Sofia |
| RF-14 | Consultar o histórico das últimas 100 entregas (sucesso/falha, payload, resposta, tempo) via `GET /webhooks/:id/deliveries` | [09:34] Marcos |
| RF-15 | Todo evento tem um identificador único (`X-Event-Id`) para deduplicação no cliente | [09:25] Diego |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
|---|---|---|
| RNF-01 | Latência de notificação < 10 s; polling de 2 s | [09:02] Marcos, [09:09] Diego |
| RNF-02 | Entrega at-least-once | [09:24] Diego |
| RNF-03 | URL do webhook obrigatoriamente `https` (validação Zod) | [09:23] Sofia |
| RNF-04 | Payload máximo de 64 KB; acima disso é erro, não truncamento | [09:23] Sofia, [09:24] Diego/Larissa |
| RNF-05 | Timeout de 10 s por chamada HTTP; estouro conta como falha | [09:42] Diego |
| RNF-06 | Worker em processo separado da API, mesmo banco | [09:11] Diego |
| RNF-07 | Ordenação por `order_id` apenas, com single-worker (limitação conhecida) | [09:13] Larissa |
| RNF-08 | Reuso dos padrões do projeto: módulo `webhooks` em `src/modules`, `AppError`, Pino, códigos `WEBHOOK_*`, IDs UUID | [09:30] Larissa, [09:51] Larissa |
| RNF-09 | CRUD acessível a qualquer role autenticada (a endurecer no futuro); replay só ADMIN | [09:36]–[09:37] Sofia |
| RNF-10 | Revisão de segurança de HMAC e geração de secret antes do deploy (2 dias úteis) | [09:46] Sofia |

## 8. Decisões e trade-offs principais

| Decisão | Trade-off aceito | ADR |
|---|---|---|
| Outbox no MySQL em vez de Redis/fila | Menos infra e atomicidade, ao custo de carga extra no MySQL | [ADR-001](adrs/ADR-001-outbox-no-mysql.md) |
| Worker separado com polling de 2 s | Simplicidade vs latência mínima de 2 s e ponto único de vazão | [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) |
| 5 tentativas + DLQ | Cobre ~15 h sem pendurar eventos; replay é manual | [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação 24 h | Segurança vs complexidade de gerir duas secrets | [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Simplicidade vs responsabilidade de dedupe no cliente | [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) |
| Reuso dos padrões do projeto | Rapidez e consistência vs herdar limitações atuais | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) |
| Evento gravado na transação de `changeStatus` | Consistência vs nova dependência no módulo de pedidos | [ADR-007](adrs/ADR-007-publicacao-transacional-em-change-status.md) |

## 9. Dependências

- Módulos existentes de pedidos (`OrderService.changeStatus`), clientes (`customers`) e autenticação (`authenticate`, `requireRole`).
- MySQL e Prisma já em uso; nenhuma infra nova.
- Time de Segurança (Sofia): agenda de revisão antes do deploy.
- Produto (Marcos): documentação no portal de desenvolvedor, destacando at-least-once e dedupe; confirmação de prazo com a Atlas.
- Operação: capacidade de subir um segundo processo (worker).

## 10. Riscos e mitigação

| # | Risco | Probabilidade | Impacto | Mitigação |
|---|---|---|---|---|
| R1 | Atlas migrar para o concorrente se o prazo (fim de novembro) for perdido | Média | Alto | Escopo enxuto, 3 sprints planejados, fora de escopo explícito |
| R2 | Falha ao gravar o evento bloquear mudança de status | Baixa | Alto | Inserção simples na mesma transação; testes de rollback; alerta em `WEBHOOK_OUTBOX_PUBLISH_FAILED` |
| R3 | Vazamento de secret do cliente | Média | Alto | Secret por endpoint, rotação com 24 h, redação em logs, revisão de segurança |
| R4 | Cliente não deduplicar e processar duas vezes | Média | Médio | `X-Event-Id` e documentação destacada no portal |
| R5 | Worker único atrasar entregas além de 10 s ou cair | Média | Médio | Batch pequeno, monitorar idade do evento mais antigo, processo reiniciável independente da API |
| R6 | Cliente receber eventos fora de ordem após retry | Média | Médio | Documentar limitação (ordem só por `order_id`, sem falhas) |
| R7 | Revisão de segurança atrasar o deploy | Baixa | Médio | 2 dias úteis reservados no fim do cronograma |

## 11. Critérios de aceitação

1. Um cliente cadastra um webhook https, recebe a secret uma única vez e escolhe os status de interesse.
2. Ao mudar o status de um pedido para um status assinado, o cliente recebe o evento em menos de 10 s, com os cinco headers e assinatura HMAC-SHA256 válida.
3. Status não assinados não geram evento.
4. Com o endpoint fora do ar, o sistema retenta em 1m/5m/30m/2h/12h e depois move o evento para a DLQ.
5. Um ADMIN reprocessa um item da DLQ; um usuário não ADMIN recebe 403; o replay fica logado com o usuário.
6. O cliente consulta as últimas 100 entregas de um webhook.
7. A rotação de secret mantém a antiga válida por 24 h.
8. URL `http`, payload > 64 KB e timeout > 10 s se comportam conforme especificado.
9. A mudança de status e seu histórico continuam funcionando se um cliente está lento ou offline.

## 12. Estratégia de testes e validação

- **Unitários (Vitest, já configurado em `vitest.config.ts`):** cálculo do HMAC, agenda de backoff, filtro de eventos, montagem do snapshot, validação Zod (https, lista de eventos).
- **Integração (padrão de `tests/orders.test.ts` e `tests/helpers/factories.ts`):** `changeStatus` insere na outbox na mesma transação; rollback quando a inserção falha; endpoints CRUD/deliveries/replay com autenticação e role.
- **Ponta a ponta:** worker contra um servidor HTTP local simulando sucesso, 5xx, timeout e resposta lenta; verificar retry, DLQ e replay.
- **Segurança:** revisão da Sofia (HMAC, geração e armazenamento de secret), verificação de ausência de secret em logs.
- **Regressão:** suíte existente (`tests/auth.test.ts`, `tests/orders.test.ts`) deve continuar verde.
- **Validação com clientes:** piloto com a Atlas antes de ampliar às demais (**sugestão**, não discutida na reunião).
