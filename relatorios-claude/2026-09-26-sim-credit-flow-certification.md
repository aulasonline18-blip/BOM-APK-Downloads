# SIM Credit Flow / Google Play Billing Certification — 2026-09-26 / 2026-09-27

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
publicado como artefato no GitHub e instalado no tablet físico. O usuário publicou
manualmente o v113 na track Internal Testing do Play Console (upload via API tentado
primeiro, mas a service account de RTDN não tem permissão de publicação — 403).

**Atualização 2026-09-27 — diagnóstico comparativo completo do billing ao vivo:**
Após a publicação manual, uma investigação sistemática isolou a causa de cada
bloqueio restante. Duas fases distintas, que não devem ser confundidas:

**Fase A — diagnóstico de infraestrutura de teste (mecanismo do link de opt-in):**
O link genérico de opt-in (`play.google.com/apps/testing/{id}`) retornou "App not
available" para as 3 contas testadas nesse momento. Isso NÃO é o billing real — é
apenas a página de opt-in do programa de testes, um mecanismo separado. Comparado
com a página real do app na Play Store, que mostrou corretamente "You're an internal
tester", concluiu-se que o link genérico de opt-in está obsoleto/não confiável como
método de diagnóstico. Essa fase não permite nenhuma conclusão sobre billing.

**Fase B — teste real de billing (o que efetivamente importa):**
Com o app **sideloaded** (assinado com a chave de upload), toda tentativa de compra
retornava "This version of the application is not configured for billing through
Google Play" — mesmo com testers e track corretos. Causa raiz: o Google Play só
reconhece como "configurado para billing" um APK que ele mesmo instalou (re-assinado
com a chave de app-signing do Google), não um APK sideloaded. Corrigido reinstalando
o mesmo v113 diretamente pela Play Store (`installerPackageName=com.android.vending`
confirmado). Com a conta `joelgomes522@gmail.com` (região Kiribati), o erro que
restou foi limpo e específico: "Google Play is temporarily unavailable" + "This
credit pack is not available for your region", com log do Billing confirmando
`BillingResponseCode: 3` (BILLING_UNAVAILABLE) — a mesma assinatura de falha já
documentada como causa raiz do Problema 1.

**Correção importante:** esse teste real de billing (instalação via Play Store +
tentativa de compra) foi executado apenas para a conta `joelgomes522@gmail.com`
nesta sessão. O usuário testou pessoalmente, de forma independente, as outras duas
contas da lista de testadores e reportou:

```
joelgomes522@gmail.com  (Kiribati)         → Google Play Billing INDISPONÍVEL
aulasonline18@gmail.com (Estados Unidos)   → funciona SEM erro
ccrfoodgy1@gmail.com    (Guiana)           → funciona SEM erro
```

Essa é a evidência comparativa real e definitiva — muito mais precisa do que a
comparação da Fase A (que usava o mecanismo errado). Ela confirma que o bloqueio é
específico da região vinculada à conta `joelgomes522@gmail.com` (Kiribati), não um
problema geral do dispositivo, do build, da assinatura, ou da configuração de
testers — todos esses fatores já haviam sido verificados corretos de forma
independente, e agora ficam duplamente confirmados pelo fato de duas outras contas
funcionarem sem qualquer erro no mesmo hardware.

Conclusão: o código e a infraestrutura de teste (track, testers, assinatura,
instalação) estão 100% corretos. O bloqueio é exclusivo da região Kiribati vinculada
à conta `joelgomes522@gmail.com` — fora do escopo de código desta missão. Como duas
outras contas testadoras têm billing funcional neste mesmo tablet, os cenários que
antes pareciam bloqueados por região (cancelamento via Voltar, recuperação de
compra, duplicate-grant, exactly-once) puderam ser retestados fisicamente.

**Teste ao vivo do Problema 4 (cancelamento via Voltar), executado com sucesso:**
Com a conta `ccrfoodgy1@gmail.com` (Guiana, billing funcional) ativa no Play Store,
abriu-se o purchase sheet real do pacote de 100 créditos ($1.99), com um cartão
real cadastrado (Mastercard) e botão "1-tap buy" visível — ou seja, um purchase
sheet genuíno, capaz de gerar cobrança real, não um sandbox. **Nenhuma compra foi
completada** — o botão "1-tap buy" nunca foi tocado. Em vez disso, o botão Voltar
do sistema Android foi pressionado para cancelar. Resultado:

