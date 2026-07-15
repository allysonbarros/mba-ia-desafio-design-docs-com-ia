# ADR-004 — Autenticação HMAC-SHA256 com secret por endpoint e rotação com grace period

## Status

**Aceita** — decidida na reunião técnica registrada em [`TRANSCRICAO.md`](../../TRANSCRICAO.md) ([09:22] Sofia: "Decidido: HMAC-SHA256 sobre o corpo do request, secret por endpoint, suporte a rotação com grace period de 24h").

**Decisores:** Sofia (Eng. de Segurança), Diego (Eng. Sênior — Plataforma), Bruno (Eng. Pleno — Pedidos), Larissa (Tech Lead).

## Contexto

Os webhooks expõem eventos com dados de pedidos para endpoints **fora da nossa infraestrutura**. O cliente precisa conseguir validar duas coisas: que a requisição veio realmente de nós, e que ninguém adulterou o payload no caminho ([09:19] Sofia).

Contexto adicional que pesou na decisão: já houve cliente que **vazou secret em log da aplicação dele** ([09:22] Diego) — o desenho precisa limitar o raio de dano de um vazamento e permitir troca de secret sem downtime.

## Decisão

1. **Assinatura HMAC-SHA256 sobre o corpo do request**, enviada no header `X-Signature`. SHA-256 é o padrão de mercado e todo cliente sério tem biblioteca para verificar ([09:20] Sofia).
2. **Secret única por endpoint de webhook**, não global da plataforma. A secret é **gerada pelo nosso sistema** na criação do webhook e devolvida na resposta ([09:21] Sofia; [09:31] Marcos).
3. **Rotação via API**: o cliente pede nova secret; a antiga permanece válida por **24 horas em paralelo** (grace period) para dar tempo de migrar os sistemas dele; depois disso, a antiga morre ([09:21] Sofia).
4. **TLS obrigatório**: URL de webhook tem que ser `https`; cadastro com `http` é recusado com erro de validação no schema Zod ([09:23] Sofia).
5. Header `X-Timestamp` acompanha o envio, permitindo ao cliente detectar replay attack se quiser ([09:44] Diego).

## Alternativas Consideradas

### 1. Secret global da plataforma — descartada

- **Trade-off que motivou o descarte:** "se vaza uma, vaza tudo" ([09:21] Sofia). Com secret por endpoint, um vazamento compromete apenas aquele endpoint, e a rotação resolve sem afetar os demais clientes.

### 2. Confiar apenas no TLS, sem assinatura de payload — alternativa plausível, não adotada

- **Trade-off:** TLS protege o canal, mas não permite ao cliente validar a **origem** da requisição (qualquer um pode fazer POST https no endpoint dele) nem a integridade fim-a-fim de forma verificável. A exigência da Sofia foi explícita: o cliente tem que conseguir validar que veio de nós e que o payload não foi adulterado ([09:19] Sofia).

## Consequências

### Positivas

- Autenticidade e integridade verificáveis pelo cliente com biblioteca padrão de HMAC-SHA256 ([09:20] Sofia).
- Raio de dano de vazamento limitado a um endpoint ([09:21] Sofia).
- Rotação sem downtime: as duas secrets convivem por 24h ([09:21] Sofia).

### Negativas / trade-offs assumidos

- Durante o grace period de 24h, uma secret antiga comprometida ainda assina requisições válidas — janela de risco aceita em troca da migração sem quebra.
- O módulo precisa gerenciar até duas secrets ativas por endpoint (atual + anterior com expiração), aumentando a complexidade do modelo de dados e da assinatura.
- A verificação do lado do cliente é responsabilidade dele — exige documentação clara no portal do desenvolvedor ([09:26] Marcos).
- A revisão de segurança da Sofia (HMAC e geração de secret) é etapa obrigatória antes do deploy, com pelo menos 2 dias úteis reservados ([09:46] Sofia).
