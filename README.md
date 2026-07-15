# Da Reunião ao Documento: Design Docs Gerados por IA — Entrega

> O enunciado original do desafio está preservado no [repositório base](https://github.com/devfullcycle/mba-ia-desafio-design-docs-com-ia) e no histórico deste fork. Este README documenta o **processo de produção** da entrega.

## Sobre o desafio

Uma empresa que opera um Order Management System (OMS) em produção decidiu, em reunião técnica, construir um **Sistema de Webhooks de Notificação de Pedidos** — mas nada foi registrado além da transcrição da call ([`TRANSCRICAO.md`](TRANSCRICAO.md)). O desafio foi transformar essa transcrição, junto com o código existente da aplicação (Node.js + TypeScript + Express + Prisma/MySQL), em um pacote completo de design docs: PRD, RFC, FDD, ADRs e um Tracker de rastreabilidade.

A regra central: **toda informação registrada precisa ser rastreável** à transcrição ou ao código — nada de requisitos inventados. E tão importante quanto documentar o que entra é registrar o que foi **explicitamente descartado ou adiado** na reunião (email de alerta, rate limiting, dashboard, etc.), sem deixar a IA "promover" esses itens a requisito. O código da aplicação não foi alterado em nada: a entrega é puramente documental.

## Ferramentas de IA utilizadas

- **Claude Code (Anthropic)** — ferramenta principal, usada de ponta a ponta: exploração do repositório (leitura de todos os módulos, middlewares, schema Prisma e testes), análise dirigida da transcrição, geração de cada documento e os passes de verificação anti-alucinação (checagem de caminhos de arquivo citados e recontagem das linhas do tracker via script).

O papel humano foi o de **maestro**: definir a ordem de produção, escrever prompts dirigidos (abaixo), revisar criticamente cada documento gerado e mandar corrigir o que estava impreciso ou sem origem rastreável.

## Workflow adotado

A ordem seguiu a lógica de "decisões primeiro, consolidação depois":

1. **Contextualização** — leitura completa da transcrição e do código relevante (`order.service.ts`, `order.status.ts`, erros, middlewares, logger, `schema.prisma`, rotas, testes), antes de escrever qualquer documento. Isso permitiu ancorar cada afirmação técnica em arquivo real.
2. **Inventário da reunião** — extração estruturada da transcrição em quatro listas: decisões fechadas (com timestamp e autor), requisitos funcionais, restrições/RNFs e itens **descartados ou adiados**. Essa lista virou o esqueleto de tudo.
3. **ADRs primeiro** (`docs/adrs/`) — as 6 decisões principais + a decisão de snapshot do payload, cada uma com alternativas reais da reunião e trade-offs.
4. **RFC** (`docs/RFC.md`) — proposta consolidada em altura de arquitetura, referenciando os ADRs; alternativas descartadas e questões em aberto têm lugar natural aqui.
5. **FDD** (`docs/FDD.md`) — o "como construir": modelo de dados, fluxos, contratos com payloads, matriz de erros `WEBHOOK_*` e a seção de integração com o código existente (12 arquivos reais mapeados).
6. **PRD** (`docs/PRD.md`) — por último entre os grandes documentos, como consolidação de produto sobre o que já estava formalizado.
7. **Tracker** (`docs/TRACKER.md`) — varredura final dos quatro documentos, item a item, ligando cada um a `[hh:mm] Falante` ou a caminho de arquivo.
8. **Revisão final** — passe de verificação contra a checklist de critérios de aceite: existência dos arquivos citados, contagem real das linhas do tracker, conferência de que nenhum item descartado aparece como requisito.

## Prompts customizados

Prompt de inventário dirigido da transcrição (etapa 2) — o pedido explícito pelo que **não** entra foi o que evitou promover itens descartados a requisito:

```text
Leia TRANSCRICAO.md e produza um inventário estruturado da reunião, sem interpretar
além do texto. Quero 4 listas separadas, cada item com timestamp [hh:mm] e nome do falante:

1. DECISÕES FECHADAS — só o que foi explicitamente decidido (procure "decidido",
   "anotado", "tá fechado" e o resumo final da Larissa às [09:48]).
2. REQUISITOS FUNCIONAIS — o que o sistema deve fazer, na fala de quem pediu.
3. RESTRIÇÕES E REQUISITOS NÃO FUNCIONAIS — números concretos (latência, timeout,
   limites, tentativas) com a fala de origem.
4. DESCARTADO OU ADIADO — tudo que foi mencionado e NÃO entra nesta fase, com o
   motivo dito na reunião. Não omita nada desta lista: ela vira a seção
   "Fora de escopo" e as "Questões em aberto".

Se algo for ambíguo (ex.: quem pode chamar cada endpoint), cite as duas falas
conflitantes em vez de escolher sozinho.
```

Prompt da seção de integração do FDD (etapa 5) — forçando ancoragem em código real em vez de descrição genérica:

```text
Agora a seção "Integração com o sistema existente" do FDD. Regras:

- Cada linha nomeia UM caminho de arquivo real do repositório (confira que existe
  antes de citar) e descreve COMO o módulo de webhooks se integra com ele.
- Obrigatório cobrir: a extensão do changeStatus em
  src/modules/orders/order.service.ts (a assinatura publishWebhookEvent(tx, order,
  fromStatus, toStatus) veio da fala do Bruno às [09:41]), o reuso das classes de
  erro de src/shared/errors/, o requireRole de src/middlewares/auth.middleware.ts
  no replay, e o molde de src/server.ts para o novo src/worker.ts.
- Se o middleware/arquivo NÃO precisa mudar, diga isso explicitamente — "zero
  mudança" é informação de design, não omissão.
- Proibido citar arquivo que não existe no repo. Na dúvida, liste os arquivos
  do diretório antes.
```

Prompt do passe de verificação final (etapa 8):

```text
Faça um passe adversarial nos 4 documentos + ADRs:
1. Liste todo caminho de arquivo citado e verifique um a um se existe no repositório.
2. Liste todo número citado (2s, 10s, 64KB, 5 tentativas, 1m/5m/30m/2h/12h, 24h,
   100 entregas, 3 sprints) e confirme o timestamp da transcrição que o sustenta.
3. Procure os itens descartados na reunião (email, rate limiting, dashboard,
   exactly-once, multi-worker, arquivamento) aparecendo como requisito em qualquer
   documento — se aparecer fora de "fora de escopo"/"questões em aberto", aponte.
4. Recalcule as contagens do resumo do TRACKER.md a partir da tabela real.
```

## Iterações e ajustes

Os principais momentos em que a saída da IA precisou de correção:

1. **Contagem do tracker estava errada.** O resumo de cobertura do `TRACKER.md` foi inicialmente escrito com números estimados (105 linhas, 80/20) que não batiam com a tabela real. O passe de verificação recontou via script (`awk` sobre a tabela) e encontrou **127 linhas (105 TRANSCRICAO / 22 CODIGO)** — o resumo foi corrigido com os números reais. Lição: a IA estima com confiança; contagem se faz com script.
2. **Grace period de rotação estava subespecificado.** A transcrição diz que a secret antiga "fica válida por 24 horas em paralelo" ([09:21] Sofia), mas não diz **como** isso funciona numa entrega assinada por nós. A primeira versão do FDD deixava implícito que se assinava só com a secret nova — o que quebraria a validade da antiga. O fluxo foi reescrito: durante o grace period a entrega carrega as **duas assinaturas** no `X-Signature`, e o texto marca explicitamente que esse mecanismo é interpretação de implementação do FDD, não decisão da reunião.
3. **Códigos de erro além dos citados na reunião.** Bruno citou apenas `WEBHOOK_NOT_FOUND`, `WEBHOOK_INVALID_URL` e `WEBHOOK_SECRET_REQUIRED` como exemplos ([09:28]). A matriz de erros precisava de mais códigos para ser acionável (`WEBHOOK_INVALID_EVENTS`, `WEBHOOK_DISABLED`, `WEBHOOK_DEAD_LETTER_NOT_FOUND`...), e a primeira tendência era apresentá-los todos como se tivessem sido decididos. A correção foi adicionar a coluna **Origem** na matriz, separando código citado na reunião de código *derivado* de um requisito rastreável.
4. **Risco de duplicação entre RFC e FDD.** O primeiro esboço do RFC descia a detalhes de tabela e agenda de retry que pertencem ao FDD. O RFC foi enxugado para altura de arquitetura (proposta + alternativas + questões em aberto, com tabela-resumo), e o detalhamento ficou só no FDD — respeitando a fronteira de altura entre os documentos.

No total, foram **4 ciclos principais** de geração → revisão crítica → correção (inventário/ADRs, RFC+FDD, PRD+Tracker, passe final de verificação).

## Como navegar a entrega

```
docs/
├── PRD.md          ← comece aqui: por que e o quê
├── RFC.md          ← a proposta técnica e o que ficou em aberto
├── adrs/
│   ├── README.md   ← índice das 7 decisões
│   └── ADR-001..007-*.md
├── FDD.md          ← como construir, em detalhe
└── TRACKER.md      ← de onde veio cada item (127 linhas rastreadas)
TRANSCRICAO.md      ← a fonte de tudo (não alterada)
```

**Ordem sugerida de leitura:**

1. [`TRANSCRICAO.md`](TRANSCRICAO.md) — opcional, mas dá contexto de tudo que vem depois.
2. [`docs/PRD.md`](docs/PRD.md) — problema, público, escopo e métricas.
3. [`docs/RFC.md`](docs/RFC.md) — proposta técnica, alternativas descartadas e questões em aberto.
4. [`docs/adrs/`](docs/adrs/README.md) — as 7 decisões, uma a uma, com trade-offs.
5. [`docs/FDD.md`](docs/FDD.md) — especificação acionável: dados, fluxos, contratos, erros, integração com o código.
6. [`docs/TRACKER.md`](docs/TRACKER.md) — auditoria de rastreabilidade de tudo acima.
