# PRD — Sistema de Webhooks de Notificação de Pedidos

> **Documentos relacionados:** [RFC](RFC.md) (proposta técnica) · [FDD](FDD.md) (especificação de implementação) · [ADRs](adrs/README.md) (decisões) · [Tracker](TRACKER.md) (rastreabilidade)

## 1. Resumo e contexto da feature

O Order Management System (OMS) vai ganhar um sistema de **webhooks outbound** que notifica clientes B2B, via HTTP, sempre que o status de um pedido deles muda. Hoje a plataforma não possui nenhum mecanismo de notificação externa — sem eventos, filas ou webhooks — e a integração dos clientes depende de polling no `GET /orders`.

A feature nasceu de um pedido formal de três clientes B2B — **Atlas Comercial, MaxDistribuição e Nova Cargo** — recebido na semana passada ([09:00] Marcos), e foi desenhada em reunião técnica com Tech Lead, PM, engenharia e segurança ([`TRANSCRICAO.md`](../TRANSCRICAO.md)).

## 2. Problema e motivação

- Os clientes ficam "batendo no `GET /orders` de tempos em tempos pra ver se mudou alguma coisa", o que deixa a integração deles **lenta e cara** ([09:00] Marcos).
- Para o cliente, "tempo real" significa **atualização em menos de 10 segundos**, sem precisar atualizar manualmente ([09:02] Marcos).
- **Risco comercial concreto:** a Atlas sinalizou que pode **migrar para o concorrente** se a funcionalidade não for entregue até o fim do trimestre ([09:00] Marcos). A expectativa de prazo comunicada é **fim de novembro** ([09:45] Marcos).

## 3. Público-alvo e cenários de uso

**Público-alvo:** clientes B2B integrados via API (inicialmente Atlas Comercial, MaxDistribuição e Nova Cargo), que consomem o OMS por sistemas próprios. Eles **recebem** notificações — o fluxo é exclusivamente outbound ([09:02] Marcos). O cadastro é feito pela nossa API, autenticado com JWT do nosso sistema, por usuários que representam o cliente ([09:32] Marcos).

**Cenários de uso:**

1. **Integração reativa:** a Atlas cadastra um webhook para os status `SHIPPED` e `DELIVERED`; quando um pedido é despachado, o sistema dela é notificado em segundos e dispara o fluxo logístico interno — sem polling ([09:33] Marcos).
2. **Vários endpoints por cliente:** um cliente com mais de um sistema cadastra múltiplos webhooks e identifica qual cadastro recebeu cada envio pelo header `X-Webhook-Id` ([09:44] Sofia).
3. **Diagnóstico de integração:** o cliente consulta o histórico das últimas entregas ("sucesso/falha, payload, response, tempo de resposta") para depurar o lado dele ([09:34] Marcos).
4. **Recuperação operacional:** após uma indisponibilidade prolongada do cliente, um admin da plataforma reprocessa manualmente os eventos que foram para a fila de falhas permanentes ([09:18] Diego).
5. **Higiene de segurança:** o cliente rotaciona a secret do webhook pela API (ex.: após suspeita de vazamento) e migra seus sistemas dentro da janela de 24h ([09:21] Sofia).

## 4. Objetivos e métricas de sucesso

| # | Objetivo | Métrica | Meta |
| --- | --- | --- | --- |
| O1 | Notificar mudanças de status em "tempo real" na percepção do cliente | Latência entre o commit da mudança de status e a primeira tentativa de entrega | **< 10 segundos** ([09:02] Marcos); piso técnico de ~2s aceito pelo polling ([09:10] Larissa) |
| O2 | Nenhuma mudança de status sem evento correspondente | Eventos registrados vs. transições de status commitadas com webhook interessado | **100%** — garantia transacional ([09:40] Bruno) |
| O3 | Sobreviver a indisponibilidades reais dos clientes | Janela coberta pela política de reentrega automática | **~15 horas** entre primeira falha e última tentativa ([09:17] Diego) |
| O4 | Reter os clientes B2B solicitantes | Entrega da feature dentro do prazo comercial | **Fim de novembro / 3 sprints**, incluindo revisão de segurança ([09:45]–[09:47]) |
| O5 | Eliminar o polling dos clientes integrados | Adoção de webhooks pelos 3 clientes solicitantes após o lançamento | 3 de 3 clientes com webhook ativo (consequência direta do pedido formal — [09:00] Marcos) |

