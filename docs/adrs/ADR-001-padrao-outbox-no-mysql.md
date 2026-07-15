# ADR-001 — Padrão Outbox no MySQL para publicação de eventos de webhook

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:08] Larissa: "Tá decidido então: outbox em MySQL").

**Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos).

## Contexto

O OMS precisa notificar clientes B2B via HTTP quando o status de um pedido muda. A mudança de status hoje acontece dentro de uma transação Prisma já pesada em `src/modules/orders/order.service.ts` (método `changeStatus`): atualiza `orders`, insere em `order_status_history` e ajusta `stock_quantity` dos produtos ([09:04] Bruno).

Dois problemas impedem o disparo do HTTP dentro dessa transação:

1. **Latência acoplada:** um endpoint de cliente lento travaria mudanças de status de outros pedidos ([09:04] Bruno).
2. **Consistência:** se o cliente estiver fora do ar, não faz sentido dar rollback na mudança de status por causa da notificação ([09:04] Bruno).

Ao mesmo tempo, a notificação não pode se perder: "não pode ter caso de status mudar e evento não sair" ([09:40] Bruno).

## Decisão

Adotar o **padrão Outbox transacional sobre o MySQL já existente**:

- Na mesma transação SQL que atualiza `orders` e `order_status_history`, inserir uma linha na nova tabela `webhook_outbox` com o evento ([09:06] Diego).
- Um worker separado lê essa tabela e dispara as chamadas HTTP (detalhado no [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- A tabela tem índice no campo de status do evento (`pendente`, `processando`, `falhou`, `entregue`) e em `created_at`; o worker lê apenas pendentes em batch pequeno ([09:08] Diego).

A semântica é: **se a transação principal commitou, o evento foi registrado; se deu rollback, o evento some junto** ([09:06] Diego).

## Alternativas Consideradas

### 1. Disparo HTTP síncrono dentro do `OrderService.changeStatus` — descartada

- **Trade-off que motivou o descarte:** cliente lento trava a mudança de status para outros pedidos, e indisponibilidade do cliente exigiria rollback de uma transação de negócio válida ([09:03]–[09:04] Bruno; [09:06] Diego: "Síncrono está fora de questão").

### 2. Fila externa (Redis Streams ou similar) — descartada

- **Trade-off que motivou o descarte:** exigiria subir e operar infraestrutura nova. "A gente é um time pequeno. Subir Redis Cluster pra isso é overengineering. Outbox no MySQL existente resolve" ([09:07] Diego, [09:07] Larissa).

## Consequências

### Positivas

- **Atomicidade garantida** entre mudança de status e registro do evento — sem inconsistência possível ([09:06] Diego).
- **Zero infraestrutura nova:** reusa o MySQL e o Prisma já operados pelo time ([09:07] Diego).
- Índices em status e `created_at` mantêm a leitura do worker eficiente mesmo com acúmulo ([09:08] Diego).

### Negativas / trade-offs assumidos

- A tabela `webhook_outbox` cresce indefinidamente; o arquivamento de linhas entregues (~30 dias) foi deixado **fora do escopo desta feature** ([09:08] Diego).
- A latência de entrega fica atrelada ao ciclo de polling do worker (mínimo ~2s no pior caso — ver [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md)).
- O MySQL passa a carregar também a carga de fila; se o volume de eventos crescer muito, será preciso revisitar (aceito conscientemente pelo tamanho atual do time e do volume).
