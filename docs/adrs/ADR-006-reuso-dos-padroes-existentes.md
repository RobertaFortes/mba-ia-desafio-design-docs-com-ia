# ADR-006: Reuso máximo dos padrões existentes do projeto

- **Status:** Aceito
- **Data:** reunião técnica de quinta-feira, 09:00 (decisão em [09:30] Larissa)
- **Decisores:** Larissa, Bruno, Diego
- **Relacionados:** [ADR-002](ADR-002-worker-separado-em-polling.md), [ADR-007](ADR-007-publicacao-transacional-em-change-status.md)

## Contexto

A codebase tem convenções claras: cada domínio é um módulo em `src/modules/<dominio>` com `controller`, `service`, `repository`, `routes` e `schemas`; erros herdam de `AppError`; logging é Pino; o `errorMiddleware` trata centralmente `AppError`, `ZodError` e erros do Prisma ([09:27] e [09:29] Bruno). O objetivo é entregar em três sprints com um time pequeno ([09:47] Larissa).

## Decisão

Seguir os padrões existentes, sem introduzir bibliotecas ou convenções novas:

- **Estrutura de módulo:** `src/modules/webhooks/` com `webhook.controller.ts`, `webhook.service.ts`, `webhook.repository.ts`, `webhook.routes.ts`, `webhook.schemas.ts`, mais `webhook.worker.ts`/`webhook.processor.ts` ([09:27]/[09:28] Bruno). Registro no `src/app.ts` (`buildControllers`) e `src/routes/index.ts` (`buildApiRouter`), como os demais módulos.
- **Erros:** classes derivadas de `AppError` (`src/shared/errors/app-error.ts`, `http-errors.ts`) com `errorCode` prefixado `WEBHOOK_` (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`) ([09:28] Bruno, [09:29] Larissa). O `errorMiddleware` (`src/middlewares/error.middleware.ts`) permanece **inalterado**.
- **Logging:** `logger` Pino de `src/shared/logger/index.ts`; nada novo ([09:29] Bruno).
- **Validação:** schemas Zod + `validate` (`src/middlewares/validate.middleware.ts`), incluindo a regra de URL https ([09:23] Sofia).
- **Autorização:** `authenticate` para o CRUD (qualquer role autenticada) e `requireRole('ADMIN')` do `src/middlewares/auth.middleware.ts` para o replay ([09:36] Larissa).
- **Identificadores:** UUID em todas as tabelas novas, como `prisma/schema.prisma` ([09:51] Larissa).

## Alternativas Consideradas

1. **Biblioteca/framework de filas ou de eventos dedicada.** Contraria o objetivo de não adicionar infra ([09:07] Diego); não foi proposta formalmente, é alternativa plausível.
2. **Estrutura própria para o módulo (fora de `src/modules`).** Não considerada na reunião; descartada implicitamente pela proposta de Bruno, aceita por Diego ([09:28] Diego).

## Consequências

**Positivas**
- Curva de aprendizado zero; code review e testes seguem o que o time já conhece.
- Respostas de erro com o mesmo formato `{ error: { code, message, details } }` já consumido pelos clientes da API.
- Menos superfície nova para a revisão de segurança.

**Negativas / trade-offs**
- O módulo herda as limitações atuais (ex.: o mapeamento de erro do Prisma no middleware é genérico).
- O worker é um segundo processo e não passa por `buildApp`; precisa montar suas próprias dependências, reaproveitando só o que for independente de HTTP (logger, erros, env).
- Aderir ao padrão de `env` significa que novas variáveis do worker devem entrar no schema Zod de `src/config/env.ts`, o que é uma mudança de código futura (fora do escopo documental).
