# ADR-006 — Reuso máximo dos padrões existentes do projeto

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:30] Larissa: "Decisão: reuso máximo do que já existe").

**Decisores:** Bruno (Eng. Pleno — Pedidos, dono da proposta), Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma).

## Contexto

A codebase do OMS tem padrões consolidados e consistentes entre os módulos existentes (`auth`, `users`, `customers`, `products`, `orders`). O módulo de webhooks é o primeiro a ser adicionado depois desses padrões estabelecidos, e a reunião discutiu se ele deveria seguir a estrutura existente ou introduzir algo novo ([09:27] Bruno: "A gente tem um padrão claro na codebase").

Padrões existentes, verificados no código:

| Padrão | Onde está no código |
| --- | --- |
| Módulo por domínio com `controller`, `service`, `repository`, `routes`, `schemas` | `src/modules/orders/`, `src/modules/customers/`, etc. |
| Classe base de erro com `errorCode` | `src/shared/errors/app-error.ts` (`AppError`) |
| Erros específicos com códigos em SCREAMING_SNAKE_CASE (`INSUFFICIENT_STOCK`, `INVALID_STATUS_TRANSITION`) | `src/shared/errors/http-errors.ts` (`InsufficientStockError`, `InvalidStatusTransitionError`) |
| Logger estruturado Pino com redação de campos sensíveis | `src/shared/logger/index.ts` |
| Middleware de erro centralizado que trata `AppError`, `ZodError` e erros do Prisma | `src/middlewares/error.middleware.ts` |
| Validação de entrada com schemas Zod | `src/middlewares/validate.middleware.ts` + `*.schemas.ts` por módulo |
| Autenticação JWT e autorização por role | `src/middlewares/auth.middleware.ts` (`authenticate`, `requireRole`) |
| Entry-point de processo | `src/server.ts` |
| IDs UUID em todos os models | `prisma/schema.prisma` (`@default(uuid())`) |

## Decisão

O módulo de webhooks **segue integralmente os padrões existentes** ([09:30] Larissa):

1. Nova pasta **`src/modules/webhooks`** com controller, service, repository, routes e schemas, igual aos demais módulos ([09:27] Bruno).
2. Erros do módulo estendem **`AppError`** com códigos prefixados **`WEBHOOK_`** (ex.: `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL`, `WEBHOOK_SECRET_REQUIRED`), no mesmo molde de `INSUFFICIENT_STOCK` / `INVALID_STATUS_TRANSITION` ([09:28] Bruno; [09:29] Larissa).
3. **Logger Pino existente**, sem nada novo; o **middleware de erro centralizado já trata `AppError`, Zod e Prisma** e pega os erros do módulo sem mudança alguma ([09:29] Bruno).
4. Worker como **entry-point separada `src/worker.ts`** com script `npm run worker`, seguindo o molde de `src/server.ts`; a lógica de processamento vive dentro do módulo (ex.: `src/modules/webhooks/webhook.processor.ts`) ([09:11] Larissa; [09:28] Bruno).
5. O worker abre **`PrismaClient` próprio** (client é por processo), mesma `DATABASE_URL` ([09:30] Bruno).
6. Replay de DLQ protegido reaproveitando o **`requireRole`** existente ([09:36] Larissa).
7. Novas tabelas usam **UUID como chave primária**, seguindo o padrão de todo o projeto ([09:51] Larissa: "UUID, segue o padrão do resto do projeto").

## Alternativas Consideradas

### 1. Introduzir framework/lib de jobs ou fila dedicada para o worker — descartada

- **Trade-off que motivou o descarte:** mesma lógica do descarte do Redis Streams — time pequeno, infra e curva de aprendizado extras sem necessidade; o requisito de latência é atendido com polling simples sobre a stack atual ([09:07] Diego; [09:09] Diego).

### 2. Estrutura de pastas própria para o módulo de webhooks — descartada

- **Trade-off que motivou o descarte:** quebraria a consistência da codebase; o padrão de módulos existente é explícito e funciona ("Webhook vai seguir igual", [09:27] Bruno; confirmado por Diego [09:28]).

## Consequências

### Positivas

- Velocidade de implementação e de revisão: qualquer pessoa do time reconhece a estrutura de imediato.
- **Zero mudança em infraestrutura transversal**: o middleware de erro, o logger e a autenticação funcionam para o novo módulo sem alteração ([09:29] Bruno).
- Erros do módulo saem no mesmo envelope JSON de erro da API inteira (formato de `src/middlewares/error.middleware.ts`).

### Negativas / trade-offs assumidos

- O módulo herda as limitações da stack atual (Express, Prisma, processo Node único por entry-point).
- Duas instâncias de `PrismaClient` (API e worker) consomem dois pools de conexão contra o mesmo MySQL — aceitável no volume atual, mas passa a ser variável de capacity planning.
