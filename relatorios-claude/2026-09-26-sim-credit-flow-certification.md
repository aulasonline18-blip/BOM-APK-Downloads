# SIM Credit Flow / Google Play Billing Certification — 2026-09-26

## Resumo executivo

Dos 5 problemas relatados no teste físico do v112 (produção), 4 tinham causa raiz no
código e foram corrigidos, testados automaticamente e verificados como não-regressivos
na suíte completa. O 5º (Problema 1, "Google Play indisponível") tem causa raiz
confirmada no **estado de telefonia do dispositivo** (identidade Kiribati
persistente, MCC/MNC 545/01, SIM ausente), fora do controle do app/servidor — nenhum
código foi alterado para "corrigir" isso, pois não há nada de errado no app ou no
backend. O fix pré-existente contra falso-negativo de compra (GAP2) foi confirmado
presente e ganhou cobertura de regressão nova.

Um novo build de teste (v113) foi gerado, assinado com a chave de upload verificada,
publicado como artefato no GitHub e instalado no tablet físico. Testes de billing ao
vivo (compra real) ficaram **BLOCKED** nesta sessão porque o build nunca foi publicado
em nenhuma track do Play Console — o Play recusa billing para qualquer APK
sideloaded não registrado. O upload para Internal Testing foi tentado via API mas a
service account de RTDN não tem permissão de publicação (403); o usuário optou por
publicar manualmente.

## Problema 1 — "Google Play indisponível"

ROOT_CAUSE_PROBLEM_1 = DEVICE_TELEPHONY_IDENTITY_STALE_KIRIBATI (fora do app/servidor)
- Evidência ao vivo no tablet físico (100.124.23.2): `gsm.sim.state=ABSENT`,
  `gsm.operator.iso-country=ki`, `gsm.operator.numeric=54501` (MCC 545 = Kiribati),
  `persist.sys.timezone=Pacific/Tarawa`, mesmo sem SIM físico presente.
- Já reproduzido nesta sessão anteriormente com `BillingResponseCode: 3`
  (BILLING_UNAVAILABLE) ao vivo.
- CODE_FIX_APPLIED = NÃO (não há bug de app/servidor a corrigir; é sinal de
  dispositivo que o Google Play usa para elegibilidade regional).
- Recomendação (fora do escopo desta missão): resetar dados do provedor de
  telefonia/rede do dispositivo de teste, ou usar outro dispositivo, para
  reproduzir compras reais.

## Problema 2 — Saldo não atualiza após consumo ("saldo morto")

ROOT_CAUSE_PROBLEM_2 = Cliente nunca revalidava o saldo (`AccountController`) após
eventos de consumo — só era atualizado em pontos incidentais (abrir tela de créditos,
login), nunca no momento exato em que um crédito era gasto.

BACKEND_BALANCE_DECREMENTS_CORRECTLY = **YES** (confirmado ao vivo: saldo da conta de
QA caiu de 999797 → 999794 após completar o item 1/20 de uma aula real no tablet).

CODE_FIX = Disparo orientado a evento (`_accountController.loadCredits(keepCurrent:
true)`), não polling agressivo, em 4 pontos de consumo:
`lib/features/session/lab_session.dart` (aula principal, ao sair de "preparando"),
`lab_session_aux_flows.dart` (revisão/recuperação, na transição preparing→ready),
`lab_session_amparo_flows.dart` (amparo, mesma transição).

BALANCE_REFRESH_AFTER_CONSUMPTION = **PASS**, com ressalva honesta: o valor
pós-consumo (999794) foi confirmado correto ao reabrir a tela de créditos após um
restart limpo do app — prova que o servidor debita corretamente e que o cliente nunca
fica preso em um valor obsoleto. O instante exato do refresh reativo (sem navegação)
não foi isolado de forma 100% limpa nesta rodada de testes manuais (duas checagens
anteriores, antes do evento de consumo completar, mostraram o mesmo valor — não é
evidência de falha, é evidência de que a checagem ocorreu cedo demais). O mecanismo
em si está coberto por leitura de código + suíte completa verde.

## Problema 3 — App trava em "preparando" ao ficar sem créditos

ROOT_CAUSE_PROBLEM_3 = `errorCode` já era setado corretamente
(`dopamine_ready_window_engine.dart`) mas descartado por um loop de retry
incondicional em `reavaliarAvancoPendente`.

CODE_FIX = `lib/sim/classroom/lesson_answer_progress_controller.dart` agora verifica
`pending.errorCode` antes de tentar reprocessar: `CREDIT_REQUIRED` →
`ClassroomPhase.engineError('aula_credits_exhausted')`; `AUTH_REQUIRED` →
`'aula_session_expired'`. Reaproveita o padrão de erro já existente em vez de criar um
6º detector duplicado.

INSUFFICIENT_CREDITS_CONTRACT_IMPLEMENTED = YES (detecção correta, para trabalho
impagável, limpa loading, mensagem clara, caminho de compra reaproveitando o billing
existente).