## 5. Escopo

### Incluso

- Cadastro, edição, remoção e listagem de webhooks por cliente, com escolha dos status que cada endpoint quer ouvir ([09:31]–[09:33]).
- Notificação HTTP assinada (HMAC-SHA256) a cada mudança de status de pedido relevante, com payload enxuto e headers de identificação/dedup ([09:43]–[09:44]).
- Reentrega automática com backoff em caso de falha e fila de falhas permanentes (DLQ) com reprocessamento manual por admin ([09:14]–[09:18]).
- Rotação de secret pela API com período de convivência de 24h ([09:21]).
- Histórico de entregas consultável pelo cliente ([09:34]).

### Fora de escopo (descartado ou adiado na reunião)

| Item | Situação | Origem |
| --- | --- | --- |
| **Notificação por email** quando o webhook do cliente falha repetidamente | Adiado — "Email tá fora de escopo dessa fase. Talvez próxima fase, depois que a gente medir o impacto" | [09:37] Larissa |
| **Dashboard/painel visual** para o cliente acompanhar webhooks | Descartado nesta feature — "Só endpoints. Painel é projeto separado do time de frontend" | [09:40] Larissa |
| **Rate limiting de envio** para clientes com alto volume | Adiado — "a gente observa e implementa se virar problema" | [09:39] Diego |
| **Webhooks inbound** (cliente enviando eventos para a plataforma) | Fora de escopo — "Só saindo da gente pra eles" | [09:02] Marcos |
| **Arquivamento** de eventos entregues (~30 dias) | Fora do escopo desta feature | [09:08] Diego |
| **Escala para múltiplos workers** | "Problema do futuro" | [09:13] Diego |

