# Da Reunião ao Documento — Design Docs de Webhooks de Pedidos

Entrega do desafio *"Da Reunião ao Documento: Design Docs Gerados por IA"* (MBA Full Cycle). O enunciado original está no repositório base: <https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia>.

## Sobre o desafio

Um Order Management System (Node.js + TypeScript + MySQL/Prisma) vai ganhar um **Sistema de Webhooks de Notificação de Pedidos**. A decisão técnica foi tomada numa reunião de ~55 minutos entre tech lead, PM, dois engenheiros e segurança, mas só existe a transcrição ([TRANSCRICAO.md](TRANSCRICAO.md)). A tarefa foi transformar essa transcrição, cruzada com o código existente, em um pacote de design docs (PRD, RFC, FDD, ADRs e Tracker) acionável para o time começar a implementar.

A regra central é a rastreabilidade: nada pode ser inventado. Por isso o desafio também pede o descarte consciente: e-mail de falha, rate limiting, dashboard e arquivamento foram discutidos e **não** viram requisito. A entrega é puramente documental; `src/`, `prisma/` e `tests/` não foram alterados.

## Ferramentas de IA utilizadas

- **Claude Code (modelo Claude Sonnet 5.5)**: ferramenta principal. Leu o repositório e a transcrição, extraiu decisões/requisitos/descartes, redigiu os documentos em Markdown e rodou verificações de consistência por script.
- **Shell/Python (scripts de verificação)**: não é IA, mas foi parte do ciclo. Conferiu existência dos arquivos citados, contou linhas do Tracker e a proporção TRANSCRICAO/CODIGO.

## Workflow adotado

1. **Fork e setup.** Clone do repositório base (o fork público no GitHub é criado antes do push final).
2. **Contextualização.** Leitura integral da transcrição e dos arquivos que a reunião toca: `order.service.ts`, `order.status.ts`, `shared/errors/*`, `error.middleware.ts`, `auth.middleware.ts`, `app.ts`, `routes/index.ts`, `server.ts`, `env.ts`, logger, `schema.prisma`, `tests/setup.ts`.
3. **Triagem da transcrição** antes de escrever: lista separada de (a) decisões fechadas, (b) requisitos funcionais, (c) restrições/NFRs, (d) ganchos com o código, (e) descartado/adiado, (f) lacunas não discutidas.
4. **ADRs primeiro** (7 arquivos), como esqueleto das decisões.
5. **RFC** em cima dos ADRs, com alternativas e questões em aberto vindas da reunião.
6. **FDD** com modelo de dados, fluxos, contratos, matriz de erros, observabilidade e a seção "Integração com o sistema existente".
7. **PRD** por último entre os grandes documentos, como consolidação.
8. **Tracker** varrendo os documentos prontos, linha a linha.
9. **Revisão** com script (arquivos citados existem? ≥70% TRANSCRICAO? ≥5 CODIGO?) e checklist de critérios de aceite.

Regra de ouro adotada nos prompts: tudo que não está na transcrição ou no código precisa vir rotulado como **[Proposta FDD]** e apontar a fala ou o arquivo que o motivou. Isso mantém visível a fronteira entre decisão e sugestão.

## Prompts customizados

**1. Extração dirigida da transcrição, com filtro do que NÃO entra**

```text
Leia TRANSCRICAO.md inteira. Não escreva nenhum documento ainda.
Devolva uma tabela com 6 blocos, sempre citando [hh:mm] Falante:
 A) Decisões FECHADAS (quem decidiu e o que foi decidido, em uma frase)
 B) Requisitos funcionais explícitos
 C) Restrições e requisitos não funcionais (números exatos: timeouts, tamanhos, intervalos)
 D) Ganchos com o código existente (cite função/arquivo que a fala menciona)
 E) DESCARTADO ou ADIADO (o que NÃO pode virar requisito) e o motivo
 F) Lacunas: pontos que o documento precisaria definir e que ninguém discutiu
Se uma informação não tiver timestamp de origem, não inclua. Não complete lacunas com suposição.
```

**2. Geração de FDD ancorada no código real, com marcação de proposta**

```text
Escreva docs/FDD.md. Antes, abra e leia: order.service.ts, order.status.ts, shared/errors/*,
error.middleware.ts, auth.middleware.ts, app.ts, routes/index.ts, server.ts, logger.
Regras:
 - Cada afirmação sobre o código deve citar um caminho que exista (verifique com ls).
 - Tudo que a reunião não definiu (nome de rota de rotação, colunas auxiliares, métricas)
   deve ser marcado "[Proposta FDD]" com a fala que motivou.
 - Nada do bloco E (descartado/adiado) pode aparecer como requisito.
 - Contratos: >= 4 endpoints com request, response e status codes; erros com prefixo WEBHOOK_.
 - Seção "Integração com o sistema existente": tabela caminho -> como o módulo se integra.
 - Não duplique o nível de detalhe no RFC; o RFC fala em decisão, o FDD em implementação.
```

