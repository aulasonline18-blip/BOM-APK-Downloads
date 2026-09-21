# Rastreamento estático do 3º stall do Amparo ("Preparando próximo passo" travado) — candidatos a causa raiz

**Contexto:** continuação do diagnóstico do stall reportado em `2026-09-21-amparo-novo-stall-preparando-proximo-passo.md` (commit `ec35083`). Esta rodada fez um rastreamento estático completo da cadeia de decisão (sem reprodução física nova, por restrição de orçamento/tempo desta execução) e não chegou a uma correção confirmada — deixo os pontos exatos do código que precisam de instrumentação (`debugPrint`) na próxima reprodução com `flutter run` attached.

## Cadeia completa rastreada (app BOM)

1. **UI trigger**: `chat_aula_screen.dart:_ensurePostFeedbackNextAdvancePrepared` só agenda qualquer trabalho se `hasActivePostFeedbackActions` for true (existe uma mensagem `doubtAction` não-histórica visível). Se essa condição ficar falsa em algum ponto do fluxo de erros consecutivos, o pump inteiro para de rodar silenciosamente — **candidato nº1**.
2. Quando agenda, chama `session.ensureNextAulaAdvancePrepared()` uma vez por "scope" (chave = lessonLocalId + itemMarker + itemTitle + headerLabel + phase.signal.value). Se o resultado não ficar pronto, arma um `Timer(2s)` que só faz `setState()` — depende do próximo `build()` re-invocar `_ensurePostFeedbackNextAdvancePrepared` com o MESMO scope para continuar tentando.
3. `lab_session.dart:ensureNextAulaAdvancePrepared` → se `organism.lessonRuntimeEngine.nextAdvanceReady()` for false, cai em `_retryNextAdvancePrefetchIfDue`.
4. `_retryNextAdvancePrefetchIfDue`: bomba com **budget de 4 tentativas, cooldown de 20s entre elas, reset de budget só depois de 2 minutos parado** (`_nextAdvanceRetryMaxAttempts=4`, `_nextAdvanceRetryCooldown=20s`, `_nextAdvanceRetryBudgetResetAfter=2min`), chaveada por `scopeKey = lessonLocalId:itemIdx:itemMarker:layer`. Quando dentro do budget, chama `organism.prepareNextItemPackageIfAuthorized(...)`.
5. **`sim_organism.dart:prepareNextItemPackageIfAuthorized`** (linha ~362) tem DOIS early-returns silenciosos que merecem instrumentação:
   - `if (current == null || current.layer != LessonLayer.l2 || state.curriculum == null) return;` — só dispara prefetch quando a experiência ATIVA está em L2. Tracei manualmentea semântica de agravantes (item recebe L1 então L2 antes de avançar de item) e, PELO MENOS nas transições descritas no relatório anterior (agravante 2→item2, agravante 4→item3), o layer ativo no momento do erro deveria ser L2 — ou seja, esse guard não parece ser o bloqueio direto nesses pontos específicos, mas pode ser relevante se a semântica de layer durante o loop de reforço for diferente do que assumi (não confirmado ao vivo).
   - `if (PackageAuthority.resolve(state).hasNext) return;` — `PackageAuthority.resolve` recalcula `current`/`next` a partir do cursor de experiência ATUAL a cada chamada (não é uma flag global obsoleta — conferi `student_learning_state.dart:538-596`), então uma prefetch antiga de um item anterior não deveria bloquear isso para o item seguinte. Ainda assim, vale confirmar ao vivo com print do valor de `hasNext` e do `itemIdx` resolvido no momento exato do stall.
6. Mesmo que o prefetch rode, o gate que realmente decide "pronto" é `lesson_runtime_engine.dart:_nextAdvanceReady` → `_nextAdvanceTarget` (calcula item/layer alvo via `decideNextActionFromState`) → `_slotMaterialReady` → `_visualSettledForSlot` → `PackageAuthority.packageFor(state, itemIdx, marker).contentFor(experience).visualSettled`, que por sua vez é só `imageStatus != 'processing'` (`student_learning_state.dart:327`).
7. **Pergunta ainda sem resposta**: o `POST /api/visual-route` que retornou 200 no log do servidor (05:04:27–05:04:37) — era para o MESMO `(itemIdx, marker, layer)` que `_nextAdvanceTarget` está esperando, ou para um slot diferente (ex.: o item que acabou de ser respondido errado, não o próximo)? Se for para um slot diferente, a resposta nunca atualizaria o `imageStatus` do slot que `_visualSettledForSlot` está checando, e o app ficaria honestamente esperando um evento que nunca vai atualizar aquele slot específico — esse é o candidato mais provável no momento, mas **não confirmado**.

## O que falta (próxima sessão, com orçamento para reprodução física completa)

Instrumentar com `debugPrint` (temporário, remover depois de confirmar) exatamente estes 4 pontos antes de rodar `flutter run` attached e repetir a sequência de 4 erros com a conta `qa-amparo-20260921@sim-internal-test.invalid`:

1. `sim_organism.dart` linha ~366-377: logar `current?.layer`, `state.curriculum != null`, `PackageAuthority.resolve(state).hasNext`, e qual `nextItemIdx`/`marker` seria prefetchado, toda vez que a função é chamada.
2. `lab_session.dart:_retryNextAdvancePrefetchIfDue`: logar `scopeKey`, `_nextAdvanceRetryAttempts`, se está em cooldown ou budget exaurido, a cada chamada.
3. `lesson_runtime_engine.dart:_nextAdvanceTarget`: logar `targetItemIdx`/`targetLayer` calculado.
4. `lesson_runtime_engine.dart:_visualSettledForSlot`: logar `itemIdx`/`marker`/`layer` recebidos e o `imageStatus` efetivamente lido do `PackageAuthority.packageFor(...)`.

Comparar o `(itemIdx, marker, layer)` do ponto 4 com o que o servidor realmente respondeu no `/api/visual-route` (payload da requisição, não só o status 200) — se forem diferentes, a causa raiz é uma incompatibilidade entre "qual slot o app está esperando" e "qual slot o servidor preencheu", e o fix é garantir que o prefetch/retry mire exatamente o slot que `_nextAdvanceTarget` está aguardando (hoje `prepareNextItemPackageIfAuthorized` sempre prefetcha `currentIdx+1` a partir da experiência ativa, que pode não coincidir com `_nextAdvanceTarget` durante o loop de reforço/agravantes).

## Estado

Nenhuma mudança de código feita nesta rodada — só rastreamento estático (leitura de código, sem `flutter run`). Task #122 continua **IN_PROGRESS**, não fechada. Conta de teste `qa-amparo-20260921@sim-internal-test.invalid` continua pronta e reutilizável para a próxima tentativa.
