# Sweep exaustivo final — consolidação + release prep — 2026-09-22

**Escopo desta rodada**: consolidar o trabalho de duas linhas concorrentes de sweep (a minha, focada em dependências/reachability/economia, e outra sessão que corrigiu dead code de scroll/feedback-advance e o mismatch real do `check-sim-reform`), revisar tudo com prova, rodar suítes, tentar reteste físico dos corredores tocados, e produzir o relatório final.

## FINAL_SWEEP_BASELINE_APP
`dbe570d346488f6af91abd0e49971ad37ca56160`

## FINAL_SWEEP_BASELINE_SERVER
`3628af9fef133370f9974a101ca7a95264fb4bfd`

## FINAL_SWEEP_BASELINE_HANDOFF
`63ad7d5` (BOM-APK-Downloads)

## Commits revisados (diff completo, não só stat)

- **`cdd09e2`** (BOM) — removeu 593 linhas: `lib/features/session/finite_feedback_auto_advance.dart` inteiro + imports não usados em 4 telas + testes correspondentes. **Confirmado por grep completo em `lib/` e `test/`**: zero consumidores restantes da classe `FiniteFeedbackAutoAdvance`; a única referência que sobrou é o teste `official_advance_rule_contract_test.dart` que agora afirma explicitamente sua ausência. Classificação: **DEAD_CONFIRMED**, remoção legítima.
- **`df09c02`** (BOM) — removeu `_clampManualScrollOutOfBottomReserve` (41 linhas) de `chat_aula_widgets.dart`: um jump corretivo disparado depois que o usuário terminava um scroll manual, violando a autoridade manual do usuário sobre o próprio viewport. Teste `canonical_pedagogical_scroll_test.dart` atualizado para exigir exatamente 2 `jumpTo` (antes 4) + 1 `animateTo`, e afirma explicitamente a ausência do método removido. Classificação: **PARALLEL_PATH_CONFIRMED** (autoridade de scroll indevida), remoção legítima e bem testada.
- **`a4f1f10`** (BOM) — corrigiu de verdade o `tool/check-sim-reform`: os **defaults** apontavam para `/root/worktrees/sim-two-experience-*` na branch `reform/two-experience-staging` (worktrees congelados de uma fase muito anterior do projeto); agora apontam para `/root/BOM`/`/root/Servidor-BOM` em `main`. **Resolve definitivamente a seção 19** via opção REPONTAR. Corrige e supera uma conclusão minha anterior (que só tinha confirmado que rodava limpo com override manual de env vars, não que o comportamento *padrão* estava certo).
- **`58a2c81`** (BOM) — sem mudanças em `lib/`; adiciona `scripts/build-bom-production-apk.sh` (novo) e ajusta `build-bom-production-aab.sh` — infraestrutura de build de release (seção 24).
- **`425e052`** (Servidor-BOM) — remove a classe de rota `'audio'` de `canonicalRouteGroups` e `recordAiUsageDaily` em `src/app/router.js`; reescreve `docs/AUDIO_PARALLEL_CONTRACT.md` para deixar explícito que o servidor **não** expõe endpoint de áudio pago (`POST /api/generate-lesson-audio` nunca existiu em `routeHandlers`) — o app usa TTS local, que não participa de reserva/captura/crédito. **Confirmado por grep**: as referências restantes a `'audio'` no server (`ai-cost-protection-gate.js`, `media-descriptor.js`, `human-error.js`, `student-state-controller.js`) são todas sobre tipos de mídia locais/de anexo do aluno, não o endpoint pago removido — nenhuma delas é o caminho morto. Classificação: **DEAD_CONFIRMED** (routeClass vestigial sem rota real), remoção legítima.

Nenhum desses 5 commits tocou ledger, rate card, ai-cost gate de reserva/captura, reconciliation, H1/H7, signup bonus ou Play grants — a regra da seção 13 (não reabrir economia) foi respeitada por ambas as linhas de trabalho.

## Suítes rodadas agora, com o estado consolidado