## 6. Requisitos funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RF-01 | Cadastrar webhook via `POST`, informando `url` e lista de status desejados; a **secret é gerada pelo sistema e devolvida na criação**; `customer_id` informado no body/path (não vem do JWT) | [09:31] Marcos; [09:32] Larissa |
| RF-02 | Editar webhook via `PATCH` (url, eventos, estado ativo) | [09:33] Bruno |
| RF-03 | Remover webhook via `DELETE` | [09:33] Bruno |
| RF-04 | Listar os webhooks de um customer via `GET` | [09:33] Bruno |
| RF-05 | Filtrar eventos por webhook: cada endpoint escolhe a lista de status que quer ouvir; o filtro é aplicado **na inserção na outbox** (se ninguém escuta, não insere) | [09:33] Marcos; [09:34] Bruno |
| RF-06 | Consultar histórico de entregas (`GET /webhooks/:id/deliveries`): últimas 100, com sucesso/falha, payload, response e tempo de resposta | [09:34] Marcos |
| RF-07 | Reprocessar item da DLQ via endpoint admin (`POST /admin/webhooks/dead-letter/:id/replay`), restrito à role `ADMIN`, com log de auditoria de quem executou | [09:18] Diego; [09:35]–[09:36] Sofia/Larissa |
| RF-08 | Rotacionar secret via API, mantendo a secret antiga válida por 24h em paralelo | [09:21] Sofia |
| RF-09 | Registrar o evento de webhook na **mesma transação** da mudança de status do pedido (rollback conjunto) | [09:40] Bruno |
| RF-10 | Enviar em cada entrega os headers `X-Event-Id`, `X-Signature`, `X-Timestamp`, `X-Webhook-Id` e `Content-Type: application/json` | [09:44] Diego/Sofia |
| RF-11 | Payload JSON enxuto com `event_id`, `event_type` (`order.status_changed`), `timestamp` ISO 8601, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id` e `total_cents` — **sem `items`** (detalhes via `GET /orders/:id`) | [09:43] Diego |
| RF-12 | Recusar cadastro de URL não-`https` com erro de validação | [09:23] Sofia |
| RF-13 | CRUD de configuração acessível a qualquer role autenticada; apenas o replay de DLQ exige `ADMIN` | [09:36]–[09:37] Sofia/Marcos |

## 7. Requisitos não funcionais

| ID | Requisito | Origem |
| --- | --- | --- |
| RNF-01 | **Latência:** entrega iniciada em < 10s após a mudança de status; latência mínima de ~2s aceita (ciclo de polling) | [09:02] Marcos; [09:10] Larissa |
| RNF-02 | **Confiabilidade:** garantia at-least-once; duplicatas possíveis, dedup pelo cliente via `X-Event-Id` | [09:24]–[09:26] |
| RNF-03 | **Resiliência:** 5 tentativas com backoff 1m/5m/30m/2h/12h; timeout de 10s por chamada; falha permanente vai à DLQ | [09:17] Larissa; [09:42] Diego |
| RNF-04 | **Segurança:** assinatura HMAC-SHA256 do corpo, secret única por endpoint, TLS obrigatório | [09:20]–[09:23] Sofia |
| RNF-05 | **Limite de payload:** 64KB; acima disso o evento falha com erro (não trunca) | [09:23]–[09:24] |
| RNF-06 | **Desempenho do fluxo crítico:** a notificação não pode travar a transação de mudança de status (nada de HTTP síncrono) | [09:04] Bruno |
| RNF-07 | **Ordering:** garantida apenas por `order_id` e enquanto houver um único worker — limitação conhecida e documentada; ordering global não é requisito dos clientes | [09:13] Larissa; [09:14] Marcos |
| RNF-08 | **Disponibilidade do consumo:** worker roda em processo separado da API; restart da API não interrompe entregas | [09:11] Diego |
| RNF-09 | **Auditabilidade:** replays de DLQ registram quem executou | [09:36] Sofia |

## 8. Decisões e trade-offs principais

Resumo em altura de produto — o racional completo está nos [ADRs](adrs/README.md):

| Decisão | Trade-off aceito | ADR |
| --- | --- | --- |
| Outbox no MySQL existente (sem fila externa) | Latência atrelada a polling e crescimento de tabela, em troca de consistência total e zero infra nova | [ADR-001](adrs/ADR-001-padrao-outbox-no-mysql.md) |
| Worker separado, polling de 2s | Piso de ~2s de latência, em troca de simplicidade e isolamento de falhas | [ADR-002](adrs/ADR-002-worker-em-processo-separado-com-polling.md) |
| 5 tentativas / ~15h de janela + DLQ | Cliente fora por >15h depende de replay manual | [ADR-003](adrs/ADR-003-retry-com-backoff-exponencial-e-dlq.md) |
| HMAC-SHA256, secret por endpoint, rotação 24h | Complexidade de duas secrets ativas, em troca de raio de dano limitado | [ADR-004](adrs/ADR-004-hmac-sha256-com-secret-por-endpoint.md) |
| At-least-once com `X-Event-Id` | Dedup vira responsabilidade do cliente (padrão de mercado) | [ADR-005](adrs/ADR-005-entrega-at-least-once-com-x-event-id.md) |
| Reuso integral dos padrões da codebase | Acoplamento à stack atual, em troca de velocidade e consistência | [ADR-006](adrs/ADR-006-reuso-dos-padroes-existentes-do-projeto.md) |
| Payload snapshot na inserção | Duplicação de dados em disco, em troca de consistência temporal do evento | [ADR-007](adrs/ADR-007-snapshot-do-payload-na-insercao.md) |

## 9. Dependências

- **Código existente:** transação de mudança de status (`src/modules/orders/order.service.ts`), máquina de estados (`src/modules/orders/order.status.ts`), autenticação/roles (`src/middlewares/auth.middleware.ts`), padrão de erros e middleware (`src/shared/errors/`, `src/middlewares/error.middleware.ts`), logger Pino (`src/shared/logger/index.ts`) e MySQL via Prisma (`prisma/schema.prisma`).
- **Time de segurança:** revisão da Sofia (HMAC e geração de secret) antes do deploy — **mínimo 2 dias úteis reservados** ([09:46] Sofia).
- **Produto/documentação:** Marcos documenta no portal do desenvolvedor a integração via API e, com destaque, a responsabilidade de dedup por `X-Event-Id` ([09:26], [09:40] Marcos).
- **Clientes:** implementação da verificação HMAC e da deduplicação no lado deles.

## 10. Riscos e mitigação

| Risco | Probabilidade | Impacto | Mitigação |
| --- | --- | --- | --- |
| **Churn da Atlas** se o prazo (fim de novembro) não for cumprido | Média | Alto — perda de cliente para concorrente ([09:00] Marcos) | Escopo enxuto sobre stack existente; estimativa de 3 sprints validada na reunião com revisão de segurança incluída ([09:45]–[09:47]); PM confirma prazo com o cliente ([09:47] Marcos) |
| **Indisponibilidade prolongada do cliente** (> ~15h) esgota os retries | Baixa | Médio — eventos param na DLQ e exigem ação manual | Janela de ~15h cobre os casos reais observados (~2h de manutenção — [09:16] Diego); DLQ preserva evidência e replay admin recupera ([09:18] Diego) |
| **Vazamento de secret pelo cliente** (já ocorreu — [09:22] Diego) | Média | Médio — terceiros podem forjar notificações para aquele endpoint | Secret única por endpoint limita o raio de dano ([09:21] Sofia); rotação via API com grace period de 24h; secrets redigidas nos logs |
| **Acúmulo de eventos na outbox** degrada o worker | Baixa | Médio — atraso nas notificações | Índices em status e `created_at` + leitura em batch pequeno ([09:08] Diego); backlog monitorável; arquivamento em fase futura |
| **Cliente sem dedup** processa eventos duplicados | Média | Baixo — efeitos duplicados no sistema do cliente | Documentação destacada no portal do desenvolvedor ([09:26] Marcos); `X-Event-Id` em toda entrega ([09:25] Diego) |
| **Falha de segurança na implementação de HMAC/secret** | Baixa | Alto — quebra da confiança na autenticidade | Revisão dedicada de segurança antes do deploy, com ≥ 2 dias úteis ([09:46] Sofia) |

## 11. Critérios de aceitação

1. Cliente cadastra webhook `https` com lista de status e recebe a secret na resposta de criação; cadastro `http` é recusado (RF-01, RF-12).
2. Mudança de status commitada com webhook interessado sempre gera notificação; rollback nunca gera (RF-09, O2).
3. Notificação chega ao endpoint do cliente em menos de 10 segundos no caminho feliz (O1).
4. Cliente que escuta apenas `SHIPPED`/`DELIVERED` não recebe eventos de outros status (RF-05).
5. Com o endpoint do cliente fora do ar, o sistema retenta 5 vezes no cronograma 1m/5m/30m/2h/12h e então move o evento para a DLQ (RNF-03).
6. Admin reprocessa item da DLQ com sucesso; usuário sem role `ADMIN` recebe `403`; a ação fica registrada com o autor (RF-07, RNF-09).
7. Toda entrega carrega os 5 headers definidos e payload no formato acordado, verificável via HMAC-SHA256 (RF-10, RF-11, RNF-04).
8. Rotação de secret mantém as entregas verificáveis pela secret antiga por 24h; depois disso, só pela nova (RF-08).
9. `GET /webhooks/:id/deliveries` mostra as últimas entregas com sucesso/falha, payload, response e tempo de resposta (RF-06).
10. Reentregas do mesmo evento carregam o mesmo `X-Event-Id`, permitindo dedup no cliente (RNF-02).

## 12. Estratégia de testes e validação

Alinhada à estrutura de testes existente (`tests/` com Vitest + Supertest, factories em `tests/helpers/factories.ts`):

- **Testes de integração da API** (padrão de `tests/orders.test.ts`): CRUD de webhooks, validação de URL `https`, geração/rotação de secret, autorização do replay (`ADMIN` vs. `OPERATOR`), histórico de deliveries.
- **Testes transacionais:** provas de atomicidade — rollback da mudança de status não deixa evento na outbox; commit deixa exatamente os eventos filtrados (base do critério 2).
- **Testes do worker:** com endpoint fake (servidor HTTP de teste): entrega com headers e assinatura corretos, timeout de 10s, agenda de retry, movimentação para DLQ na 5ª falha, replay.
- **Testes de segurança dirigidos + revisão manual:** verificação HMAC (inclusive durante grace period de rotação) e geração de secret, seguidos da revisão formal da Sofia com ≥ 2 dias úteis antes do deploy ([09:46] Sofia).
- **Validação ponta a ponta:** integração no `order.service` e testes end-to-end previstos na estimativa de sprints da Larissa ([09:46] Larissa: "Integração no order.service e testes ponta a ponta").
- **Validação com clientes:** Marcos comunica os clientes e publica a documentação no portal ([09:47]–[09:49] Marcos); acompanhamento da adoção pelos 3 solicitantes (O5).
