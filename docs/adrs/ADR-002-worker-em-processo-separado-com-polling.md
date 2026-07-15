# ADR-002 — Worker em processo separado com polling de 2 segundos

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:10] Larissa: "Vamos registrar isso como uma decisão. Worker em polling, 2s").

**Decisores:** Larissa (Tech Lead), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Marcos (PM — validou a latência).

## Contexto

Com o padrão Outbox adotado ([ADR-001](ADR-001-padrao-outbox-no-mysql.md)), é preciso definir **como** os eventos pendentes na tabela `webhook_outbox` são consumidos e entregues.

Restrições levantadas na reunião:

- O requisito de negócio é "tempo real" na percepção do cliente: **qualquer coisa abaixo de 10 segundos** ([09:02] Marcos).
- MySQL não tem mecanismo de notificação a processos externos como o `NOTIFY/LISTEN` do Postgres; triggers só executam SQL ([09:09] Diego).
- Se o consumo rodar dentro da instância da API, um restart da API mata o worker ([09:11] Diego).

## Decisão

1. **Polling em loop**: a cada **2 segundos**, o worker busca os eventos pendentes mais antigos (ordenados por `created_at`), processa em batch pequeno e marca como entregues ([09:09] Diego; [09:08] Diego).
2. **Processo separado da API**: nova entry-point `src/worker.ts` no mesmo projeto, espelhando o padrão de `src/server.ts`, com script `npm run worker` ([09:11] Larissa).
3. **Mesma stack, mesmo banco**: o worker conecta na mesma `DATABASE_URL`, mas abre **instância própria de `PrismaClient`** — o client é por processo ([09:30] Bruno; [09:11] Diego).
4. **Single-worker por enquanto**: com um único worker processando em ordem de `created_at`, a entrega mantém ordem por `order_id`. Escalar para múltiplos workers quebra essa garantia e fica como problema futuro (particionamento por `order_id` ou lock pessimista) ([09:12]–[09:13] Diego; [09:13] Larissa: "Documentamos como limitação conhecida").

## Alternativas Consideradas

### 1. Trigger de banco para notificar o worker reativamente — descartada

- **Trade-off que motivou o descarte:** trigger MySQL não notifica processo externo, só executa SQL; para avisar o worker seria preciso improvisar (escrever em arquivo, bater em endpoint), o que "fica esquisito". Polling de 2s atende o requisito de <10s com folga ([09:09] Diego).

### 2. Worker embutido no mesmo processo da API — descartada

- **Trade-off que motivou o descarte:** restart ou crash da API derrubaria o consumo de eventos junto ([09:11] Diego: "Senão se a API reinicia, perde o worker").

## Consequências

### Positivas

- Latência de pior caso de ~2s no ciclo de polling, bem abaixo do teto de 10s aceito pelo negócio ([09:10] Marcos: "2 segundos serve, perfeito").
- Isolamento de falhas: API e worker sobem, caem e fazem deploy de forma independente ([09:11] Diego).
- Reuso total da stack (Node, TypeScript, Prisma, mesmo repositório) — sem infra nova.

### Negativas / trade-offs assumidos

- **Latência mínima de até 2 segundos** no pior caso, aceita explicitamente ([09:10] Larissa).
- **Ordering global não garantido**: garantia de ordem só por `order_id` e somente enquanto houver um único worker — registrado como limitação conhecida ([09:13] Larissa). Os clientes não pediram ordering global ([09:14] Marcos).
- Polling gera consultas constantes ao banco mesmo sem eventos pendentes (custo aceito pela simplicidade).
- Um processo a mais para operar e monitorar em produção.
