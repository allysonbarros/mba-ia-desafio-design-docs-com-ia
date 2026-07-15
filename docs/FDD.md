# FDD — Sistema de Webhooks de Notificação de Pedidos

> **Documentos relacionados:** [PRD](PRD.md) (por que e o quê) · [RFC](RFC.md) (proposta e alternativas) · [ADRs](adrs/README.md) (decisões) · [Tracker](TRACKER.md) (rastreabilidade)

## 1. Contexto e motivação técnica

O OMS controla o ciclo de vida de pedidos com máquina de estados explícita (`src/modules/orders/order.status.ts`) e efetua mudanças de status em transação única no `OrderService.changeStatus` (`src/modules/orders/order.service.ts`): update em `orders`, insert em `order_status_history` e débito/reposição de `stockQuantity`. Não existe hoje nenhum mecanismo de notificação externa — clientes B2B fazem polling em `GET /orders` ([09:00] Marcos).

Este documento especifica **como implementar** o sistema de webhooks outbound decidido na reunião técnica ([`TRANSCRICAO.md`](../TRANSCRICAO.md)): outbox transacional no MySQL, worker separado em polling, retry com backoff + DLQ, HMAC-SHA256 e entrega at-least-once. O racional das decisões está nos [ADRs](adrs/README.md); aqui está o detalhe acionável.

## 2. Objetivos técnicos

1. Registrar o evento de webhook **na mesma transação** da mudança de status — nunca pode haver status alterado sem evento registrado, nem evento registrado de transação que sofreu rollback ([09:40] Bruno; [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md)).
2. Entregar notificações com latência total **abaixo de 10 segundos** no caminho feliz ([09:02] Marcos), com piso de ~2s imposto pelo polling ([09:10] Larissa).
3. Sobreviver a indisponibilidades de clientes de até ~15h via retry com backoff; além disso, preservar o evento em DLQ com evidência para replay ([09:15]–[09:18]).
4. Permitir ao cliente validar origem e integridade de cada entrega (HMAC-SHA256, secret por endpoint) ([09:19]–[09:22] Sofia).
5. Zero mudança em infraestrutura transversal existente: error middleware, logger, autenticação e padrão de módulos são reusados como estão ([09:29]–[09:30]; [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)).

## 3. Escopo e exclusões

### No escopo

- Módulo `src/modules/webhooks` (controller, service, repository, routes, schemas) e entry-point `src/worker.ts` com script `npm run worker` — **arquivos novos, a criar**.
- Tabelas novas: `webhook_endpoints`, `webhook_outbox`, `webhook_deliveries`, `webhook_dead_letter`.
- CRUD de configuração de webhooks, rotação de secret, histórico de entregas, replay de DLQ (admin).
- Extensão pontual do `OrderService.changeStatus` com `publishWebhookEvent(tx, order, fromStatus, toStatus)`.

### Exclusões (explicitamente descartadas ou adiadas na reunião)

