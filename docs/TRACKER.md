# Tracker de Rastreabilidade

Mapeamento de cada item registrado nos documentos ([PRD](PRD.md), [RFC](RFC.md), [FDD](FDD.md), [ADRs](adrs/README.md)) à sua origem na transcrição ([`TRANSCRICAO.md`](../TRANSCRICAO.md)) ou no código fonte da aplicação.

**Convenções:**

- **Fonte = TRANSCRICAO** → Localização no formato `[hh:mm] Nome` (fala na transcrição).
- **Fonte = CODIGO** → Localização é o caminho real do arquivo no repositório.
- Itens marcados como *derivado* no conteúdo indicam detalhe de engenharia deduzido diretamente de um item rastreável (a linha aponta para a origem que o motivou).

| ID | Documento | Tipo | Conteúdo (resumo) | Fonte | Localização |
| --- | --- | --- | --- | --- | --- |
| PRD-CTX-01 | docs/PRD.md | Contexto | Pedido formal de 3 clientes B2B (Atlas, MaxDistribuição, Nova Cargo) por notificação em tempo real | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-02 | docs/PRD.md | Problema | Clientes fazem polling no GET /orders; integração lenta e cara | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-03 | docs/PRD.md | Restrição | Atlas pode migrar para concorrente se não entregar até fim do trimestre | TRANSCRICAO | [09:00] Marcos |
| PRD-CTX-04 | docs/PRD.md | Contexto | Aplicação não possui nenhum mecanismo de notificação externa (módulos existentes: auth, users, customers, products, orders) | CODIGO | src/routes/index.ts |
| PRD-OBJ-01 | docs/PRD.md | Objetivo/Métrica | Latência de notificação < 10 segundos ("tempo real" na percepção do cliente) | TRANSCRICAO | [09:02] Marcos |
| PRD-OBJ-02 | docs/PRD.md | Objetivo/Métrica | 100% das mudanças de status commitadas geram evento (garantia transacional) | TRANSCRICAO | [09:40] Bruno |
| PRD-OBJ-03 | docs/PRD.md | Objetivo/Métrica | Janela de reentrega automática de ~15h | TRANSCRICAO | [09:17] Diego |
| PRD-OBJ-04 | docs/PRD.md | Objetivo/Métrica | Prazo: fim de novembro / 3 sprints com revisão de segurança | TRANSCRICAO | [09:45] Marcos |
| PRD-OBJ-05 | docs/PRD.md | Objetivo/Métrica | Adoção pelos 3 clientes solicitantes (eliminar polling) | TRANSCRICAO | [09:00] Marcos |
| PRD-ESC-01 | docs/PRD.md | Fora de escopo | Email de alerta ao cliente sobre falhas — adiado para próxima fase | TRANSCRICAO | [09:37] Larissa |
| PRD-ESC-02 | docs/PRD.md | Fora de escopo | Dashboard/painel visual — projeto separado do frontend | TRANSCRICAO | [09:40] Larissa |
| PRD-ESC-03 | docs/PRD.md | Fora de escopo | Rate limiting de envio — "observar e decidir depois" | TRANSCRICAO | [09:39] Diego |
| PRD-ESC-04 | docs/PRD.md | Fora de escopo | Webhooks inbound — fluxo é só outbound | TRANSCRICAO | [09:02] Marcos |
| PRD-ESC-05 | docs/PRD.md | Fora de escopo | Arquivamento de linhas entregues da outbox (~30 dias) | TRANSCRICAO | [09:08] Diego |
| PRD-ESC-06 | docs/PRD.md | Fora de escopo | Escala para múltiplos workers — "problema do futuro" | TRANSCRICAO | [09:13] Diego |
| PRD-RF-01 | docs/PRD.md | Requisito Funcional | POST de cadastro com url + lista de status; secret gerada pelo sistema e devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| PRD-RF-01b | docs/PRD.md | Requisito Funcional | customer_id vai no body/path, não vem do JWT (JWT é do usuário operador) | TRANSCRICAO | [09:32] Larissa |
| PRD-RF-02 | docs/PRD.md | Requisito Funcional | PATCH para editar webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-03 | docs/PRD.md | Requisito Funcional | DELETE para remover webhook | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-04 | docs/PRD.md | Requisito Funcional | GET para listar webhooks de um customer | TRANSCRICAO | [09:33] Bruno |
| PRD-RF-05 | docs/PRD.md | Requisito Funcional | Filtro por lista de status, aplicado na inserção na outbox | TRANSCRICAO | [09:34] Bruno |
| PRD-RF-06 | docs/PRD.md | Requisito Funcional | Histórico de entregas: últimos 100, sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| PRD-RF-07 | docs/PRD.md | Requisito Funcional | Replay de DLQ via endpoint admin com role ADMIN e log de auditoria | TRANSCRICAO | [09:36] Sofia |
| PRD-RF-08 | docs/PRD.md | Requisito Funcional | Rotação de secret via API com grace period de 24h | TRANSCRICAO | [09:21] Sofia |
| PRD-RF-09 | docs/PRD.md | Requisito Funcional | Evento registrado na mesma transação da mudança de status | TRANSCRICAO | [09:40] Bruno |
| PRD-RF-10 | docs/PRD.md | Requisito Funcional | Headers X-Event-Id, X-Signature, X-Timestamp, X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego |
| PRD-RF-10b | docs/PRD.md | Requisito Funcional | X-Webhook-Id para cliente com vários cadastros identificar o endpoint | TRANSCRICAO | [09:44] Sofia |
| PRD-RF-11 | docs/PRD.md | Requisito Funcional | Payload JSON enxuto (event_id, event_type, timestamp, order_id, order_number, from/to_status, customer_id, total_cents), sem items | TRANSCRICAO | [09:43] Diego |
| PRD-RF-12 | docs/PRD.md | Requisito Funcional | URL http recusada com erro de validação (TLS obrigatório) | TRANSCRICAO | [09:23] Sofia |
| PRD-RF-13 | docs/PRD.md | Requisito Funcional | CRUD com qualquer role autenticada; só replay exige ADMIN | TRANSCRICAO | [09:37] Sofia |
| PRD-RNF-01 | docs/PRD.md | Requisito Não Funcional | Latência mínima de ~2s aceita (polling); pior caso 2s | TRANSCRICAO | [09:10] Larissa |
| PRD-RNF-02 | docs/PRD.md | Requisito Não Funcional | Garantia at-least-once; cliente deve estar preparado para duplicatas | TRANSCRICAO | [09:24] Diego |
| PRD-RNF-03 | docs/PRD.md | Requisito Não Funcional | 5 tentativas, backoff 1m/5m/30m/2h/12h | TRANSCRICAO | [09:17] Larissa |
| PRD-RNF-04 | docs/PRD.md | Requisito Não Funcional | HMAC-SHA256 sobre o corpo, secret por endpoint | TRANSCRICAO | [09:22] Sofia |
| PRD-RNF-05 | docs/PRD.md | Requisito Não Funcional | Limite de payload 64KB, erro caso ultrapasse (não trunca) | TRANSCRICAO | [09:24] Larissa |
| PRD-RNF-06 | docs/PRD.md | Requisito Não Funcional | Notificação não pode travar a transação de mudança de status | TRANSCRICAO | [09:04] Bruno |
| PRD-RNF-07 | docs/PRD.md | Requisito Não Funcional | Ordering só por order_id e só com single-worker — limitação conhecida | TRANSCRICAO | [09:13] Larissa |
| PRD-RNF-07b | docs/PRD.md | Requisito Não Funcional | Clientes não pediram ordering global | TRANSCRICAO | [09:14] Marcos |
| PRD-RNF-08 | docs/PRD.md | Requisito Não Funcional | Worker em processo separado; restart da API não derruba entregas | TRANSCRICAO | [09:11] Diego |
| PRD-RNF-09 | docs/PRD.md | Requisito Não Funcional | Auditoria: log de quem executou o replay | TRANSCRICAO | [09:36] Sofia |
| PRD-DEP-01 | docs/PRD.md | Dependência | Revisão de segurança da Sofia (HMAC/secret), mínimo 2 dias úteis antes do deploy | TRANSCRICAO | [09:46] Sofia |
| PRD-DEP-02 | docs/PRD.md | Dependência | Documentação no portal do desenvolvedor (dedup destacada) | TRANSCRICAO | [09:26] Marcos |
| PRD-DEP-03 | docs/PRD.md | Dependência | Transação existente de mudança de status como ponto de acoplamento | CODIGO | src/modules/orders/order.service.ts |
| PRD-RISK-01 | docs/PRD.md | Risco | Churn da Atlas se prazo não cumprido | TRANSCRICAO | [09:00] Marcos |
| PRD-RISK-02 | docs/PRD.md | Risco | Indisponibilidade real de cliente já observada (~2h em manutenção planejada) | TRANSCRICAO | [09:16] Diego |
| PRD-RISK-03 | docs/PRD.md | Risco | Cliente já vazou secret em log de aplicação | TRANSCRICAO | [09:22] Diego |
| PRD-RISK-04 | docs/PRD.md | Risco | Acúmulo de eventos na outbox pode degradar o worker | TRANSCRICAO | [09:07] Bruno |
| PRD-RISK-05 | docs/PRD.md | Risco | Dedup é responsabilidade do cliente ("joga responsabilidade pro cliente") | TRANSCRICAO | [09:25] Sofia |
| PRD-RISK-06 | docs/PRD.md | Risco | Falha de implementação em HMAC/geração de secret | TRANSCRICAO | [09:46] Sofia |
| PRD-TEST-01 | docs/PRD.md | Estratégia de testes | Integração no order.service e testes ponta a ponta previstos na estimativa | TRANSCRICAO | [09:46] Larissa |
| PRD-TEST-02 | docs/PRD.md | Estratégia de testes | Estrutura de testes existente (Vitest + Supertest + factories) como base | CODIGO | tests/orders.test.ts |
| RFC-ALT-01 | docs/RFC.md | Alternativa descartada | Disparo HTTP síncrono no changeStatus — trava status, rollback impossível | TRANSCRICAO | [09:04] Bruno |
| RFC-ALT-02 | docs/RFC.md | Alternativa descartada | Redis Streams / fila externa — infra nova, overengineering para time pequeno | TRANSCRICAO | [09:07] Diego |
| RFC-ALT-03 | docs/RFC.md | Alternativa descartada | Trigger de banco para acordar o worker — MySQL não notifica processo externo | TRANSCRICAO | [09:09] Diego |
| RFC-ALT-04 | docs/RFC.md | Alternativa descartada | Exactly-once — coordenação dos dois lados, complexidade excessiva | TRANSCRICAO | [09:25] Diego |
| RFC-ALT-05 | docs/RFC.md | Alternativa descartada | Secret global da plataforma — "se vaza uma, vaza tudo" | TRANSCRICAO | [09:21] Sofia |
| RFC-ALT-06 | docs/RFC.md | Alternativa descartada | Retry em 3 tentativas (agressivo demais) e retry indefinido (evento pendurado) | TRANSCRICAO | [09:16] Diego |
| RFC-QA-01 | docs/RFC.md | Questão em aberto | Rate limiting de saída — observar e implementar se virar problema | TRANSCRICAO | [09:39] Diego |
| RFC-QA-02 | docs/RFC.md | Questão em aberto | Escala multi-worker: particionamento por order_id ou lock pessimista | TRANSCRICAO | [09:13] Diego |
| RFC-QA-03 | docs/RFC.md | Questão em aberto | Arquivamento da outbox após ~30 dias | TRANSCRICAO | [09:08] Diego |
| RFC-QA-04 | docs/RFC.md | Questão em aberto | Endurecimento futuro de roles no CRUD de webhooks | TRANSCRICAO | [09:37] Sofia |
| RFC-QA-05 | docs/RFC.md | Questão em aberto | Email de alerta de falha — próxima fase, após medir impacto | TRANSCRICAO | [09:37] Larissa |
| RFC-IMP-01 | docs/RFC.md | Impacto | Único ponto de mudança em código existente: changeStatus chama publishWebhookEvent | TRANSCRICAO | [09:41] Bruno |
| ADR-001 | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Decisão | Outbox no MySQL, evento na mesma transação da mudança de status | TRANSCRICAO | [09:08] Larissa |
| ADR-001b | docs/adrs/ADR-001-padrao-outbox-no-mysql.md | Contexto | Transação de status já é pesada: orders + history + estoque | CODIGO | src/modules/orders/order.service.ts |
| ADR-002 | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | Worker separado (src/worker.ts, npm run worker), polling 2s, batch pequeno, ordem created_at | TRANSCRICAO | [09:10] Larissa |
| ADR-002b | docs/adrs/ADR-002-worker-em-processo-separado-com-polling.md | Decisão | PrismaClient próprio por processo, mesma DATABASE_URL | TRANSCRICAO | [09:30] Bruno |
| ADR-003 | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | 5 tentativas, backoff 1m/5m/30m/2h/12h, DLQ em tabela separada | TRANSCRICAO | [09:17] Larissa |
| ADR-003b | docs/adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md | Decisão | DLQ com payload, motivo da falha e timestamp; evidência para debug | TRANSCRICAO | [09:18] Diego |
| ADR-004 | docs/adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md | Decisão | HMAC-SHA256 sobre o corpo, secret por endpoint, rotação com grace de 24h | TRANSCRICAO | [09:22] Sofia |
| ADR-005 | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Decisão | At-least-once com X-Event-Id (UUID gerado na entrada da outbox) | TRANSCRICAO | [09:26] Larissa |
| ADR-005b | docs/adrs/ADR-005-entrega-at-least-once-com-x-event-id.md | Contexto | Padrão de mercado: Stripe e GitHub fazem assim | TRANSCRICAO | [09:25] Diego |
| ADR-006 | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Decisão | Reuso máximo: AppError, Pino, error middleware, módulos, Zod, códigos de erro | TRANSCRICAO | [09:30] Larissa |
| ADR-006b | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto | Padrão de erros existente: AppError com errorCode | CODIGO | src/shared/errors/app-error.ts |
| ADR-006c | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto | Classes de erro com códigos: InsufficientStockError, InvalidStatusTransitionError | CODIGO | src/shared/errors/http-errors.ts |
| ADR-006d | docs/adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md | Contexto | UUID como padrão de PK em todos os models | CODIGO | prisma/schema.prisma |
| ADR-007 | docs/adrs/ADR-007-snapshot-do-payload-na-insercao.md | Decisão | Payload renderizado (snapshot) na inserção; reflete estado no momento da transição | TRANSCRICAO | [09:52] Larissa |
| FDD-FLUXO-01 | docs/FDD.md | Fluxo | Inserção na outbox dentro da transação do changeStatus; falha = rollback total | TRANSCRICAO | [09:40] Bruno |
| FDD-FLUXO-01b | docs/FDD.md | Contrato interno | Função publishWebhookEvent(tx, order, fromStatus, toStatus) recebendo o tx client | TRANSCRICAO | [09:41] Bruno |
| FDD-FLUXO-01c | docs/FDD.md | Decisão | Função pura recebendo tx, sem injetar repository inteiro no OrderService | TRANSCRICAO | [09:41] Diego |
| FDD-FLUXO-02 | docs/FDD.md | Fluxo | Worker: a cada 2s busca pendentes mais antigos, processa, marca entregue | TRANSCRICAO | [09:09] Diego |
| FDD-FLUXO-03 | docs/FDD.md | Fluxo | Agenda de retry 1m/5m/30m/2h/12h (~15h total) | TRANSCRICAO | [09:17] Diego |
| FDD-FLUXO-04 | docs/FDD.md | Fluxo | DLQ: após 5ª falha move para webhook_dead_letter; replay recoloca como pendente | TRANSCRICAO | [09:18] Diego |
| FDD-FLUXO-05 | docs/FDD.md | Fluxo | Rotação de secret: antiga válida 24h em paralelo para migração do cliente | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-01 | docs/FDD.md | Contrato | POST /webhooks — cadastro com url, events; secret devolvida na criação | TRANSCRICAO | [09:31] Marcos |
| FDD-CONTRATO-02 | docs/FDD.md | Contrato | GET /webhooks?customerId= — listagem por customer | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-03 | docs/FDD.md | Contrato | PATCH /webhooks/:id — edição | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-04 | docs/FDD.md | Contrato | DELETE /webhooks/:id — remoção | TRANSCRICAO | [09:33] Bruno |
| FDD-CONTRATO-05 | docs/FDD.md | Contrato | POST /webhooks/:id/rotate-secret — nova secret via API | TRANSCRICAO | [09:21] Sofia |
| FDD-CONTRATO-06 | docs/FDD.md | Contrato | GET /webhooks/:id/deliveries — últimos 100 com sucesso/falha, payload, response, tempo | TRANSCRICAO | [09:34] Marcos |
| FDD-CONTRATO-07 | docs/FDD.md | Contrato | POST /admin/webhooks/dead-letter/:id/replay — replay manual admin | TRANSCRICAO | [09:35] Diego |
| FDD-CONTRATO-08 | docs/FDD.md | Contrato | Headers da entrega: X-Event-Id, X-Signature, X-Timestamp (detecção de replay attack), X-Webhook-Id, Content-Type | TRANSCRICAO | [09:44] Diego |
| FDD-CONTRATO-09 | docs/FDD.md | Contrato | Payload: event_id, event_type "order.status_changed", timestamp ISO 8601, order_id, order_number, from/to_status, customer_id, total_cents; sem items (detalhe via GET /orders/:id) | TRANSCRICAO | [09:43] Diego |
| FDD-CONTRATO-10 | docs/FDD.md | Contrato | Autenticação JWT nos endpoints do módulo (API direta, JWT do nosso sistema) | TRANSCRICAO | [09:32] Marcos |
| FDD-CONTRATO-11 | docs/FDD.md | Contrato | Formato do order_number nos exemplos (ORD-000123) | CODIGO | src/modules/orders/order.service.ts |
| FDD-ERRO-01 | docs/FDD.md | Matriz de erros | Códigos WEBHOOK_NOT_FOUND, WEBHOOK_INVALID_URL, WEBHOOK_SECRET_REQUIRED citados como exemplos do padrão | TRANSCRICAO | [09:28] Bruno |
| FDD-ERRO-02 | docs/FDD.md | Matriz de erros | Prefixo WEBHOOK_ para tudo do módulo | TRANSCRICAO | [09:29] Larissa |
| FDD-ERRO-03 | docs/FDD.md | Matriz de erros | WEBHOOK_PAYLOAD_TOO_LARGE: > 64KB erra, não trunca | TRANSCRICAO | [09:23] Sofia |
| FDD-ERRO-04 | docs/FDD.md | Matriz de erros | WEBHOOK_DELIVERY_TIMEOUT: cliente sem resposta em 10s = falha para retry | TRANSCRICAO | [09:42] Diego |
| FDD-ERRO-05 | docs/FDD.md | Matriz de erros | Envelope de erro { error: { code, message, details } } do middleware central | CODIGO | src/middlewares/error.middleware.ts |
| FDD-DADOS-01 | docs/FDD.md | Modelo de dados | webhook_endpoints: url + secret + customer_id + estado ativo | TRANSCRICAO | [09:21] Bruno |
| FDD-DADOS-02 | docs/FDD.md | Modelo de dados | Outbox com status (pendente/processando/falhou/entregue) e índices em status e created_at | TRANSCRICAO | [09:08] Diego |
| FDD-DADOS-03 | docs/FDD.md | Modelo de dados | webhook_deliveries: sucesso/falha, payload, response, tempo de resposta | TRANSCRICAO | [09:34] Marcos |
| FDD-DADOS-04 | docs/FDD.md | Modelo de dados | webhook_dead_letter: payload, motivo da falha, timestamp | TRANSCRICAO | [09:18] Diego |
| FDD-DADOS-05 | docs/FDD.md | Modelo de dados | UUID como PK das novas tabelas, padrão do projeto | TRANSCRICAO | [09:51] Larissa |
| FDD-INT-01 | docs/FDD.md | Integração | changeStatus: única alteração em código existente — insere na outbox dentro do $transaction | CODIGO | src/modules/orders/order.service.ts |
| FDD-INT-02 | docs/FDD.md | Integração | Máquina de estados como fonte dos status válidos para o filtro de eventos | CODIGO | src/modules/orders/order.status.ts |
| FDD-INT-03 | docs/FDD.md | Integração | Erros WEBHOOK_* estendem AppError no molde das classes existentes | CODIGO | src/shared/errors/http-errors.ts |
| FDD-INT-04 | docs/FDD.md | Integração | Error middleware já trata AppError/Zod/Prisma — zero mudança | CODIGO | src/middlewares/error.middleware.ts |
| FDD-INT-05 | docs/FDD.md | Integração | authenticate em todas as rotas; requireRole('ADMIN') no replay | CODIGO | src/middlewares/auth.middleware.ts |
| FDD-INT-06 | docs/FDD.md | Integração | validate({body, params, query}) com schemas Zod do módulo | CODIGO | src/middlewares/validate.middleware.ts |
| FDD-INT-07 | docs/FDD.md | Integração | Logger Pino compartilhado; redactPaths ganham '*.secret' | CODIGO | src/shared/logger/index.ts |
| FDD-INT-08 | docs/FDD.md | Integração | src/worker.ts segue o molde de bootstrap/graceful shutdown do server | CODIGO | src/server.ts |
| FDD-INT-09 | docs/FDD.md | Integração | Worker cria PrismaClient próprio pelo padrão createPrismaClient() | CODIGO | src/config/database.ts |
| FDD-INT-10 | docs/FDD.md | Integração | Rotas registradas no buildApiRouter; controllers na cadeia de injeção manual | CODIGO | src/routes/index.ts |
| FDD-INT-11 | docs/FDD.md | Integração | Envelope de paginação paginated() nos GETs de listagem | CODIGO | src/shared/http/response.ts |
| FDD-INT-12 | docs/FDD.md | Integração | Estrutura de módulo (controller/service/repository/routes/schemas) em src/modules/webhooks | TRANSCRICAO | [09:27] Bruno |
| FDD-INT-13 | docs/FDD.md | Integração | Lógica de processamento em arquivo do módulo (webhook.processor.ts) | TRANSCRICAO | [09:28] Bruno |
| FDD-OBS-01 | docs/FDD.md | Observabilidade | Logger Pino já no projeto inteiro; nada novo de logging | TRANSCRICAO | [09:29] Bruno |
| FDD-OBS-02 | docs/FDD.md | Observabilidade | Log de auditoria com quem executou o replay | TRANSCRICAO | [09:36] Sofia |
| FDD-OBS-03 | docs/FDD.md | Observabilidade | Correlação por request id no padrão existente (X-Request-Id), estendida com eventId | CODIGO | src/middlewares/request-logger.middleware.ts |
| FDD-OBS-04 | docs/FDD.md | Observabilidade | Métricas derivadas das tabelas (backlog, taxa de sucesso, durationMs, tamanho da DLQ) sem stack nova — coerente com decisão de não subir infra | TRANSCRICAO | [09:07] Diego |
| FDD-RES-01 | docs/FDD.md | Resiliência | Timeout de 10s por chamada HTTP do worker | TRANSCRICAO | [09:42] Diego |
| FDD-RES-02 | docs/FDD.md | Resiliência | Semântica do outbox: commit = evento registrado; rollback = evento some | TRANSCRICAO | [09:06] Diego |
| FDD-RES-03 | docs/FDD.md | Resiliência | Worker separado: restart da API não perde o worker | TRANSCRICAO | [09:11] Diego |
| FDD-DEP-01 | docs/FDD.md | Dependência | Node >= 20, Express/Zod/Pino/Prisma/uuid já presentes; sem dependência npm nova | CODIGO | package.json |
| FDD-DEP-02 | docs/FDD.md | Dependência | Mesmo banco, mesma stack; só não pode ser o mesmo processo | TRANSCRICAO | [09:11] Diego |

## Resumo de cobertura

| Indicador | Valor |
| --- | --- |
| Total de linhas | 127 |
| Fonte = TRANSCRICAO | 105 (~83%) |
| Fonte = CODIGO | 22 (~17%) |

Itens dos documentos que são **derivações de engenharia** (ex.: campos exatos das tabelas, códigos de erro complementares como `WEBHOOK_INVALID_EVENTS`/`WEBHOOK_DISABLED`, formato das duas assinaturas durante o grace period) estão anotados como *derivado* no próprio documento e apontam aqui para a fala/arquivo que os motivou — nenhum item foi criado sem origem identificável.