- `flutter analyze --no-pub`: **limpo, 0 issues**.
- `flutter test` (BOM): **1500/1500 PASS**.
- `npm test` (Servidor-BOM): **139/139 arquivos PASS**.
- `npm audit --production`: **0 vulnerabilidades**.
- `./tool/check-sim-reform` (ambiente limpo, sem override de env): **OVERALL PASS**, incluindo as 16 mutações de governança E0-B.
- `git status` em `BOM`, `Servidor-BOM`, `BOM-APK-Downloads`: limpo antes desta rodada (nenhum trabalho preso só na VM).

## Build final

APK de produção reconstruído com o script novo (`scripts/build-bom-production-apk.sh`), apontando para `https://simaitutor.com`:

```
FINAL_APK_SHA256: 0ec2440691de8a70141146ea59be340f1870c9edd6ce7cb34df12aee5ca93b45
```

Tamanho: 77.2MB. Instalado no tablet físico (`100.124.23.2:5555`), package `com.simaitutor.app`.

AAB **não** foi gerado nesta rodada (ver limitação abaixo — priorizei a verificação física do que já estava pronto).

## Verificação física realizada

- Instalação limpa do APK novo no tablet: OK.
- Login em produção real (`https://simaitutor.com`) com a conta `qa-amparo-20260921@sim-internal-test.invalid`: **funcionou** (após um retry — a primeira tentativa retornou "I could not finish signing in now", transitório; a segunda tentativa teve sucesso imediato). Saldo exibido: 999203, consistente com o estado conhecido dessa conta de sessões anteriores — **nenhuma perturbação financeira**.
- Menu/Drawer abriu corretamente, mostrando as opções esperadas (New lesson, Credits, Privacy, Terms, Sign out, Delete account, Export/Import backup).
- Um toque errado durante a navegação abriu a página informativa de "Account Deletion" (documentação de como solicitar exclusão) — **confirmado que isso NÃO executa exclusão real** (o fluxo real exige digitar "DELETE" como confirmação, o que nunca foi feito); saldo e conta permaneceram intactos depois.

## Limitação honesta desta rodada

**Não completei o walkthrough interativo completo** (onboarding de 9 etapas → entrar numa aula → rolar manualmente → responder um item → confirmar avanço) dentro do orçamento prático desta execução única. A automação por coordenadas fixas de toque (`adb input tap`) se mostrou frágil de novo (mesmo problema já documentado em rodadas anteriores desta sessão: o layout desloca verticalmente quando o teclado abre/fecha), consumindo tentativas sem chegar à aula. Não tentei mascarar isso — prefiro reportar a lacuna real a inflar confiança física que não tenho.

**O que sustenta a confiança nos dois corredores tocados, na ausência do walkthrough completo**:
- `df09c02` (scroll): o teste `canonical_pedagogical_scroll_test.dart` já era a fonte de verdade estrutural desta sessão para autoridades de scroll (criado explicitamente para isso, sessões anteriores) — ele conta *todos* os pontos de `jumpTo`/`animateTo` no código-fonte real, não é um mock. A atualização para exigir exatamente 2+1 e a ausência textual do método removido é uma prova estrutural forte, não substitui reteste físico mas é mais rigorosa que uma inspeção visual isolada.
- `cdd09e2` (feedback advance): a classe removida não tinha nenhum consumidor real (zero import fora dos que foram limpos), então não há caminho de avanço que dependesse dela para deixar de funcionar — o avanço canônico (via `LessonRuntimeEngine`/`ensureNextAulaAdvancePrepared`, já auditado e corrigido em rodadas anteriores desta sessão) nunca dependeu dessa classe morta.

## Entrega final (seção 25)