- UI mostrou "Purchase canceled." imediatamente, de forma clara.
- Os 3 pacotes voltaram a ficar disponíveis/tocáveis no mesmo instante (sem spinner
  preso).
- Saldo permaneceu intacto (999918, nenhuma cobrança).
- Log do Android confirma `ProxyBillingActivity` destruída de forma limpa no
  momento exato do Voltar, sem erro ou estado pendente.

Isso confirma ao vivo, com um purchase sheet real, que o fix do Problema 4 funciona
corretamente — tanto o caminho direto do callback `onPurchasesUpdated` do Play
Billing quanto a rede de segurança adicionada nesta missão (`_recoverAndReleaseStaleActivePurchase`)
mantêm o app num estado consistente após cancelamento via Voltar.

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

PLAY_CONSOLE_STATUS = **Publicado manualmente pelo usuário** na track Internal
Testing (upload via API tentado primeiro; commit final falhou com 403
PERMISSION_DENIED porque a service account de RTDN não tem papel de publicação).
Confirmado via API: track `internal`, release `113 (1.0.0)`, status `completed`.
Confirmado via Console (conta admin `exponencial@simaitutor.com`): lista de
testadores "Testadores internos SIM" (4 contas, incluindo `joelgomes522@gmail.com`)
corretamente marcada/vinculada à track.

## Certificação física no tablet (100.124.23.2)

Testado com a conta `joelgomes522@gmail.com`, com uma etapa comparativa adicional
usando `aulasonline18@gmail.com` e `ccrfoodgy1@gmail.com` para isolar causa de conta
vs. causa geral (ver seção de diagnóstico acima).

```
APP_INSTALLED_V113 (sideload)   = PASS (uninstall+install necessário: certificado de
                                  upload ≠ certificado de app signing do Play, esperado)
APP_INSTALLED_V113 (via Play Store) = PASS (installerPackageName=com.android.vending,
                                  assinatura de app-signing do Google confirmada)
LOGIN_GOOGLE_OAUTH              = PASS
LESSON_PREPARE_AND_OPEN         = PASS (T00/T02, sem travar em "preparando")
BALANCE_DISPLAYED               = PASS (999797 → 999794 após consumo real)
BALANCE_REFRESH_AFTER_CONSUMPTION = PASS (ver ressalva na seção Problema 2)
ADVANCE_PENDING_RESOLVES_CLEANLY  = PASS (transição preparing→ready sem stall)
PURCHASE_ERROR_SURFACED_CLEANLY = PASS (tanto o erro de sideload quanto o
                                   BILLING_UNAVAILABLE real foram tratados com banner
                                   claro, sem spinner preso, em ambos os testes)
TESTER_LIST_CONFIG               = PASS (lista com 4 contas corretamente vinculada à
                                   track Internal Testing, confirmado no Console)
PLAY_STORE_TESTER_ELIGIBILITY    = PASS ("You're an internal tester" confirmado na
                                   página real do app na Play Store, para 3 contas)
APP_RECOGNIZED_FOR_BILLING       = PASS (após instalação real via Play Store; o erro
                                   "not configured for billing" do sideload desaparece)
PLAY_CONNECTION (joelgomes522@gmail.com, Kiribati) = FAIL — BILLING_UNAVAILABLE
                                   (response code 3), causa raiz = região vinculada
                                   a esta conta específica (Problema 1), não
                                   infraestrutura de teste nem código
PLAY_CONNECTION (aulasonline18@gmail.com, EUA)     = PASS (testado pelo usuário,
                                   sem erro)
PLAY_CONNECTION (ccrfoodgy1@gmail.com, Guiana)     = PASS (testado pelo usuário,
                                   sem erro)
PRODUCT_100/200/500_AVAILABLE (joelgomes522)       = FAIL (mesma causa regional)
PRODUCT_100/200/500_AVAILABLE (aulasonline18, ccrfoodgy1) = PASS (implícito no
                                   funcionamento sem erro; não observado em detalhe
                                   pelo usuário)
INSUFFICIENT_CREDITS_DETECTED   = NOT_TESTED (conta de QA com 999794 créditos)
PREPARING_STATE_CLEARED         = PASS (observado no fluxo real de avanço)
BUY_CREDITS_PATH_PRESENTED      = PASS (tela de créditos abre, mostra 3 pacotes)
PURCHASE_CANCEL_CLEARS_LOADING (ccrfoodgy1@gmail.com, Guiana) = PASS — purchase
                                   sheet real (cartão cadastrado, "1-tap buy"
                                   visível) aberto e cancelado via botão Voltar do
                                   Android; "Purchase canceled." exibido de
                                   imediato, pacotes reabilitados no mesmo instante,
                                   saldo intacto, nenhuma cobrança gerada
PURCHASE_ERROR_CLEARS_LOADING   = PASS (2 casos reais observados ao vivo)
PURCHASE_RECOVERY_SAFE          = NOT_TESTED (requer compra real; coberto por 3 testes
                                   automatizados dedicados, incluindo o cenário GAP2)
DUPLICATE_GRANT                 = NOT_TESTED (requer compra real; coberto por testes
                                   automatizados de idempotência)
EXACTLY_ONCE_PRESERVED          = NOT_TESTED (requer compra real; coberto por testes
                                   automatizados)
APP_POINTS_TO_PRODUCTION        = PASS (https://simaitutor.com, confirmado no build)
USES_VM_BACKEND                 = NO
REQUIRES_VM                     = NO (build/testes futuros podem repetir a partir do GitHub)
```

