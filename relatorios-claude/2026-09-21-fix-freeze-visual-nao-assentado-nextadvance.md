# Fix: freeze pós-item-3 (e além) causado por dessincronia entre nextAdvanceReady() e o gate real de promoção

**Data:** 2026-09-21
**Escopo:** investigação dedicada do "novo freeze" descrito em `2026-09-21-retest-producao-amparo-novo-freeze.md` (tela em branco ao avançar de item, servidor mudo, botão "Continue to next item" nunca recupera).
**APP SHA (fix commitado e pushado):** `9bf2eea1aad41b600031e9a0660e8979a1ea8d6e` (branch `main`, `BOM`)
**SERVER SHA (inalterado — fix é 100% client-side):** `5f7e0cf11c9ba3a773e5f9360dea8d180683ca12` (`/opt/sim/current` no droplet `sim-api-sgp1`)
**Base URL usada na validação física:** `https://simaitutor.com` (via `--dart-define=SIM_SERVER_URL=https://simaitutor.com`)

## Causa raiz

`LessonRuntimeEngine.nextAdvanceReady()` (usado para (a) habilitar o botão "Continue to next item" e (b) decidir se a bomba de prefetch pós-feedback precisa retentar) só verificava se o **texto** do próximo item estava pronto (`isLessonMaterialReadyInStateOrCache`). Já `LessonMaterialController.carregarRapidoSePronto()` — o gate que de fato decide se pode *renderizar* o próximo item quando o usuário toca em "Continue" — também exige que o **visual obrigatório** daquele item esteja assentado (`_isVisualSettledForPosition`: qualquer status exceto `'processing'`).

Quando o texto do próximo item chega antes do visual (janela comum, visto nos logs reais: `media_n3_start`/`media_n3_end` levando 5-6s), as duas autoridades discordavam: o botão via texto pronto e se declarava "ready" (`nextAdvanceReady() == true`), mas `carregarRapidoSePronto` recusava (corretamente) promover um item com visual ainda em geração. O toque em "Continue" então:

1. Cai no ramo de `avancar()` que marca `phase = avancoPendente` com `letter`/`signal` preenchidos (`postAnswerAdvancePending == true`).
2. Isso **suprime** a bolha de "preparando" (`aula_advance_pending`) no `chat_aula_timeline_builder.dart`, porque essa mensagem só é adicionada quando `!postAnswerAdvancePending`.
3. Como `content == null` (o próximo item ainda não tem conteúdo local), o bloco inteiro que constrói explicação/pergunta/opções/sinais/feedback é pulado — zero balões renderizados. Tela em branco.
4. A única bomba que poderia reavaliar e reparar isso (`_ensurePostFeedbackNextAdvancePrepared` em `chat_aula_screen.dart`) exige uma mensagem `doubtAction` ativa renderizada — que nunca existe nesse estado (fase não é `concluido`, `content` é `null`). Beco sem saída permanente: nenhuma nova requisição sai do app, nenhum retry, nenhum erro — só silêncio.

Reproduzido fisicamente em produção real (antes do fix): item 3, segundo agravante, toque em "Continue to next item" → tela em branco permanente, `Review` some do topo, zero requisições novas por mais de 4 minutos (confirmado via `journalctl` no droplet e via log Dart completo com `flutter run` anexado ao tablet).

## Fix

Uma linha de autoridade só: `LessonRuntimeEngine._slotMaterialReady()` (a função que `nextAdvanceReady()` usa) agora também verifica o mesmo `visualSettled` que `carregarRapidoSePronto` já aplicava, via `PackageAuthority.packageFor` + `Sim109ExperienceValue.fromTransportLayer` — os mesmos dados que a segunda checagem já lia, sem duplicar lógica nova. Arquivo: `lib/sim/classroom/lesson_runtime_engine.dart`.

Com isso, enquanto o visual do próximo item está `'processing'`:
- O botão mostra honestamente "Preparing the next step." (em vez de mentir que está pronto).
- A bomba de retry pós-feedback (`ensureNextAulaAdvancePrepared`) volta a rodar (porque `nextAdvanceReady()` agora é `false`), então o app continua reavaliando a cada poucos segundos até o visual assentar — sem travar.

## Testes

- Novo teste de regressão em `test/classroom_phase_test.dart` (`nextAdvanceReady must agree with carregarRapidoSePronto...`): conduz um `LessonRuntimeEngine` real por E1→E2→conclusão do item 0, depois injeta o pacote do item 1 com `imageStatus: 'processing'` via `StoreSim109ItemPackageCommand`/`PatchSim109PackageExperienceImageCommand` (as APIs canônicas do `StudentStateStore`, não escrita direta de estado). Confirmado: **falha sem o fix** (`Expected: false, Actual: true`) e **passa com o fix**.
- Suíte completa do app: `flutter test` → **1482 testes, todos passando**, no commit `9bf2eea` (mesmo commit pushado).
- `flutter analyze`: limpo, sem apontamentos.

