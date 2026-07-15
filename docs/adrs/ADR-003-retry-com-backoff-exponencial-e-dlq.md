# ADR-003 — Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:17] Larissa: "Decidido: 5 tentativas, backoff 1m/5m/30m/2h/12h").

**Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Sofia (Segurança — requisito de auditoria no replay).

## Contexto

Endpoints de clientes ficam indisponíveis na prática — houve caso real de cliente com **duas horas de indisponibilidade em manutenção planejada** ([09:16] Diego). É preciso definir o que acontece quando uma entrega de webhook falha: quantas vezes tentar de novo, com que espaçamento, e qual o destino final de um evento que nunca conseguiu ser entregue.

Também conta como falha a chamada que estoura o **timeout de 10 segundos** definido para o HTTP do worker ([09:42] Diego).

## Decisão

1. **Retry com backoff exponencial, 5 tentativas no total**, com progressão **1min / 5min / 30min / 2h / 12h** — janela total de quase 15 horas entre a primeira falha e a última tentativa ([09:15]–[09:17] Diego).
2. **Após esgotar as tentativas, o evento vai para uma tabela separada `webhook_dead_letter`** (DLQ), com payload, motivo da falha e timestamp ([09:17] Diego).
3. **Reprocessamento manual via endpoint admin** `POST /admin/webhooks/dead-letter/:id/replay`, que recoloca o evento na outbox como pendente ([09:18] Diego).
4. O replay **exige role `ADMIN`** no JWT e **registra em log quem executou**, para auditoria ([09:36] Sofia; [09:36] Larissa) — reusa o `requireRole` existente em `src/middlewares/auth.middleware.ts`.

## Alternativas Consideradas

### 1. Três tentativas (mais agressivo) — descartada

- **Trade-off que motivou o descarte:** três tentativas em ~30 minutos matariam eventos durante indisponibilidades reais de clientes, como a manutenção planejada de 2 horas já observada ([09:16] Bruno propôs; [09:16] Diego refutou com o caso real).

### 2. Retry indefinido com backoff — descartada

- **Trade-off que motivou o descarte:** eventos ficariam pendurados para sempre se o cliente sumiu de vez ([09:15] Diego: "isso traz o problema de evento ficar pendurado pra sempre").

### 3. Marcar como `failed` na própria outbox em vez de tabela separada — descartada

- **Trade-off que motivou o descarte:** poluiria a leitura da outbox principal. A tabela separada deixa a outbox limpa e serve de evidência para debug e reprocessamento ([09:17] Larissa levantou; [09:18] Diego).

## Consequências

### Positivas

- Janela de ~15h cobre com folga os cenários reais de indisponibilidade observados (2h) ([09:16] Diego; [09:17] Marcos: "Se um cliente meu cair por 15 horas, ele já tá com problema sério dele").
- DLQ preserva payload e motivo da falha como evidência para depuração ([09:18] Diego).
- Caminho de recuperação definido (replay admin auditado) — nenhuma falha é terminal sem intervenção possível.

### Negativas / trade-offs assumidos

- Cliente indisponível por mais de ~15h **perde a entrega automática** e depende de replay manual.
- Replay é manual e exige um admin — sem reprocessamento automático em lote nesta fase.
- Notificação proativa ao cliente sobre falhas (ex.: email após 3 falhas seguidas) ficou **explicitamente fora de escopo** desta fase ([09:37] Larissa).