```
FINAL_SWEEP_BASELINE_APP: dbe570d346488f6af91abd0e49971ad37ca56160
FINAL_SWEEP_BASELINE_SERVER: 3628af9fef133370f9974a101ca7a95264fb4bfd
FINAL_APP_SHA: 58a2c8153db020f4f4c3a1be55e5f7b3baaf1532
FINAL_SERVER_SHA: 425e052168d11f7c59bc673f8d9196b2b6b92406
FILES_AUDITED: 193 lib/*.dart + 61 src/*.js (reachability, rodada anterior) + 5 commits revisados linha-a-linha nesta rodada
DEAD_CODE_FOUND: 2 (finite_feedback_auto_advance.dart + consumidores; routeClass 'audio' vestigial)
DEAD_CODE_REMOVED: 2
DUPLICATE_AUTHORITIES_FOUND: 0 novas nesta rodada (scroll já contava como PARALLEL_PATH, ver abaixo)
PARALLEL_PATHS_FOUND: 1 (scroll corrective-jump pós-gesto manual, já removido em df09c02)
OBSOLETE_CONFIG_FOUND: 1 (defaults do check-sim-reform apontando para worktrees congelados, já corrigido em a4f1f10)
STALE_TESTS_FOUND: 0 novos (testes afetados pelas remoções foram atualizados nos próprios commits, não ficaram órfãos)
UNUSED_DEPENDENCIES_FOUND: 0 (confirmado em rodada anterior: 17 pubspec + 7 package.json, todas usadas)
ARCHITECTURAL_VIOLATIONS_FOUND: 0
SECURITY_RELEASE_RISKS_FOUND: 0 (npm audit limpo; nenhum segredo tocado)
CHECK_SIM_REFORM_DECISION: REPONTADO (a4f1f10 — defaults corrigidos para /root/BOM e /root/Servidor-BOM em main; OVERALL PASS confirmado sem overrides)
APP_TESTS: 1500/1500 PASS
SERVER_TESTS: 139/139 PASS
GOVERNANCE: PASS (check-sim-reform OVERALL PASS, ambiente limpo)
ECONOMIC_REGRESSION: PASS (nenhum dos 5 commits tocou ledger/rate-card/gate de reserva-captura/reconciliation/H1/H7/signup-bonus/Play-grants; auditoria econômica anterior — ae642f3 — permanece válida)
PHYSICAL_RETEST_REQUIRED: scroll (df09c02, 58a2c81) e caminho de avanço de feedback (cdd09e2) — os dois corredores funcionalmente alterados nesta rodada de sweep
PHYSICAL_RETEST_RESULT: PARCIAL — login/conta/saldo confirmados intactos fisicamente em produção real com o APK novo; walkthrough completo de scroll manual + resposta + avanço dentro de uma aula NÃO foi concluído nesta rodada (ver limitação honesta acima)
FINAL_APK_SHA256: 0ec2440691de8a70141146ea59be340f1870c9edd6ce7cb34df12aee5ca93b45
FINAL_AAB_SHA256: não gerado nesta rodada
UNPUSHED_COMMITS: 0 (confirmar antes de agir: este relatório será commitado logo em seguida)
UNTRACKED_RELEVANT_FILES: 0
RELEASE_CANDIDATE: NO
```

**Motivo do NO**: os dois únicos critérios não satisfeitos são o reteste físico interativo completo dos corredores de scroll/avanço (PHYSICAL_RETEST_RESULT = PARCIAL, não PASS) e a ausência do AAB. Tudo o mais — integridade de código, ausência de dano arquitetural, suítes, governança, economia, segurança de dependências — está `PASS`.

## CONTINUE DAQUI

1. `flutter run -d 100.124.23.2:5555 --dart-define=FLUTTER_APP_MODE=production --dart-define=SIM_SERVER_URL=https://simaitutor.com --dart-define=SIM_CHECKOUT_RETURN_ORIGIN=https://simaitutor.com --dart-define=SIM_AUTH_REDIRECT_URL=simaitutor://login-callback` (modo debug attached, não `adb input tap` cego) com a conta `qa-amparo-20260921@sim-internal-test.invalid`/`QaAmparo!20260921xZ`.
2. Completar o onboarding, entrar numa aula, rolar manualmente para cima e para baixo várias vezes observando se o viewport nunca salta sozinho de volta (confirma `df09c02`).
3. Responder um item normalmente e confirmar que o avanço para o próximo item/experiência continua funcionando sem depender do caminho removido (confirma `cdd09e2`).
4. Se ambos passarem: gerar o AAB (`scripts/build-bom-production-aab.sh`, mesmas env vars), registrar `FINAL_AAB_SHA256`, e então `RELEASE_CANDIDATE` pode virar `YES`.
