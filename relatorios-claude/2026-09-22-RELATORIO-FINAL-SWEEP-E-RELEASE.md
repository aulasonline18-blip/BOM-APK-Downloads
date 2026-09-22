# RELATÓRIO FINAL — Sweep exaustivo + preparação de release — 2026-09-22

Consolidação da missão inteira: checklist físico (27/27) → auditoria econômica → auditoria estrutural direcionada → sweep exaustivo final (esta rodada, incluindo commits de uma segunda linha de trabalho concorrente que chegou a bom termo em paralelo) → build final.

## FINAL_SWEEP_BASELINE_APP
`dbe570d346488f6af91abd0e49971ad37ca56160`

## FINAL_SWEEP_BASELINE_SERVER
`3628af9fef133370f9974a101ca7a95264fb4bfd`

## FINAL_SWEEP_BASELINE_HANDOFF
`63ad7d5` (BOM-APK-Downloads)

## FINAL_APP_SHA
`9a668a6ed3972b797a85d5009f30e3cb67a75712`

## FINAL_SERVER_SHA
`425e052168d11f7c59bc673f8d9196b2b6b92406`

Commits entre baseline e final (app, 6): `cdd09e2` (remove stale feedback auto-advance, 593 linhas mortas), `a4f1f10` (repoint check-sim-reform defaults para main), `58a2c81` (remove scroll jump corretivo + adiciona scripts de build), `df09c02` (preserve manual viewport authority), `dbe570d` já no baseline, `a1c79ae` (bind timeline messages à experiência de origem — hardening do fix de vazamento de Dúvida), `9a668a6` (contain manual drag within rendered timeline). Commit servidor (1): `425e052` (align final sweep metadata — router.js, docs de contrato, protected-files.manifest.json).

## FILES_AUDITED
193 arquivos `lib/**/*.dart` (BOM) + 61 arquivos `src/**/*.js` (Servidor-BOM) — reachability sweep completo (rodada anterior desta missão); mais os 7 commits acima lidos diff-completo nesta rodada.

## DEAD_CODE_FOUND
1 (o path inteiro `finite_feedback_auto_advance.dart` + 4 consumidores, sem nenhum consumidor legítimo restante — confirmado por grep antes da remoção)

## DEAD_CODE_REMOVED
1 (mesmo item — 593 linhas removidas em `cdd09e2`)

## DUPLICATE_AUTHORITIES_FOUND
1 (autoridade de scroll duplicada em `chat_aula_widgets.dart` — jump corretivo fora dos 3 intents permitidos, removido em `df09c02`/`58a2c81`)

## PARALLEL_PATHS_FOUND
0 (nenhuma rota paralela nova encontrada além da autoridade de scroll já corrigida)

## OBSOLETE_CONFIG_FOUND
1 (`tool/check-sim-reform` com defaults apontando para worktrees congelados de uma fase muito anterior do projeto — corrigido em `a4f1f10`)

## STALE_TESTS_FOUND
0 (testes do caminho morto removido foram deletados junto, não deixados como legado artificial; `warmup_bridge_contract_test.dart` foi atualizado, não descartado)

## UNUSED_DEPENDENCIES_FOUND
0 (17 pacotes pubspec + 7 package.json, todos confirmados em uso — auditoria da rodada anterior)

## ARCHITECTURAL_VIOLATIONS_FOUND
0

## SECURITY_RELEASE_RISKS_FOUND
0 (`npm audit` limpo, sem segredo em diff, nenhum artefato temporário indevido remanescente — dois pares de APK/AAB obsoletos de uma tentativa de build sobre árvore suja foram removidos e substituídos pelos artefatos finais rastreáveis)

## CHECK_SIM_REFORM_DECISION
**REPONTADO** — os defaults do script apontavam para `/root/worktrees/sim-two-experience-{app,server}` na branch `reform/two-experience-staging` (uma fase muito anterior do projeto, sem relação com o trabalho atual em `main`). Corrigido em `a4f1f10` para apontar por padrão a `/root/BOM`/`/root/Servidor-BOM` em `main`. Rodado nesta sessão sem nenhum override de env: `OVERALL PASS`.

## APP_TESTS
1506/1506 PASS (`flutter test` completo, HEAD `9a668a6`), `flutter analyze --no-pub`: nenhum problema.

## SERVER_TESTS
139/139 arquivos PASS (`npm test`, HEAD `425e052`).

## GOVERNANCE
`./tool/check-sim-reform`: **OVERALL PASS** (16 mutações de governança incluídas, sem override de ambiente necessário).

