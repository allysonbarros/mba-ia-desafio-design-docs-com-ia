# Architectural Decision Records

Este diretório armazena os ADRs (Architectural Decision Records) da feature **Sistema de Webhooks de Notificação de Pedidos**, extraídos da reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md).

Cada ADR segue o formato MADR: Status, Contexto, Decisão, Alternativas Consideradas e Consequências.

## Índice

| ADR | Decisão | Origem principal na transcrição |
| --- | --- | --- |
| [ADR-001](ADR-001-padrao-outbox-no-mysql.md) | Padrão Outbox no MySQL para publicação de eventos | [09:06]–[09:08] |
| [ADR-002](ADR-002-worker-em-processo-separado-com-polling.md) | Worker em processo separado com polling de 2s | [09:09]–[09:13] |
| [ADR-003](ADR-003-retry-com-backoff-exponencial-e-dlq.md) | Retry com backoff exponencial (5 tentativas) e DLQ em tabela separada | [09:14]–[09:18] |
| [ADR-004](ADR-004-hmac-sha256-com-secret-por-endpoint.md) | HMAC-SHA256 com secret por endpoint e rotação com grace period de 24h | [09:19]–[09:23] |
| [ADR-005](ADR-005-entrega-at-least-once-com-x-event-id.md) | Entrega at-least-once com deduplicação por X-Event-Id | [09:24]–[09:26] |
| [ADR-006](ADR-006-reuso-dos-padroes-existentes-do-projeto.md) | Reuso máximo dos padrões existentes do projeto | [09:27]–[09:30] |
| [ADR-007](ADR-007-snapshot-do-payload-na-insercao.md) | Snapshot do payload renderizado na inserção da outbox | [09:51]–[09:52] |
