# Causa raiz real do 3º stall do Amparo encontrada e corrigida — 2026-09-21

**Contexto**: retomando o trace estático de `2026-09-21-amparo-stall-trace-estatico-candidatos.md` (commit `46ed8e0`), instrumentei o código com `debugPrint` temporário e usei `flutter run` attached ao tablet contra produção real (`https://simaitutor.com`) para confirmar a causa raiz do stall "Preparando próximo passo"/"Continuar para a aula" travado.

## Causa raiz real (diferente da hipótese anterior)

Ao reproduzir o cenário do handoff (login com `qa-amparo-20260921@sim-internal-test.invalid`, criar uma aula, avançar até o botão "Continuar para a aula"), a **primeira** aula da sessão sempre funcionou perfeitamente. O bug só aparece ao criar uma **segunda aula nova na mesma sessão do app** (mesmo processo, sem reiniciar): o botão "Continuar para a aula" da segunda aula não faz absolutamente nada ao ser tocado — nem sequer entra no estado "Preparando..." — porque:

- `WarmupBridgeCoordinator.aulaNavigationStarted` é setado para `true` quando a primeira aula abre (`markAulaNavigationStarted()`).
- `shouldOpenOfficialAula()` tem um curto-circuito: `if (aulaNavigationStarted) return false;` — sempre, independente de `officialLessonReady`/`continueRequested`.
- `saveObjectiveEntry()` (o handler do botão "Preparar minha aula", que inicia uma aula nova) **nunca resetava** o `WarmupBridgeCoordinator` — só uma função irmã (`prepareObjectiveWorkflow`, usada por outro caminho de entrada) fazia esse reset.
- Resultado: qualquer aula criada depois da primeira, na mesma sessão do app, tem `aulaNavigationStarted` preso em `true` desde o início, e `continueFromWarmupToAula()` nunca consegue navegar — silenciosamente, sem erro, sem requisição de rede, exatamente como reportado.

Log capturado no momento exato do bug (antes do fix), tocando "Continuar" numa segunda aula já pronta (`officialLessonReady=true`):
```
continueFromWarmupToAula tapped. officialLessonReady=true continueRequested=false aulaNavigationStarted=true
continueFromWarmupToAula proceeding to _tryOpenOfficialAula
_tryOpenOfficialAula source=warmup_continue blocked: shouldOpenOfficialAula=false (ready=true continueRequested=true navStarted=true)
```

Isso explica também por que o bug pareceu "aleatório" nas rodadas anteriores desta sessão: só aparece na **segunda aula em diante** dentro do mesmo processo do app — a primeira aula de qualquer sessão sempre funciona.

## Fix

`lib/features/session/lab_session_entry_flows.dart`, em `saveObjectiveEntry()`: chama `clearWarmupState()` + `resetEntryCoordinator()` logo depois de aceitar a nova submissão de aula, espelhando o que `prepareObjectiveWorkflow()` já fazia para o seu próprio caminho.

## Verificação

1. Teste de regressão determinístico (`test/warmup_bridge_contract_test.dart`): cria duas aulas completas na mesma sessão (`LabSession`), afirma que ambas alcançam `/cyber/aula`. **Falha sem o fix** (segunda aula trava em `/cyber/warmup`), **passa com o fix**.
2. Suíte completa: 1483 testes verdes, `flutter analyze` limpo.
3. **Confirmação física em produção real**: rebuild do APK (`flutter build apk --release --dart-define=FLUTTER_APP_MODE=production --dart-define=SIM_SERVER_URL=https://simaitutor.com --dart-define=SIM_AUTH_REDIRECT_URL=simaitutor://login-callback`), e também via `flutter run` attached para visibilidade de log. Criei duas aulas novas seguidas na mesma sessão com a conta `qa-amparo-20260921@sim-internal-test.invalid` contra `https://simaitutor.com`: a segunda aula ("Divisão de frações") abriu corretamente (Item 1/20, conteúdo gerado corretamente), sem nenhum travamento.

**Commit**: `f85bcfa` (app, `lib/features/session/lab_session_entry_flows.dart` + `test/warmup_bridge_contract_test.dart`). Pushado para `main`.

## Nota operacional

Durante a verificação, o build de release exigiu um quarto dart-define não documentado no handoff anterior: `SIM_AUTH_REDIRECT_URL=simaitutor://login-callback` (gate `BOM_RELEASE_CALLBACK_REQUIRED` em `android/app/build.gradle.kts`). Atualizando o handoff master com isso.

Também: instalar um APK de release por cima de uma instalação de debug (ou vice-versa) falha com `INSTALL_FAILED_UPDATE_INCOMPATIBLE` (assinaturas diferentes) — é preciso `adb uninstall com.simaitutor.app` antes de trocar entre os dois tipos de build no mesmo dispositivo.

## Impacto no Amparo (#122)

Com essa causa raiz corrigida, o ciclo completo de 5 agravantes → Amparo pode finalmente ser tentado sem o risco de travar antes de chegar lá. Ainda não testei o ciclo de 5 erros especificamente nesta rodada (o tempo foi consumido pela investigação + fix + verificação física) — próxima ação exata na seção abaixo.

## PRÓXIMA AÇÃO EXATA

1. Login na conta `qa-amparo-20260921@sim-internal-test.invalid` (já tem 2 aulas criadas nesta sessão de verificação — pode reusar uma delas ou criar uma terceira).
2. Entrar numa aula, errar 5 vezes seguidas com "Tenho certeza"/"I am sure".
3. Confirmar que a sala de Amparo abre e completa um ciclo sem travar.
4. Marcar task #122 como completed e `OK_PRODUCTION` na matriz do handoff.
5. Seguir para os itens #123-127, #120-121 ainda pendentes.
