# RFC-001: Sistema de Webhooks de Notificação de Pedidos

| Campo | Valor |
|---|---|
| **Autor** | Roberta Fortes (rascunho assistido por IA a partir da reunião técnica) |
| **Status** | Em revisão |
| **Data** | 2026-10-01 |
| **Revisores** | Larissa (Tech Lead), Marcos (PM), Bruno (Eng. Pleno, Pedidos), Diego (Eng. Sênior, Plataforma), Sofia (Segurança) |
| **Documentos relacionados** | [PRD](PRD.md) · [FDD](FDD.md) · [Tracker](TRACKER.md) · ADRs listados na seção 7 |

## 1. Resumo executivo (TL;DR)

Propomos notificar clientes B2B, via HTTP outbound, sempre que o status de um pedido mudar. A mudança grava um evento numa **tabela outbox no MySQL**, na **mesma transação** de `OrderService.changeStatus`. Um **worker em processo separado**, com **polling de 2 s**, entrega o evento assinado com **HMAC-SHA256** (secret por endpoint), com **retry exponencial (5 tentativas, 1m→12h)** e **DLQ** em tabela própria. A garantia é **at-least-once**, com `X-Event-Id` para deduplicação no cliente. O módulo segue os padrões existentes em `src/modules/*`. Prazo estimado: 3 sprints, incluindo a revisão de segurança da Sofia.

## 2. Contexto e problema

Atlas Comercial, MaxDistribuição e Nova Cargo consultam `GET /orders` periodicamente para saber se o status mudou, o que torna a integração lenta e cara; a Atlas condiciona a permanência à entrega até o fim do trimestre. Para eles, "tempo real" é qualquer coisa abaixo de 10 segundos. O sistema atual não tem eventos, filas nem webhooks. A mudança de status ocorre numa transação que já atualiza o pedido, grava o histórico e movimenta o estoque, então chamadas HTTP síncronas ali são inviáveis (cliente lento travaria outras mudanças; cliente offline não pode gerar rollback). O fluxo é apenas **outbound**: o cliente recebe, não envia.

## 3. Proposta técnica

Visão geral (o detalhe de implementação está no [FDD](FDD.md)):

```
changeStatus (tx) ──► webhook_outbox ──(poll 2s)──► Worker ──HTTPS+HMAC──► endpoint do cliente
                                                      │ falha
                                                      ├─ retry 1m/5m/30m/2h/12h
                                                      └─ esgotou ─► webhook_dead_letter ─► replay (ADMIN)
```