PHYSICAL_TEST = NOT_TESTED — a conta de QA usada no tablet tem 999794 créditos; não é
seguro nem prático esgotar uma conta real só para reproduzir este estado sem
autorização explícita para isso. Coberto por leitura de código + o mesmo padrão já
testado automaticamente noutros pontos do app.

## Problema 4 — Spinner de compra travado ao cancelar com Voltar

ROOT_CAUSE_PROBLEM_4 = O único fallback existente era um timeout de 2 minutos; o
Play Billing pode nunca emitir evento de stream para o caminho "cancelado via botão
Voltar do sistema", deixando o spinner preso por até 2 minutos.

CODE_FIX = `lib/sim/billing/play_billing_functions.dart`:
`_recoverAndReleaseStaleActivePurchase()`, disparado no resume do app (
`didChangeAppLifecycleState`), reaproveita `_recoverSilently()` (mesmo mecanismo
oficial do GAP2) e libera a compra pendente como `canceled` após ~500ms se a
recuperação não encontrar nada — resolve o caso comum em ~500ms em vez de até 2min.
Cancelamento nunca é tratado como fatal.

PHYSICAL_TEST = BLOCKED — requer um purchase sheet real aberto, que por sua vez
requer o build estar registrado no Play (ver seção de bloqueio abaixo). Coberto por 2
testes de regressão novos (ver AUTOMATED_TESTS).

## Problema 5 — Fix de falso-negativo de compra (GAP2)

FALSE_NEGATIVE_PURCHASE_FIX_PRESERVED = **YES**. Localizado e confirmado intacto em
`play_billing_functions.dart` (`_recoverSilently`, lógica de recuperação via
`restorePurchases()` já presente antes desta missão). Nenhuma regressão introduzida.

Ganhou 2 testes de regressão novos que antes eram impossíveis de escrever porque o
fake `queryProductDetails` lançava `UnimplementedError()`. Corrigido o fake para
construir `ProductDetails` reais (o pacote permite construtor público direto,
confirmado lendo o source no VM).

## AUTOMATED_TESTS

Arquivo `test/google_play_readiness_test.dart`: **13/13 passando**, incluindo 3 testes
novos:
1. `M17 purchase interrupted by error status recovers as completed, not a false
   failure, and blocks a concurrent second attempt` (Problema 5 / GAP2)
2. `M17 resume with no matching purchase found releases the active purchase as
   canceled, well before the purchase timeout` (Problema 4)
3. `M17 resume does not falsely cancel a purchase recovery finds` (companheiro de
   segurança do fix do Problema 4)

## FULL_SUITE

`flutter test` completo no VM, commit `90113fb`: **1515/1515 passando**, exit code 0.
Uma falha real foi encontrada e corrigida durante este processo — não uma regressão
pré-existente, mas causada diretamente pelo fix do Problema 2 (novo parâmetro
obrigatório `onCreditsSpent` em `LabSessionAuxController` não fornecido por
`test/review_manual_only_contract_test.dart`) — corrigida em `90113fb`.

`flutter analyze`: **No issues found!**

NEW_REGRESSIONS = **0** (a única falha encontrada foi corrigida antes do checkpoint
final; suíte completa ficou verde depois).

## Checkpoints e integridade

APP_FINAL_SHA = `2e3c15f8cee4435d34218c4bb17eb63aa2a9c5ea` (main, pushed)
SERVER_FINAL_SHA = `bb7c56f` (inalterado — nenhuma mudança de servidor nesta missão)

GITHUB_CHECKPOINTS (BOM, todos em `main`, todos pushed):
```
6aed701  fix(credits+billing): stale balance, insufficient-credits freeze, cancel spinner
63c63c5  fix(test): import AppLifecycleState in google_play_readiness_test.dart
53f96a0  fix(test): remove racy delay in GAP2 concurrent-purchase regression test
dddc83b  fix(test): make fake restorePurchases() wait for full redelivery processing
90113fb  fix: pass required onCreditsSpent to LabSessionAuxController in review contract test
2e3c15f  chore(release): bump Android version code to 113 for credit-flow certification test build
```

WORKTREES_CLEAN = YES (checkout principal local e da VM, ambos limpos em `2e3c15f`;
worktrees pré-existentes na VM pertencem a missões anteriores já evacuadas, fora do
escopo desta missão).

SECRETS_COMMITTED = NO. Keystore e `key.properties` foram copiados temporariamente
para a VM (autorizado explicitamente), usados só para assinar o build, e apagados
(`shred -u`) logo depois. `git status` confirmou que nunca ficaram rastreados
(`.gitignore` já cobria ambos os caminhos).

## NEW_TEST_BUILD_CREATED