## Confirmação física em produção real

Build de debug com `--dart-define=SIM_SERVER_URL=https://simaitutor.com`, instalado no Galaxy Tab A9 (SM-X216B) via `flutter run` anexado (visibilidade completa de log Dart).

1. **Antes do fix** (binário do commit anterior, `b0481cc`): reproduzido o freeze exatamente como descrito acima — item 3, 2º agravante, toque em "Continue to next item" → tela em branco permanente, confirmado via log Dart e via `journalctl` no droplet (silêncio total de requisições).
2. **Depois do fix** (commit `9bf2eea`, mesma sequência de toques, mesma conta de teste): o app avançou do item 3 para o item 4 normalmente, sem tela em branco.
3. Repeti a sequência (mais um erro no item 4): desta vez o visual do item 5 ainda estava gerando no momento do toque — o botão mostrou corretamente "Preparing the next step." (cinza, não clicável) em vez de travar. Este é o comportamento pretendido pelo fix: ele nunca mais promete algo que não pode entregar.

### Limitação encontrada ao tentar fechar o ciclo completo de 5 agravantes (Amparo) com conta isolada

A conta de QA usada nas etapas 1-3 acima (`7cf6000c-4795-475a-bd2b-8b5e05d822c7`) já está catalogada em `2026-09-20-incidente-app-mode-reconfirmacao-e-decisao-qa.md` como tendo uma divergência de histórico irreconciliável entre estado local e Supabase (2148 revisões no Supabase contra 44 localmente) — confirmei isso na prática: toda tentativa de `persist` retornava `409 STATE_EXPECTED_REVISION_MISMATCH` (`existingRevision: 233` vs `expectedRevision: 224`), mesmo após restart completo do app (`force-stop` + reabrir). Isso trava o ciclo de preparação do próximo item de forma **não relacionada a este fix** — é uma dessincronia de estado preexistente e documentada dessa conta específica.

Tentei então criar uma conta nova e isolada (conforme pedido) para fechar o ciclo completo de 5 agravantes → Amparo:
- Todas as 5 contas de QA já catalogadas têm a mesma classe de problema (ver relatório citado) — nenhuma serve como "conta limpa".
- Tentativa de `Create new account` (email/senha, dois emails diferentes, incluindo após `pm clear` completo do app para eliminar qualquer estado local residual) falhou consistentemente com "I could not finish signing in now. Try again soon." — sem nenhum traço de exceção Dart capturado no log (a app não expõe detalhe do erro além dessa mensagem genérica). Não investiguei mais a fundo por já ter excedido bastante o orçamento de tempo desta sessão; a hipótese mais provável é rate-limit de signup do Supabase Auth para o IP deste tablet, dado o volume de sessões de teste rodadas nele ao longo da missão.

**Consequência:** não fechei o ciclo completo de 5 erros → Amparo com uma conta genuinamente limpa nesta sessão. O que está confirmado com alta confiança é o fix do freeze em si (root cause identificada, testada automaticamente, e confirmada fisicamente em produção real, duas vezes, no ponto exato onde travava antes). O ciclo completo do Amparo (task #122) precisa de uma sessão futura com uma conta de teste utilizável (nova, via painel Supabase diretamente, ou aguardando o rate-limit expirar).

## Achado relacionado, não resolvido nesta sessão

O relatório `2026-09-21-validacao-fisica-amparo-freeze-descoberto.md` (sessão anterior, mesmo dia) descreve um freeze de tela em branco especificamente na *transição de abertura do Amparo* após ~5 erros, contra um terceiro servidor (`167.179.109.137:3020`, build antiga instalada em 2026-09-19/20). Os sintomas descritos lá (tela em branco, botão Review some, servidor mudo) são consistentes com a mesma família de bug corrigida aqui, mas essa sessão não tinha logs Dart para confirmar. Como não consegui reproduzir o cenário completo de Amparo nesta sessão (bloqueado pela limitação de conta acima), **não está confirmado se este fix também resolve aquele caso específico** — recomendo que a próxima sessão com uma conta utilizável valide isso explicitamente como primeiro passo, antes de assumir que a task #122 está pronta para ser fechada.

## Itens de checklist

- [x] Causa raiz identificada e confirmada em log real (produção)
- [x] Fix aplicado (`lib/sim/classroom/lesson_runtime_engine.dart`)
- [x] Teste de regressão determinístico (falha sem fix, passa com fix)
- [x] Suíte completa (1482 testes) verde + `flutter analyze` limpo
- [x] Commit + push (`9bf2eea`, `origin/main`)
- [x] Confirmação física em produção real (`https://simaitutor.com`) do ponto exato de freeze, antes/depois
- [ ] Ciclo completo de 5 agravantes → Amparo com conta isolada (bloqueado por limitação externa de teste, não pelo fix)
- [ ] Task #122 permanece **não marcada como completed** até a confirmação acima ser possível