| Item | Situação | Origem |
| --- | --- | --- |
| Notificação por email quando webhook falha repetidamente | Adiado para próxima fase | [09:37] Larissa |
| Rate limiting de envio para o cliente | "Observar e decidir depois" | [09:38]–[09:39] Diego/Larissa |
| Dashboard/painel visual para o cliente | Fora de escopo — projeto do time de frontend | [09:39]–[09:40] Larissa |
| Webhooks inbound (cliente → plataforma) | Fora de escopo — só outbound | [09:02] Marcos |
| Arquivamento de linhas entregues da outbox (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| Múltiplos workers em paralelo (particionamento/lock) | "Problema do futuro" | [09:13] Diego |
| Garantia exactly-once | Descartada | [09:25] Diego |

## 4. Modelo de dados

Novas tabelas no `prisma/schema.prisma`, seguindo os padrões existentes: PK `String @id @default(uuid()) @db.Char(36)` e nomes de tabela em snake_case via `@@map` ([09:51] Larissa: "UUID, segue o padrão do resto do projeto").

### 4.1 `webhook_endpoints` — configuração de webhook

Campos derivados de [09:21] Bruno ("url + secret + customer_id + estado ativo"), [09:21] Sofia (rotação com grace period) e [09:33] Marcos (filtro de eventos):

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | CHAR(36) UUID | PK; enviado como `X-Webhook-Id` nas entregas |
| `customerId` | CHAR(36) | FK → `customers.id` |
| `url` | VARCHAR(2048) | Obrigatoriamente `https` (validação Zod) |
| `secret` | VARCHAR(255) | Gerada pelo sistema; usada na assinatura HMAC |
| `previousSecret` | VARCHAR(255) NULL | Secret anterior durante rotação |
| `previousSecretExpiresAt` | DATETIME NULL | `now() + 24h` no momento da rotação |
| `events` | JSON | Lista de `OrderStatus` que o endpoint quer ouvir (ex.: `["SHIPPED","DELIVERED"]`) |
| `active` | BOOLEAN default true | Endpoint inativo não recebe eventos |
| `createdAt` / `updatedAt` | DATETIME | Padrão do projeto |

Índice: `customerId`.

### 4.2 `webhook_outbox` — eventos pendentes de entrega

Derivada de [09:06]–[09:08] Diego (outbox, estados, índices) e [09:52] (payload snapshot):

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | CHAR(36) UUID | PK; **é o `event_id`** enviado em `X-Event-Id` ([09:25] Diego) |
| `webhookEndpointId` | CHAR(36) | FK → `webhook_endpoints.id` |
| `orderId` | CHAR(36) | FK → `orders.id` |
| `eventType` | VARCHAR(64) | `order.status_changed` ([09:43] Diego) |
| `payload` | JSON | **Snapshot renderizado na inserção** ([ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md)) |
| `status` | ENUM `PENDING` / `PROCESSING` / `FAILED` / `DELIVERED` | Estados citados em [09:07] Diego (pendente, processando, falhou, entregue) |
| `attempts` | INT default 0 | Máximo 5 ([ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md)) |
| `nextAttemptAt` | DATETIME NULL | Agenda do backoff |
| `lastError` | VARCHAR(500) NULL | Última causa de falha |
| `createdAt` / `updatedAt` | DATETIME | — |

Índices: `status` e `createdAt` ([09:07] Diego), compostos com `nextAttemptAt` para a query do worker.

### 4.3 `webhook_deliveries` — histórico de tentativas de entrega

Derivada de [09:34] Marcos ("sucesso/falha, payload, response, tempo de resposta"):

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | CHAR(36) UUID | PK |
| `outboxEventId` | CHAR(36) | `event_id` do evento |
| `webhookEndpointId` | CHAR(36) | FK → `webhook_endpoints.id` |
| `attempt` | INT | Nº da tentativa (1–5) |
| `success` | BOOLEAN | 2xx dentro do timeout |
| `responseStatus` | INT NULL | Status HTTP retornado pelo cliente |
| `responseBody` | TEXT NULL | Truncado para armazenamento |
| `durationMs` | INT | Tempo de resposta |
| `errorCode` | VARCHAR(64) NULL | Ex.: `WEBHOOK_DELIVERY_TIMEOUT` |
| `createdAt` | DATETIME | — |

Índices: `webhookEndpointId` + `createdAt` (consulta dos "últimos 100").

### 4.4 `webhook_dead_letter` — falhas permanentes

Derivada de [09:17] Diego ("payload, motivo da falha e timestamp"):

| Campo | Tipo | Notas |
| --- | --- | --- |
| `id` | CHAR(36) UUID | PK (usado no replay) |
| `outboxEventId` | CHAR(36) | `event_id` original — preservado no replay para manter a dedup do cliente |
| `webhookEndpointId` | CHAR(36) | FK |
| `payload` | JSON | Snapshot original |
| `failureReason` | VARCHAR(500) | Ex.: `5 tentativas esgotadas: connect timeout` |
| `failedAt` | DATETIME | Timestamp da falha permanente |

## 5. Fluxos detalhados

### 5.1 Criação do evento na outbox (dentro da transação de mudança de status)

Extensão do `OrderService.changeStatus` (`src/modules/orders/order.service.ts`), que hoje executa dentro de `this.prisma.$transaction`: validação da transição (`canTransition`), débito/reposição de estoque, update da order e insert no history. O novo passo entra **antes do commit**, na mesma transação ([09:40] Bruno; [09:41] Diego: "Se ficar fora da transação, perde a garantia toda").

Contrato da função ([09:41] Bruno — função pura recebendo o `tx`, sem injetar repository):

```ts
// src/modules/webhooks/webhook.publisher.ts (arquivo novo)
export async function publishWebhookEvent(
  tx: Prisma.TransactionClient,
  order: Order,
  fromStatus: OrderStatus,
  toStatus: OrderStatus,
): Promise<void>
```

Passos:

1. Busca `webhook_endpoints` ativos do `order.customerId` cuja lista `events` contém `toStatus`.
2. **Filtro na inserção** ([09:34] Bruno): se nenhum endpoint escuta esse status, não insere nada — "economiza linha na tabela".
3. Para cada endpoint interessado: gera `event_id` (UUID), renderiza o **payload snapshot** (seção 6.3) com o estado atual da order e insere linha `PENDING` na `webhook_outbox`.
4. Serializa o payload e valida o teto de **64KB**; acima disso a inserção falha com `WEBHOOK_PAYLOAD_TOO_LARGE` ([09:23]–[09:24] — "erra", não trunca).
5. Qualquer falha na inserção propaga a exceção → **rollback da transação inteira**, inclusive da mudança de status ([09:40] Bruno).

### 5.2 Processamento pelo worker

Entry-point `src/worker.ts` (molde de `src/server.ts`: bootstrap com logger, graceful shutdown em SIGINT/SIGTERM) + `src/modules/webhooks/webhook.processor.ts` ([09:28] Bruno). `PrismaClient` **próprio do processo**, criado pelo mesmo padrão de `src/config/database.ts` ([09:30] Bruno).

Loop ([09:09] Diego):

```
a cada 2s:
  eventos = SELECT * FROM webhook_outbox
            WHERE (status = 'PENDING')
               OR (status = 'FAILED' AND next_attempt_at <= NOW())
            ORDER BY created_at ASC
            LIMIT <batch pequeno>          -- [09:08] Diego
  para cada evento:
    marca status = 'PROCESSING'
    monta headers (seção 6.3) e assina payload com HMAC-SHA256
    POST na url do endpoint, timeout 10s   -- [09:42] Diego
    registra linha em webhook_deliveries (attempt, success, responseStatus, durationMs)
    se resposta 2xx: status = 'DELIVERED'
    senão: fluxo de retry (5.3)
```

Processamento sequencial em ordem de `created_at`, single-worker ⇒ ordering por `order_id` preservada ([09:12] Diego).

### 5.3 Retry

Política do [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) ([09:17] Larissa):

| Tentativa | Espera após a falha | Tempo acumulado desde a 1ª falha |
| --- | --- | --- |
| 1ª → 2ª | 1 minuto | 1 min |
| 2ª → 3ª | 5 minutos | 6 min |
| 3ª → 4ª | 30 minutos | 36 min |
| 4ª → 5ª | 2 horas | ~2h36 |
| 5ª → DLQ | 12 horas | **~14h36 (~15h)** ([09:17] Diego) |

Em cada falha (timeout de 10s, erro de rede, resposta não-2xx):

1. `attempts += 1`, `lastError` preenchido, delivery registrada com `success = false`.
2. Se `attempts < 5`: `status = 'FAILED'`, `nextAttemptAt = now() + backoff[attempts]`.
3. Se `attempts = 5`: fluxo de DLQ (5.4).

### 5.4 DLQ e replay

1. Ao esgotar as 5 tentativas, o worker insere em `webhook_dead_letter` (payload, `failureReason`, `failedAt`) e marca o evento da outbox como encerrado — a tabela separada mantém a outbox limpa e serve de evidência para debug ([09:18] Diego).
2. Replay manual: `POST /api/v1/admin/webhooks/dead-letter/:id/replay` ([09:18] Diego), exige **role `ADMIN`** via `requireRole('ADMIN')` de `src/middlewares/auth.middleware.ts` ([09:36] Larissa).
3. O replay recoloca o evento na outbox como `PENDING` com `attempts = 0`, **preservando o `event_id` original** — para o cliente é uma reentrega do mesmo evento (dedup via `X-Event-Id`, [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md)).
4. **Auditoria obrigatória**: log estruturado com o `userId` de quem executou o replay ([09:36] Sofia: "o endpoint de admin tem que logar quem fez o replay").

### 5.5 Rotação de secret

Fluxo de [09:21] Sofia:

1. Cliente chama `POST /api/v1/webhooks/:id/rotate-secret`.
2. Sistema gera nova secret; a atual vira `previousSecret` com `previousSecretExpiresAt = now() + 24h`.
3. Durante o grace period, cada entrega é assinada com **ambas as secrets** e o header `X-Signature` carrega as duas assinaturas separadas por vírgula (nova primeiro); o cliente valida contra qualquer uma. *(Detalhe de implementação definido neste FDD para honrar "a antiga fica válida por 24 horas em paralelo".)*
4. Expirado o grace period, `previousSecret` é descartada e a assinatura volta a ser única.

## 6. Contratos públicos

Todos os endpoints ficam sob `/api/v1` (padrão de `src/app.ts`), autenticados com JWT via middleware `authenticate` ([09:32] Marcos: "Pela nossa API direto, autenticado com JWT do nosso sistema"). O `customer_id` vai no body/query — o JWT identifica o usuário operador, não o cliente ([09:32] Larissa). Erros seguem o envelope de `src/middlewares/error.middleware.ts`:

```json
{ "error": { "code": "WEBHOOK_NOT_FOUND", "message": "...", "details": {} } }
```

### 6.1 CRUD de configuração

#### `POST /api/v1/webhooks` — cadastrar webhook ([09:31] Marcos)

Request:

```json
{
  "customerId": "9b2f1c34-6d1a-4f0e-9b1d-2a7c3e5f8a01",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "events": ["SHIPPED", "DELIVERED"]
}
```

Response `201 Created` — a secret é **gerada pelo sistema e devolvida na criação** ([09:31] Marcos); é exibida apenas aqui e na rotação:

```json
{
  "id": "1a2b3c4d-0000-4000-8000-1234567890ab",
  "customerId": "9b2f1c34-6d1a-4f0e-9b1d-2a7c3e5f8a01",
  "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
  "events": ["SHIPPED", "DELIVERED"],
  "active": true,
  "secret": "whsec_4f8a01c3e5f2a7c39b1d6d1a4f0e9b2f",
  "createdAt": "2026-11-03T12:00:00.000Z"
}
```

Status codes: `201` criado · `400` `WEBHOOK_INVALID_URL` (url não-https) ou `WEBHOOK_INVALID_EVENTS` · `401` sem JWT · `404` `NOT_FOUND` (customer inexistente).

#### `GET /api/v1/webhooks?customerId=<uuid>` — listar webhooks de um customer ([09:33] Bruno)

Response `200 OK` (paginação no padrão de `src/shared/http/response.ts`; a secret **não** é retornada na listagem):

```json
{
  "data": [
    {
      "id": "1a2b3c4d-0000-4000-8000-1234567890ab",
      "customerId": "9b2f1c34-6d1a-4f0e-9b1d-2a7c3e5f8a01",
      "url": "https://integracao.atlascomercial.com.br/oms/webhooks",
      "events": ["SHIPPED", "DELIVERED"],
      "active": true,
      "createdAt": "2026-11-03T12:00:00.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 20, "total": 1, "totalPages": 1 }
}
```

Status codes: `200` · `400` query inválida · `401` sem JWT.

#### `PATCH /api/v1/webhooks/:id` — editar webhook ([09:33] Bruno)

Request (campos opcionais: `url`, `events`, `active`):

```json
{ "events": ["PAID", "SHIPPED", "DELIVERED"], "active": true }
```

Response `200 OK` — recurso atualizado (sem secret). Status codes: `200` · `400` `WEBHOOK_INVALID_URL` / `WEBHOOK_INVALID_EVENTS` · `401` · `404` `WEBHOOK_NOT_FOUND`.

#### `DELETE /api/v1/webhooks/:id` — remover webhook ([09:33] Bruno)

Response `204 No Content`. Status codes: `204` · `401` · `404` `WEBHOOK_NOT_FOUND`.

### 6.2 Operação

#### `POST /api/v1/webhooks/:id/rotate-secret` — rotacionar secret ([09:21] Sofia)

Request: corpo vazio. Response `200 OK`:

```json
{
  "id": "1a2b3c4d-0000-4000-8000-1234567890ab",
  "secret": "whsec_novo9b1d6d1a4f0e9b2f4f8a01c3e5f2a7",
  "previousSecretExpiresAt": "2026-11-04T12:00:00.000Z"
}
```

Status codes: `200` · `401` · `404` `WEBHOOK_NOT_FOUND` · `409` `WEBHOOK_DISABLED` (endpoint inativo).

#### `GET /api/v1/webhooks/:id/deliveries` — histórico de entregas ([09:34] Marcos)

Retorna as últimas entregas (default e teto: 100 — "esses são os últimos 100 webhooks que vocês mandaram pra mim"). Response `200 OK`:

```json
{
  "data": [
    {
      "id": "7c8d9e0f-0000-4000-8000-abcdef123456",
      "eventId": "5f0c9a7e-0000-4000-8000-fedcba654321",
      "attempt": 2,
      "success": true,
      "responseStatus": 200,
      "responseBody": "{\"ok\":true}",
      "durationMs": 184,
      "payload": { "event_type": "order.status_changed", "to_status": "SHIPPED" },
      "createdAt": "2026-11-20T14:32:13.000Z"
    }
  ],
  "pagination": { "page": 1, "pageSize": 100, "total": 42, "totalPages": 1 }
}
```

Status codes: `200` · `401` · `404` `WEBHOOK_NOT_FOUND`.

#### `POST /api/v1/admin/webhooks/dead-letter/:id/replay` — replay de DLQ ([09:35] Diego)

Exige role `ADMIN` ([09:36] Sofia). Request: corpo vazio. Response `202 Accepted`:

```json
{
  "deadLetterId": "3e4f5a6b-0000-4000-8000-111122223333",
  "eventId": "5f0c9a7e-0000-4000-8000-fedcba654321",
  "status": "PENDING",
  "replayedBy": "8a7b6c5d-0000-4000-8000-999988887777"
}
```

Status codes: `202` reenfileirado · `401` sem JWT · `403` `FORBIDDEN` (role ≠ ADMIN, via `requireRole`) · `404` `WEBHOOK_DEAD_LETTER_NOT_FOUND`.

### 6.3 Contrato da entrega (nossa chamada ao endpoint do cliente)

`POST <url do webhook>` com headers ([09:44] Diego e Sofia):

| Header | Conteúdo |
| --- | --- |
| `Content-Type` | `application/json` |
| `X-Event-Id` | UUID do evento (gerado na inserção na outbox) — chave de dedup do cliente |
| `X-Signature` | HMAC-SHA256 (hex) do corpo do request; durante grace period de rotação, duas assinaturas separadas por vírgula |
| `X-Timestamp` | Timestamp ISO 8601 do envio — permite ao cliente detectar replay attack |
| `X-Webhook-Id` | id do endpoint webhook — distingue múltiplos cadastros do mesmo cliente |

Body — payload snapshot ([09:43] Diego: enxuto, **sem `items`**; detalhes via `GET /orders/:id`):

```json
{
  "event_id": "5f0c9a7e-0000-4000-8000-fedcba654321",
  "event_type": "order.status_changed",
  "timestamp": "2026-11-20T14:32:11.000Z",
  "order_id": "0d1e2f3a-0000-4000-8000-444455556666",
  "order_number": "ORD-000123",
  "from_status": "PROCESSING",
  "to_status": "SHIPPED",
  "customer_id": "9b2f1c34-6d1a-4f0e-9b1d-2a7c3e5f8a01",
  "total_cents": 15990
}
```

Semântica de resposta esperada do cliente: **qualquer 2xx dentro de 10s = entregue**; qualquer outra coisa (não-2xx, timeout, erro de rede) = falha e entra no fluxo de retry. Entrega é **at-least-once**: o cliente deve deduplicar por `event_id` ([09:25] Diego).

## 7. Matriz de erros (`WEBHOOK_*`)

Padrão do projeto: classes estendendo `AppError` (`src/shared/errors/app-error.ts`), no molde de `InsufficientStockError`/`InvalidStatusTransitionError` (`src/shared/errors/http-errors.ts`), tratadas sem alteração pelo `src/middlewares/error.middleware.ts` ([09:28]–[09:29] Bruno). Prefixo `WEBHOOK_` obrigatório ([09:29] Larissa).

| Código | HTTP | Quando ocorre | Origem |
| --- | --- | --- | --- |
| `WEBHOOK_NOT_FOUND` | 404 | Webhook `:id` inexistente em qualquer operação do CRUD/deliveries/rotação | [09:28] Bruno (código citado) |
| `WEBHOOK_INVALID_URL` | 400 | URL inválida ou não-`https` no cadastro/edição (validação Zod) | [09:28] Bruno; [09:23] Sofia (TLS obrigatório) |
| `WEBHOOK_SECRET_REQUIRED` | 422 | Operação que exige secret ativa em endpoint sem secret válida (estado inconsistente detectado) | [09:28] Bruno (código citado) |
| `WEBHOOK_INVALID_EVENTS` | 400 | Lista `events` vazia ou com valor fora do enum `OrderStatus` (`prisma/schema.prisma`) | Derivado de [09:33] Marcos (filtro por lista de status) |
| `WEBHOOK_DISABLED` | 409 | Operação de rotação/replay sobre endpoint com `active = false` | Derivado de [09:21] Bruno (campo "estado ativo") |
| `WEBHOOK_DEAD_LETTER_NOT_FOUND` | 404 | Replay de item inexistente na DLQ | Derivado de [09:35] Diego (endpoint de replay por `:id`) |
| `WEBHOOK_PAYLOAD_TOO_LARGE` | 422 | Payload serializado excede 64KB — erro, não trunca | [09:23] Sofia; [09:24] Diego; [09:24] Larissa |
| `WEBHOOK_DELIVERY_TIMEOUT` | — (interno) | Cliente não respondeu em 10s; registrado em `webhook_deliveries.errorCode` e conta como falha para retry | [09:42] Diego |
| `WEBHOOK_DELIVERY_FAILED` | — (interno) | Resposta não-2xx ou erro de rede; registrado na delivery | Derivado de [09:34] Marcos (histórico sucesso/falha) |

## 8. Estratégias de resiliência

| Mecanismo | Especificação | Origem |
| --- | --- | --- |
| **Timeout** | 10s por chamada HTTP do worker; estouro = falha marcada para retry | [09:42] Diego |
| **Retry** | Backoff exponencial 1m/5m/30m/2h/12h, máximo 5 tentativas (~15h de janela) | [09:17] Larissa |
| **Fallback (falha permanente)** | Evento vai à `webhook_dead_letter` com payload, motivo e timestamp; replay manual admin | [09:17]–[09:18] Diego |
| **Atomicidade** | Evento inserido na transação da mudança de status; falha na outbox = rollback total | [09:40] Bruno |
| **Isolamento de processo** | Worker separado da API: restart de um não afeta o outro | [09:11] Diego |
| **Recuperação do worker** | Estado vive no banco (`status`/`nextAttemptAt`); após crash/restart, o polling retoma do ponto onde parou sem perda — eventos apenas atrasam | Derivado do desenho outbox ([09:06] Diego) |
| **Limite de payload** | 64KB; acima disso erro na inserção (rollback) — nunca envio truncado | [09:24] Larissa |
| **Duplicatas** | Esperadas por design (at-least-once); dedup do cliente por `X-Event-Id` | [09:25] Diego |

## 9. Observabilidade

### Logs

- Logger **Pino existente** (`src/shared/logger/index.ts`), sem novidades ([09:29] Bruno). O worker cria o logger pelo mesmo `createLogger()`, com `base.service` distinto (ex.: `order-management-worker`).
- Acrescentar `'*.secret'` e `'*.previousSecret'` aos `redactPaths` do logger para as secrets nunca vazarem em log — motivado pelo incidente real de secret vazada em log ([09:22] Diego).
- Eventos logados no worker (estruturados, um por linha, padrão do `request-logger.middleware.ts`): `webhook_delivery_attempt`, `webhook_delivery_success`, `webhook_delivery_failure`, `webhook_moved_to_dlq` — sempre com `eventId`, `webhookId`, `orderId`, `attempt`, `durationMs`, `responseStatus`.
- **Auditoria**: `webhook_dlq_replay` com `userId` do admin que executou ([09:36] Sofia).

### Métricas

Sem stack nova de métricas nesta fase (coerente com a decisão de não subir infra — [09:07] Diego); as métricas operacionais saem das tabelas e dos logs estruturados:

- **Backlog da outbox**: `COUNT(*) WHERE status = 'PENDING'` — se cresce, worker parado ou lento.
- **Taxa de sucesso/falha por endpoint**: agregação de `webhook_deliveries.success`.
- **Latência de entrega**: `webhook_deliveries.durationMs` (p50/p95) + diferença `createdAt` outbox → delivery (latência fim-a-fim vs. meta de 10s).
- **Tamanho da DLQ**: `COUNT(*)` em `webhook_dead_letter` — alarme operacional natural.

### Tracing

- Correlação por **`eventId`** em todos os logs do ciclo de vida do evento (inserção → tentativa → sucesso/DLQ), no mesmo espírito do `X-Request-Id` que o `src/middlewares/request-logger.middleware.ts` usa para correlacionar requests da API.
- Nos endpoints HTTP do módulo, o `requestId` existente continua funcionando sem mudança; o log de inserção na outbox registra `requestId` + `eventId`, ligando a chamada da API à entrega feita pelo worker.

## 10. Dependências e compatibilidade

| Dependência | Situação |
| --- | --- |
| Node >= 20 (`package.json` engines) | Já atendida — `crypto` nativo para HMAC-SHA256 e `fetch` nativo para o HTTP do worker; **nenhuma dependência npm nova** |
| Express 4.21.1, Zod 3.23.8, Pino 9.5.0, uuid 11.0.3 | Reusados como estão ([ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md)) |
| Prisma 5.22.0 + MySQL | Migration aditiva: 4 tabelas novas + 1 enum; **nenhum model existente é alterado** |
| `package.json` scripts | Novo script `"worker": "tsx watch --env-file=.env src/worker.ts"` (dev) e equivalente de produção, molde dos scripts atuais ([09:11] Larissa: "npm run worker") |
| API `/api/v1` | Mudança 100% aditiva: rotas novas registradas em `src/routes/index.ts`; nenhuma rota existente muda |
| Worker × API | Mesmo repositório, mesma `DATABASE_URL`, `PrismaClient` separado por processo ([09:30] Bruno) |

## 11. Integração com o sistema existente

Seção obrigatória — como o módulo de webhooks se acopla a cada parte real da codebase. Todos os arquivos abaixo **existem hoje no repositório** (os arquivos novos a criar estão na seção 3):

| Arquivo existente | Integração |
| --- | --- |
| `src/modules/orders/order.service.ts` | **Única alteração em código de produção existente.** O `changeStatus` ganha a chamada `await publishWebhookEvent(tx, order, from, to)` dentro do `this.prisma.$transaction`, após o insert em `orderStatusHistory` e antes do retorno — mesma transação, rollback conjunto ([09:40]–[09:41] Bruno/Diego). |
| `src/modules/orders/order.status.ts` | Fonte de verdade das transições. Os valores válidos para a lista `events` do webhook são os do enum `OrderStatus`; a máquina de estados (`canTransition`) já garante que só transições legais geram eventos — o módulo de webhooks não revalida transição. |
| `src/shared/errors/app-error.ts` e `src/shared/errors/http-errors.ts` | Novas classes (`WebhookNotFoundError`, `WebhookInvalidUrlError`, etc.) estendem `AppError`/`NotFoundError`/`ConflictError` com códigos `WEBHOOK_*`, no molde exato de `InsufficientStockError` e `InvalidStatusTransitionError` ([09:28] Bruno). |
| `src/middlewares/error.middleware.ts` | **Zero mudança**: já trata `AppError`, `ZodError` e erros do Prisma — captura os erros do módulo automaticamente ([09:29] Bruno). |
| `src/middlewares/auth.middleware.ts` | `authenticate` protege todas as rotas do módulo; `requireRole('ADMIN')` protege o replay de DLQ ([09:36] Larissa: "reaproveita o requireRole que já existe"). |
| `src/middlewares/validate.middleware.ts` | Rotas do módulo usam `validate({ body, params, query })` com os novos `webhook.schemas.ts` (Zod), incluindo a validação de URL `https` ([09:23] Sofia). |
| `src/shared/logger/index.ts` | Worker e módulo usam o `createLogger()` existente; `redactPaths` ganham `'*.secret'` (seção 9). |
| `src/routes/index.ts` e `src/app.ts` | `buildWebhookRouter` registrado no `buildApiRouter`; `buildControllers` instancia repository → service → controller do módulo, seguindo a cadeia de injeção manual existente. |
| `src/server.ts` | Molde da nova entry-point `src/worker.ts`: bootstrap, logger, graceful shutdown (SIGINT/SIGTERM) e `prisma.$disconnect()` ([09:11] Larissa). |
| `src/config/database.ts` | O worker cria seu `PrismaClient` com o mesmo `createPrismaClient()` — instância própria por processo ([09:30] Bruno). |
| `prisma/schema.prisma` | Ganha os 4 models novos (seção 4) com os mesmos padrões: UUID `@db.Char(36)`, `@@map` snake_case, índices explícitos. |
| `src/shared/http/response.ts` | `GET /webhooks` e `GET /webhooks/:id/deliveries` usam `paginated()` para o envelope de paginação padrão. |

## 12. Critérios de aceite técnicos

1. **Atomicidade:** teste de integração prova que (a) rollback da transação de `changeStatus` não deixa linha na `webhook_outbox`; (b) commit com endpoint interessado sempre deixa exatamente uma linha por endpoint ([09:40] Bruno).
2. **Filtro na inserção:** transição para status que nenhum webhook do customer escuta não insere linha na outbox ([09:34] Bruno).
3. **Latência:** evento inserido é entregue (1ª tentativa) em < 10s com worker ativo ([09:02] Marcos); intervalo de polling configurado em 2s ([09:09] Diego).
4. **Retry:** falha simulada gera reagendamentos exatamente em 1m/5m/30m/2h/12h; na 5ª falha o evento aparece na `webhook_dead_letter` com motivo e timestamp, e some do fluxo ativo da outbox ([09:17] Larissa).
5. **Replay:** `POST .../replay` com role `OPERATOR` → `403`; com `ADMIN` → evento reenfileirado com mesmo `event_id` e log de auditoria com `userId` ([09:36] Sofia).
6. **HMAC:** assinatura do corpo verificável com a secret do endpoint; após rotação, entregas durante o grace period validam com a secret antiga **e** com a nova; após 24h, apenas com a nova ([09:21]–[09:22] Sofia).
7. **Validações:** cadastro com URL `http` → `400 WEBHOOK_INVALID_URL` ([09:23] Sofia); payload > 64KB → `WEBHOOK_PAYLOAD_TOO_LARGE` e rollback ([09:24] Larissa).
8. **Headers:** toda entrega contém `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` ([09:44]).
9. **Snapshot:** alterar o pedido após a inserção do evento não altera o payload entregue ([09:52] Larissa).
10. **Deliveries:** `GET /webhooks/:id/deliveries` retorna as últimas entregas (teto 100) com sucesso/falha, payload, response e `durationMs` ([09:34] Marcos).
11. **Ordering:** com single-worker, eventos do mesmo `order_id` são entregues na ordem das transições ([09:12] Diego).

## 13. Riscos e mitigação

| Risco | Mitigação |
| --- | --- |
| Falha de escrita na `webhook_outbox` bloqueia mudanças de status (acoplamento intencional) | Comportamento desejado para consistência ([09:40] Bruno); insert simples e indexado minimiza chance de falha; monitorar erros de transação. |
| Crescimento da `webhook_outbox` degrada o polling | Índices em `status` e `createdAt` + leitura em batch pequeno ([09:08] Diego); arquivamento planejado para fase futura. |
| Worker parado acumula backlog silenciosamente | Nenhum evento se perde (estado no banco); monitorar `COUNT(status='PENDING')` e idade do evento mais antigo (seção 9). |
| Cliente indisponível > ~15h perde entrega automática | DLQ preserva evidência; replay manual admin ([09:18] Diego); notificação proativa (email) avaliada em fase futura ([09:37] Larissa). |
| Secret vazada pelo cliente | Secret por endpoint (raio de dano limitado) + rotação com grace period 24h ([09:21] Sofia); redação de secrets nos logs (seção 9). |
| Duplicatas confundem cliente que não implementou dedup | Documentação destacada no portal do desenvolvedor ([09:26] Marcos); `X-Event-Id` em toda entrega. |
| Escala futura para multi-worker quebra ordering | Limitação documentada ([09:13] Larissa); caminhos conhecidos: particionamento por `order_id` ou lock pessimista ([09:13] Diego). |
| Implementação incorreta de HMAC/geração de secret | Revisão de segurança dedicada da Sofia (≥ 2 dias úteis) antes do deploy ([09:46] Sofia). |