1. **Outbox atômica.** O evento é inserido, já com o payload renderizado (snapshot), dentro da transação que muda o status. Só são criadas linhas para webhooks que assinam aquele status. → [ADR-001](adrs/ADR-001-outbox-no-mysql.md), [ADR-007](adrs/ADR-007-publicacao-transacional-em-change-status.md)
2. **Worker separado.** Novo entry-point `src/worker.ts` *(arquivo novo, a criar)* (`npm run worker`), com `PrismaClient` próprio, single-worker. → [ADR-002](adrs/ADR-002-worker-separado-em-polling.md)
3. **Resiliência.** Timeout de 10 s por chamada; 5 tentativas com backoff exponencial; esgotado, vai para a DLQ, reprocessável manualmente por um ADMIN. → [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md)
4. **Segurança.** HTTPS obrigatório, HMAC-SHA256 do corpo, secret única por endpoint, rotação com 24 h de convivência. → [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
5. **Semântica de entrega.** At-least-once; o cliente deduplica por `X-Event-Id`. → [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md)
6. **API.** CRUD de configuração de webhooks e consulta de entregas para qualquer usuário autenticado; replay de DLQ exige role `ADMIN`. Novo módulo `src/modules/webhooks`, reaproveitando `AppError`, Pino e `errorMiddleware`, com códigos `WEBHOOK_*`. → [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md)

## 4. Alternativas consideradas

| # | Alternativa | Por que foi descartada (trade-off) | Origem |
|---|---|---|---|
| A1 | **Webhook síncrono dentro de `changeStatus`** | Simples, mas acopla a latência e a disponibilidade do cliente à transação de estoque/histórico; cliente offline forçaria rollback ou perda de evento. | [09:04] Bruno, [09:06] Diego |
| A2 | **Redis Streams / fila dedicada** | Entrega mais "reativa" e desacoplada, mas exige subir e operar infra nova num time pequeno: overengineering. | [09:07] Larissa, Diego |
| A3 | **Trigger de banco para acordar o worker** | Evitaria polling, porém MySQL não tem LISTEN/NOTIFY e trigger só executa SQL, sem avisar processo externo. Polling de 2 s já atende o limite de 10 s. | [09:09] Diego |
| A4 | **Retry indefinido** ou **só 3 tentativas** | Indefinido deixa evento pendurado para sempre; 3 tentativas (~30 min) não cobre manutenção de 2 h. | [09:15]–[09:16] Diego |
| A5 | **Exactly-once** | Exigiria coordenação dos dois lados; at-least-once com `X-Event-Id` cobre a maior parte dos casos e é o padrão de mercado. | [09:25] Diego |
| A6 | **Secret global da plataforma** | Operacionalmente mais simples, mas um vazamento expõe todos os clientes. | [09:21] Sofia |

## 5. Questões em aberto

**Levantadas e adiadas na reunião**

| # | Questão | Situação |
|---|---|---|
| Q1 | **Rate limiting de saída por cliente** (ex.: 50 pedidos mudando em 1 min geram 50 chamadas). | "Observar e decidir depois" ([09:39] Larissa). |
| Q2 | **Ordenação com múltiplos workers.** Hoje só há ordem por `order_id` com um único worker. | Solução futura possível: particionar por `order_id` ou lock pessimista ([09:13] Diego); documentado como limitação conhecida. |
| Q3 | **Arquivamento de eventos entregues** (~30 dias). | Fora desta feature ([09:08] Diego). |
| Q4 | **Aviso ao cliente por e-mail** quando o webhook falha repetidamente. | Próxima fase, após medir impacto ([09:37] Larissa). |

**Lacunas identificadas ao consolidar a documentação (não discutidas na reunião; precisam de decisão da revisão)**

| # | Questão | Observação |
|---|---|---|
| Q5 | Como contar "5 tentativas": a soma 1m+5m+30m+2h+12h ≈ 15 h ([09:17] Diego) indica 5 **reenvios** após a falha inicial (6 chamadas no total). Confirmar. | A FDD adota essa leitura provisoriamente. |
| Q6 | Como proteger a secret em repouso, dado que o HMAC exige o valor recuperável. | Sofia revisará geração e uso da secret ([09:46]). |
| Q7 | Durante os 24 h de rotação, com qual(is) secret(s) assinar o `X-Signature`? | A FDD propõe uma opção para ser validada. |
| Q8 | `OrderService.create` também grava histórico (`fromStatus: null`); emitir evento na criação do pedido? | A reunião tratou só `changeStatus`; a FDD assume que **não** emite. |
| Q9 | Critério de sucesso da entrega (quais status HTTP contam como sucesso). | Não mencionado; a FDD propõe 2xx. |

## 6. Impacto e riscos

- **Código existente:** alteração localizada em `src/modules/orders/order.service.ts` (`changeStatus`), registro do módulo em `src/app.ts` e `src/routes/index.ts`, novas tabelas no `prisma/schema.prisma` e nova entry-point `src/worker.ts`. O contrato HTTP de pedidos não muda.
- **Desempenho:** a transação de `changeStatus` fica ligeiramente mais longa (consulta + inserções na outbox); o polling adiciona leitura constante ao MySQL.
- **Riscos principais:** falha na outbox bloqueia mudança de status (intencional); worker único é ponto único de falha e de vazão; duplicatas exigem que o cliente implemente dedupe; vazamento de secret (mitigado por secret por endpoint e rotação). Detalhes e mitigações no [PRD](PRD.md#10-riscos-e-mitigação) e no [FDD](FDD.md).
- **Prazo:** 3 sprints, mais 2 dias úteis de revisão de segurança antes do deploy; meta do cliente: fim de novembro ([09:45] Marcos, [09:46] Larissa, Sofia).

## 7. Decisões relacionadas

| ADR | Tema |
|---|---|
| [ADR-001](adrs/ADR-001-outbox-no-mysql.md) | Outbox no MySQL com snapshot do payload |
| [ADR-002](adrs/ADR-002-worker-separado-em-polling.md) | Worker em processo separado, polling de 2 s |
| [ADR-003](adrs/ADR-003-retry-backoff-e-dlq.md) | Retry com backoff e DLQ |
| [ADR-004](adrs/ADR-004-hmac-sha256-secret-por-endpoint.md) | HMAC-SHA256, secret por endpoint, rotação 24 h |
| [ADR-005](adrs/ADR-005-at-least-once-com-x-event-id.md) | At-least-once com `X-Event-Id` |
| [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes.md) | Reuso dos padrões do projeto |
| [ADR-007](adrs/ADR-007-publicacao-transacional-em-change-status.md) | Publicação dentro da transação de `changeStatus` |
