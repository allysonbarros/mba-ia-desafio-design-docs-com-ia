# ADR-007 — Snapshot do payload renderizado na inserção da outbox

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:52] Bruno: "Beleza, snapshot. Decidido").

**Decisores:** Larissa (Tech Lead), Bruno (Eng. Pleno — Pedidos), Diego (Eng. Sênior — Plataforma).

## Contexto

A linha da outbox precisa de conteúdo para o worker enviar. Duas modelagens são possíveis: guardar o payload JSON **já renderizado** no momento da inserção, ou guardar apenas o `order_id` e montar o payload **na hora do envio** ([09:51] Bruno levantou a dúvida).

O detalhe que decide: entre a inserção e o envio podem se passar minutos ou horas (ciclo de polling, retries com backoff de até 12h — [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)). Nesse intervalo o pedido pode mudar de novo.

## Decisão

**Payload renderizado e persistido na outbox no momento da inserção**, dentro da mesma transação da mudança de status ([09:52] Larissa; [09:52] Diego: "snapshot na inserção").

O evento reflete o estado do pedido **no instante em que o status mudou**, independentemente de quando for entregue. O formato do payload (JSON com `event_id`, `event_type`, `timestamp`, `order_id`, `order_number`, `from_status`, `to_status`, `customer_id`, `total_cents` — sem `items`) está detalhado no [FDD](../FDD.md) ([09:43] Diego).

## Alternativas Consideradas

### 1. Guardar só `order_id` e renderizar o payload no envio — descartada

- **Trade-off que motivou o descarte:** se o pedido mudar entre a transição e o envio (ou entre retries), o evento refletiria um estado diferente do que disparou a notificação — "senão tem caso esquisito" ([09:52] Larissa). Um evento de `PAID` entregue horas depois descreveria um pedido já `SHIPPED`.

## Consequências

### Positivas

- **Consistência temporal**: o evento descreve exatamente o estado que causou a notificação ([09:52] Larissa).
- Reentregas (retries e replay de DLQ) enviam **byte a byte o mesmo payload**, mantendo a assinatura HMAC estável e a dedup por `X-Event-Id` confiável ([ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md), [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md)).
- O envio não precisa de query adicional para montar o payload — menos carga no ciclo do worker.

### Negativas / trade-offs assumidos

- Payload duplicado em disco para cada evento — mitigado pelo payload enxuto sem `items` ([09:43] Diego) e pelo teto de 64KB por evento ([09:24] Larissa).
- Evolução futura do formato do payload exigirá cuidado com eventos antigos ainda pendentes na outbox/DLQ (versionamento de payload não foi discutido nesta fase).