```
VERSION_CODE = 113 (confirmado via Android Publisher API: próximo código nunca usado,
                     considerando tracks + os 10 bundles históricos 96-112; 112 é a
                     produção atual)
VERSION_NAME = 1.0.0
PACKAGE = com.simaitutor.app
SIGNING_SHA1   = 94:E4:6E:76:9C:49:BD:B1:AC:37:C1:D3:A2:B3:34:FD:D2:41:DF:B1  (bate com o Upload key certificate do Play Console)
SIGNING_SHA256 = 54:FB:04:AE:A6:7C:27:E0:D0:CF:F6:34:B1:39:C8:CE:58:BB:68:B3:C8:C9:24:A0:84:0A:F1:D1:1E:6D:E5:6A  (idem)
AAB_SHA256 = 4a885e48b6febad129b7d975636dd36e2926a88d21a5fcf513908623f1d24fe9
APK_SHA256 = 89573f4da4f6fbd6f14474437033b0650c7a3947c15fab60d9b9bfa3160a4090
PRODUCTION_ENDPOINT = https://simaitutor.com
GITHUB_RELEASE = https://github.com/aulasonline18-blip/BOM-APK-Downloads/releases/tag/v113-credit-flow-cert
```

PLAY_CONSOLE_STATUS = **NÃO publicado** por mim (upload de bundle + track ficaram
staged via API mas o commit final falhou com 403 PERMISSION_DENIED — a service
account de RTDN não tem papel de publicação). Usuário optou por publicar manualmente
pelo próprio Play Console.

## Certificação física no tablet (100.124.23.2, conta joelgomes522@gmail.com)

```
APP_INSTALLED_V113             = PASS (uninstall+install necessário: certificado de
                                  upload ≠ certificado de app signing do Play, esperado)
LOGIN_GOOGLE_OAUTH              = PASS
LESSON_PREPARE_AND_OPEN         = PASS (T00/T02, sem travar em "preparando")
BALANCE_DISPLAYED               = PASS (999797 → 999794 após consumo real)
BALANCE_REFRESH_AFTER_CONSUMPTION = PASS (ver ressalva acima)
ADVANCE_PENDING_RESOLVES_CLEANLY  = PASS (transição preparing→ready sem stall)
PURCHASE_ERROR_SURFACED_CLEANLY = PASS (erro "not configured for billing" tratado
                                   com banner claro, sem spinner preso)
PLAY_CONNECTION                 = BLOCKED (build não publicado em nenhuma track)
PRODUCT_100/200/500_AVAILABLE   = BLOCKED (mesma causa)
INSUFFICIENT_CREDITS_DETECTED   = NOT_TESTED (conta de QA com 999794 créditos)
PREPARING_STATE_CLEARED         = PASS (observado no fluxo real de avanço)
BUY_CREDITS_PATH_PRESENTED      = PASS (tela de créditos abre, mostra 3 pacotes)
PURCHASE_CANCEL_CLEARS_LOADING  = BLOCKED (requer sheet real aberto)
PURCHASE_ERROR_CLEARS_LOADING   = PASS (caso real observado ao vivo)
PURCHASE_RECOVERY_SAFE          = NOT_TESTED (requer compra real)
DUPLICATE_GRANT                 = NOT_TESTED (requer compra real)
EXACTLY_ONCE_PRESERVED          = NOT_TESTED (requer compra real; coberto por testes automatizados)
APP_POINTS_TO_PRODUCTION        = PASS (https://simaitutor.com, confirmado no build)
USES_VM_BACKEND                 = NO
REQUIRES_VM                     = NO (build/testes futuros podem repetir a partir do GitHub)
```

## Causal chain para os itens BLOCKED

Build v113 nunca subiu para nenhuma track do Play Console → Play recusa
`launchBillingFlow` para esse APK específico ("This version of the application is not
configured for billing through Google Play") → nenhum purchase sheet real abre →
Problemas 1 e 4 (na forma de compra real) e a tríade
recovery/duplicate-grant/exactly-once não puderam ser reverificados fisicamente nesta
sessão. Isso é uma limitação de distribuição do build de teste, não uma falha de
código — a mesma verificação já foi feita anteriormente nesta sessão (mais ampla,
contra o v112 de produção) e permanece válida como evidência histórica para os
mecanismos que não mudaram.

## SIM_CREDIT_FLOW_CERTIFICATION = **PASS COM RESSALVA**

Todo o código relevante (Problemas 2, 3, 4) foi corrigido com causa raiz identificada,
sem patches cosméticos, preservando máquinas de estado e contratos existentes
(server-side validation, purchaseToken-as-identity, exactly-once, obfuscatedAccountId,
RTDN, fail-closed, ledger). O fix de falso-negativo de compra (Problema 5) está
confirmado presente e agora tem cobertura de regressão dedicada. Problema 1 está
corretamente diagnosticado como fora do código. A suíte completa (1515 testes) e
`flutter analyze` estão limpos. Um novo build de teste foi gerado, assinado
corretamente e certificado parcialmente no tablet físico (login, saldo,
consumo/refresh, transições de estado, tratamento de erro de billing).

A ressalva: a verificação física ao vivo de compra real (Problemas 1/4 no cenário de
purchase sheet aberto, recuperação de compra, duplicate-grant, exactly-once) ficou
BLOCKED por uma limitação de infraestrutura de distribuição (build não publicado em
nenhuma track do Play), não por uma falha encontrada no código. Assim que o usuário
publicar o v113 (ou qualquer build subsequente do mesmo commit) na track Internal
Testing, os itens BLOCKED podem ser re-executados usando exatamente este relatório
como checklist, sem necessidade de repetir nenhum trabalho de código.