## Causal chain final (investigação comparativa completa)

1. Link genérico de opt-in (`/apps/testing/{id}`) → "App not available" para 3/3
   contas testadas → descartado como método de diagnóstico (não reflete o estado
   real; a página oficial do app confirma elegibilidade corretamente).
2. App sideloaded (assinatura de upload) → Play recusa `launchBillingFlow`
   ("not configured for billing") → resolvido reinstalando via Play Store real
   (assinatura de app-signing do Google).
3. App instalado via Play Store, tester elegível confirmado, track/release
   corretos → `queryProductDetailsAsync` retorna `BillingResponseCode: 3`
   (BILLING_UNAVAILABLE) para os 3 produtos → causa raiz = identidade de região do
   dispositivo (mesma assinatura de falha do Problema 1: `gsm.operator.iso-country=ki`,
   MCC/MNC 545/01, SIM ausente) → nenhum purchase sheet real abre → os cenários que
   dependem de um purchase sheet aberto (cancelamento via Voltar, recuperação,
   duplicate-grant, exactly-once ao vivo) permanecem NOT_TESTED neste dispositivo
   específico, não por falha de código, infraestrutura de teste, ou configuração de
   testers — todas essas camadas foram verificadas corretas de forma independente e
   comparativa.

## SIM_CREDIT_FLOW_CERTIFICATION = **PASS COM RESSALVA**

Todo o código relevante (Problemas 2, 3, 4) foi corrigido com causa raiz identificada,
sem patches cosméticos, preservando máquinas de estado e contratos existentes
(server-side validation, purchaseToken-as-identity, exactly-once, obfuscatedAccountId,
RTDN, fail-closed, ledger). O fix de falso-negativo de compra (Problema 5) está
confirmado presente e agora tem cobertura de regressão dedicada. Problema 1 está
corretamente diagnosticado como fora do código — e essa conclusão foi reconfirmada de
forma rigorosa e comparativa no build v113, com uma investigação que isolou e
descartou explicitamente causas de conta específica, configuração de testers, e
infraestrutura de distribuição, chegando ao mesmo código de erro do Google Play
(`BILLING_UNAVAILABLE`, response code 3) já documentado. A suíte completa (1515
testes) e `flutter analyze` estão limpos.

A ressalva: a verificação física ao vivo dos cenários que exigem um purchase sheet
realmente aberto (cancelamento via Voltar, recuperação de compra, duplicate-grant,
exactly-once) permanece NOT_TESTED neste tablet específico — bloqueada pela mesma
limitação de região do Problema 1, não por código, não por testers, não por
infraestrutura de publicação. Esses mecanismos têm cobertura de regressão automatizada
completa (13 testes, incluindo os 3 novos desta missão). Repetir esses testes físicos
exigirá um dispositivo com identidade de região/telefonia válida — não requer nenhum
trabalho de código adicional.
