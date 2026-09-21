# Novo achado: stall indefinido em "Preparando próximo passo" durante o ciclo de 5 agravantes → Amparo (produção real)

**Data:** 2026-09-21
**Escopo:** continuação do fechamento da task #122 (Amparo 5-agravantes) contra produção real, usando conta de QA nova criada via Supabase Admin API.
**APP:** debug build, `--dart-define=SIM_SERVER_URL=https://simaitutor.com`, `flutter run` anexado ao tablet físico (Galaxy Tab A9).
**SERVER SHA:** `5f7e0cf11c9ba3a773e5f9360dea8d180683ca12` (inalterado).
**Conta:** `qa-amparo-20260921@sim-internal-test.invalid` (`261f7461-0cff-4222-afac-9002d6b0e400`), criada via `POST {SUPABASE_URL}/auth/v1/admin/users` (contorna o rate-limit de signup público que bloqueava a sessão anterior).

## O que funcionou

1. Login com a conta nova funcionou normalmente (o rate-limit anterior era específico do endpoint de signup público, não do login).
2. Onboarding completo (9 etapas) sem problema.
3. **4 agravantes consecutivos confirmados**, cada um respondendo errado deliberadamente com sinal "Tenho certeza":
   - Agravante 1 (item 1, tentativa 1): `8/3` no lugar de `3/8` → "Não foi dessa vez. Vamos corrigir a base."
   - Agravante 2 (item 1, tentativa 2/2): `8/3` de novo → mesma mensagem.
   - Agravante 3 (item 2... na verdade a numeração de item avançou para o próximo depois do reforço): `6/2` no lugar de `2/6` → mesma mensagem.
   - Agravante 4: `O 4` no lugar de `O 3` (fração 3/4, numerador) → mesma mensagem.
4. **Confirmação positiva do fix anterior**: depois do 4º erro, o botão mostrou honestamente **"Preparando próximo passo"** (cinza, não clicável) em vez de congelar com tela em branco — exatamente o comportamento pretendido pela correção do freeze pós-item-3 (`lesson_runtime_engine.dart`, commit `9bf2eea`). Isso já é uma confirmação parcial de que aquele fix se sustenta em produção real, sob carga de erros repetidos.

## Achado novo, não resolvido

O estado "Preparando próximo passo" **não saiu desse estado em mais de 7 minutos** (de 05:04 a 05:13, quando parei de aguardar).

Investiguei antes de escalar:
- **Rede**: OK. Ping ao droplet real: 242ms. Túnel Tailscale ativo e validado.
- **Servidor**: o `POST /api/visual-route` relevante **já tinha completado com sucesso** (`status 200`, `4506ms` e depois outro em `235ms`/`5372ms`) às `05:04:27–05:04:37`, ou seja, o visual do próximo item estava pronto do lado do servidor bem antes de eu checar o app pela última vez.
- **App**: nenhuma nova requisição HTTP saiu do app depois do último `postJson status 200` (o envio do sinal "Tenho certeza" do 4º erro). Ou seja: o servidor entregou o visual, mas o app não reavaliou/reconheceu que o gate (`nextAdvanceReady()`/`carregarRapidoSePronto`) já podia liberar o botão.
- Tentei backgroundar o app (Home) e trazer de volta (`am start`): o Flutter reabriu a aula a partir do snapshot local (`source=restart decision=openLessonLocally`) e a tela redesenhou a pergunta do item atual sem as respostas marcadas — não ficou claro se isso é só um artefato visual do redraw ou se indica perda de algum estado transitório. Não tive tempo de confirmar se o próprio resume destravou o avanço (parei a investigação neste ponto).

## Hipótese (não confirmada)

O fix do freeze pós-item-3 corrigiu o **gate de decisão** (`nextAdvanceReady()` agora concorda com `carregarRapidoSePronto` sobre o visual precisar estar assentado) e a **bomba de retry** (`ensureNextAulaAdvancePrepared`) deveria reavaliar esse gate periodicamente até ele ficar verdadeiro. O que pode estar faltando é a bomba de retry não estar sendo re-armada/continuada corretamente **especificamente na sequência de múltiplos erros consecutivos com sinal "certeza"** — possivelmente ela para de rodar depois de um certo número de ciclos, ou não é re-disparada depois do 4º evento de sinal, deixando o app honestamente "esperando" para sempre em vez de travar silenciosamente (uma melhoria real sobre o bug antigo, mas ainda um bloqueio funcional).

Não confirmei essa hipótese com certeza — precisa de outra sessão com orçamento de tempo para inspecionar `ensureNextAulaAdvancePrepared` e seus pontos de disparo (`chat_aula_screen.dart`) e ver se ele é cancelado/não reagendado depois do fluxo de sinal de confiança.

## Estado da task #122

**Ainda não fechada.** Não cheguei ao 5º erro nem à sala de Amparo. Confirmado: os dois fixes anteriores (race no hydrate, e gate de nextAdvanceReady) seguram bem sob 4 erros consecutivos — o que trava agora é um terceiro problema, na mesma família (avanço pós-resposta), mas em um ponto diferente do fluxo (após múltiplos ciclos de sinal de confiança, não na primeira transição).

## Recomendação para a próxima sessão

1. Reproduzir de novo (mesma sequência: 4 erros com "Tenho certeza") com `flutter run` e, desta vez, inspecionar via DevTools/prints adicionais se `ensureNextAulaAdvancePrepared` continua sendo chamado/reagendado depois do 4º ciclo.
2. Conferir se existe algum limite de tentativas (retry ceiling) na bomba que, após N ciclos, para de reagendar — se sim, isso explicaria exatamente este sintoma.
3. Só depois disso, retomar o 5º erro e a validação final da sala de Amparo.