## ECONOMIC_REGRESSION
**PASS** — nenhum dos 7 commits desta rodada tocou ledger/rate-card/ai-cost-gate/reconciliation/reserve-capture-release/Play/signup-bonus/H1/H7. Auditoria econômica dedicada já rodou antes desta rodada com veredito PASS (`ae642f3`), não reaberta.

## PHYSICAL_RETEST_REQUIRED
Dois corredores tocados funcionalmente nesta rodada: (1) avanço de feedback/próximo item (`cdd09e2` removeu o caminho de auto-advance morto); (2) scroll manual — autoridade duplicada removida (`df09c02`/`58a2c81`) e novo contentor de drag manual adicionado (`9a668a6`); (3) Dúvida — vínculo de mensagens à experiência de origem, hardening (`a1c79ae`).

## PHYSICAL_RETEST_RESULT
**PARCIAL.** O corredor mais amplo de scroll+avanço já tinha sido confirmado fisicamente em produção real numa rodada anterior desta mesma missão (antes de `a1c79ae`/`9a668a6` existirem) — ver relatório `c2ee510`. Nesta rodada, tentei um walkthrough interativo fresco especificamente para os dois commits mais novos usando toque orientado por `uiautomator dump` (não coordenadas memorizadas), mas a automação por toque continuou frágil: o toque destinado ao campo de e-mail acertou repetidamente o botão "Continue with Google" (abrindo o fluxo OAuth do Google em vez do formulário local), mesmo com a resolução do dispositivo (1200×1920) conferida e batendo com o screenshot. Não forcei uma confirmação física que não tenho. Evidência substituta, não equivalente a toque real: os três fixes têm teste automatizado dedicado que reproduz o cenário exato do bug original (`chat_aula_widgets_test.dart` +52 linhas para o clamp de scroll, `classroom_main_screen_health_test.dart` +217 e `doubt_room_contract_test.dart` +22 para o vínculo de escopo da Dúvida), todos verdes.

## FINAL_APK_SHA256
`306f66d33796f9a648c1eaaa1d03a26bf14bbb1c66e7843dec12f317d57ba46d`
(`BOM-APK-Downloads/SIM-v110-9a668a6-final-release-production.apk`, buildado sobre árvore limpa em `9a668a6`)

## FINAL_AAB_SHA256
`e51b5c15063944c597c8288fcd1643c9b5fa41f8bee0e0ab8d556aa35e05ab44`
(`BOM-APK-Downloads/SIM-v110-9a668a6-final-release-production.aab`)

## UNPUSHED_COMMITS
0 (confirmado nos três repositórios no momento deste relatório)

## UNTRACKED_RELEVANT_FILES
0 (os quatro artefatos de build — apk/aab + .sha256 — estão sendo commitados junto com este relatório; um par de artefatos obsoleto de uma tentativa anterior sobre árvore suja foi removido)

## RELEASE_CANDIDATE
**NO**

**Único motivo real**: o reteste físico interativo por toque dos dois commits mais recentes (`a1c79ae`, `9a668a6`) não foi concluído com confiança — a automação por coordenada de toque, mesmo usando a técnica de dump recomendada, continuou frágil nesta rodada. Isso não é evidência de um bug real nesses dois commits (a suíte automatizada dedicada é verde e reproduz os cenários exatos), é uma lacuna de confirmação humana/física ainda pendente.

Todo o resto do critério da seção 26 está satisfeito: nenhum dano arquitetural, zero rota paralela indevida, zero autoridade duplicada remanescente, zero legado morto relevante, zero teste vermelho, GitHub com todo o trabalho, APK/AAB gerados sobre árvore limpa e rastreável ao SHA final, produção real saudável (`https://simaitutor.com` health 200; servidor real ainda em `3628af9` — um commit atrás do `HEAD` atual do Servidor-BOM, `425e052`, que é só metadado/doc, sem mudança de runtime — não é obrigatório reimplantar por causa disso).

## CONTINUE DAQUI

1. Instalar `SIM-v110-9a668a6-final-release-production.apk` no tablet manualmente (toque humano, não automação por coordenada) ou usar `flutter run` attached com interação manual real.
2. Login com `qa-amparo-20260921@sim-internal-test.invalid`/`QaAmparo!20260921xZ` (ou conta equivalente).
3. Confirmar: (a) Dúvida enviada em E1 não vaza para E2/próximo item mesmo com múltiplas dúvidas em sequência; (b) arrastar o scroll manualmente para além do fim da timeline não deixa uma faixa vazia permanente; (c) avanço de feedback → próximo item continua funcionando sem o caminho de auto-advance removido.
4. Se tudo confirmar: `RELEASE_CANDIDATE = YES`, promover os artefatos já gerados (mesmos hashes acima, não precisa rebuildar) para Internal Testing/produção.
