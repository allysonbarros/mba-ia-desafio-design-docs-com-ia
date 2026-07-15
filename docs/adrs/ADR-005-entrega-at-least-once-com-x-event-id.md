# ADR-005 — Garantia de entrega at-least-once com deduplicação por X-Event-Id

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:26] Larissa: "At-least-once com X-Event-Id pra dedup do lado do cliente. Decisão").

**Decisores:** Diego (Eng. Sênior — Plataforma), Larissa (Tech Lead), Sofia (Segurança — ponderou o custo para o cliente), Marcos (PM — assumiu a documentação).

## Contexto

Com retry automático ([ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md)), o mesmo evento pode ser entregue mais de uma vez — por exemplo, quando o cliente processa a requisição mas a resposta se perde ou estoura o timeout, e o worker retenta. É preciso definir a semântica de entrega que o sistema garante e como o cliente lida com duplicatas ([09:24] Diego: "Pode acontecer de o cliente receber o mesmo evento duas vezes. Ele tem que estar preparado").

## Decisão

1. O sistema garante **at-least-once**: nenhum evento commitado se perde, mas duplicatas podem ocorrer ([09:24] Diego).
2. Cada evento recebe um **UUID (`event_id`) gerado quando entra na outbox**, único por evento, enviado em toda entrega no header **`X-Event-Id`** ([09:25] Diego).
3. A **deduplicação é responsabilidade do cliente**, usando o `event_id` ([09:25] Diego).
4. Essa responsabilidade será **documentada com destaque no portal do desenvolvedor** ([09:26] Marcos).

## Alternativas Consideradas

### 1. Garantia exactly-once — descartada

- **Trade-off que motivou o descarte:** exactly-once exigiria coordenação dos dois lados (nós e cada cliente) e ficaria muito mais complexo. At-least-once com `event_id` "resolve 99% dos casos" e é o padrão de mercado — Stripe e GitHub fazem assim ([09:25] Diego).

## Consequências

### Positivas

- Semântica simples de implementar e de operar; sem protocolo de coordenação com o cliente ([09:25] Diego).
- Alinhada ao padrão de mercado (Stripe, GitHub), o que reduz atrito de integração para clientes que já consomem webhooks de outros provedores ([09:25] Diego).
- Combinada com o snapshot do payload ([ADR-007](ADR-007-snapshot-do-payload-na-insercao.md)), a reentrega é byte a byte idêntica à original, o que torna a dedup pelo `event_id` confiável.

### Negativas / trade-offs assumidos

- **Joga responsabilidade para o cliente** ([09:25] Sofia) — cliente que não deduplicar verá efeitos duplicados; mitigado com documentação destacada no portal ([09:26] Marcos).
- Duplicatas são esperadas e legítimas: dashboards e histórico de entregas precisam tratá-las como comportamento normal, não como bug.