**3. Auditoria cruzada (usado no fim)**

```text
Varra docs/*.md e docs/adrs/*.md. Para cada requisito, decisão, restrição, risco e
contrato, confirme que existe uma linha em docs/TRACKER.md com ID, fonte e localização.
Liste o que está sem linha, e o que contradiz a transcrição ou o código.
```

## Iterações e ajustes

Foram **3 ciclos principais** de geração, revisão crítica e correção.

1. **Primeira passada superficial → triagem dirigida.** Um pedido genérico de "ler a transcrição e fazer os documentos" tenderia a misturar o que foi adiado com o que foi decidido. Foi necessário separar antes o bloco "descartado/adiado" (e-mail, rate limiting, dashboard, arquivamento, múltiplos workers), que então alimentou "Fora de escopo" do PRD e "Questões em aberto" do RFC.
2. **Lacunas escondidas na transcrição.** Ao detalhar o FDD apareceram pontos que a reunião não resolve, e que a IA tenderia a preencher em silêncio:
   - *"5 tentativas"* vs. cinco intervalos (1m/5m/30m/2h/12h somam ≈14h36 → 5 **reenvios** após a 1ª falha). Passou a constar como questão em aberto (RFC Q5) e como premissa explícita no FDD.
   - *Payload > 64 KB:* a primeira ideia foi validar no momento de inserir na outbox, o que daria **rollback da mudança de status** por causa do tamanho do evento, contradizendo a razão de existir da outbox. Foi movido para o worker (vai direto à DLQ com `WEBHOOK_PAYLOAD_TOO_LARGE`).
   - *Ordenação:* percebi que o retry pode reordenar eventos do mesmo pedido, e que a "ordenação por `order_id`" da reunião só vale sem falhas. Isso virou limitação documentada no FDD e risco no PRD.
   - *Logger:* ao ler `src/shared/logger/index.ts`, `redactPaths` não cobre `secret`; dado o histórico de vazamento citado por Diego, entrou como ajuste necessário no FDD.
3. **Verificação por script e correção de inconsistências.**
   - O script de checagem encontrou arquivos novos (`src/worker.ts`, `webhook.processor.ts`, `webhook.publisher.ts`, `webhook.worker.ts`) citados como se existissem; passaram a ser marcados *(arquivo novo, a criar)* para não violar "nenhum arquivo citado é inexistente".
   - Um exemplo de resposta de `deliveries` mostrava `statusCode: 503` junto de erro de timeout (incoerente, pois timeout não tem status); corrigido.
   - Um ADR tinha uma referência de timestamp malformada (`[08:08]→[09:08]`); corrigida.
   - Resultado do Tracker: 190 linhas, 78% com Fonte = TRANSCRICAO e 41 com Fonte = CODIGO (caminhos verificados).

Itens que propus além da reunião (rota de rotação de secret, tabela `webhook_deliveries`, `nextAttemptAt`, assinatura dupla na rotação, status 202 no replay, recuperação de `PROCESSING`, métricas) estão sempre rotulados como **Proposta** para a revisão do time.

## Como navegar a entrega

Ordem sugerida de leitura:

1. [docs/PRD.md](docs/PRD.md): por quê e o quê (produto)
2. [docs/RFC.md](docs/RFC.md): proposta técnica, alternativas e questões em aberto
3. [docs/adrs/](docs/adrs/): decisões isoladas
   - [ADR-001 Outbox no MySQL](docs/adrs/ADR-001-outbox-no-mysql.md)
   - [ADR-002 Worker separado em polling](docs/adrs/ADR-002-worker-separado-em-polling.md)
   - [ADR-003 Retry, backoff e DLQ](docs/adrs/ADR-003-retry-backoff-e-dlq.md)
   - [ADR-004 HMAC-SHA256 e secret por endpoint](docs/adrs/ADR-004-hmac-sha256-secret-por-endpoint.md)
   - [ADR-005 At-least-once com X-Event-Id](docs/adrs/ADR-005-at-least-once-com-x-event-id.md)
   - [ADR-006 Reuso dos padrões existentes](docs/adrs/ADR-006-reuso-dos-padroes-existentes.md)
   - [ADR-007 Publicação transacional em changeStatus](docs/adrs/ADR-007-publicacao-transacional-em-change-status.md)
4. [docs/FDD.md](docs/FDD.md): como construir (modelo de dados, fluxos, contratos, erros, integração)
5. [docs/TRACKER.md](docs/TRACKER.md): origem de cada item na transcrição ou no código

Fonte primária: [TRANSCRICAO.md](TRANSCRICAO.md) (não alterada).
