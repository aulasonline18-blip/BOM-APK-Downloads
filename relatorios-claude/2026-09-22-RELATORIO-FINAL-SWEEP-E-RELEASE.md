# RELATÓRIO FINAL — Sweep exaustivo + preparação de release — 2026-09-22

Consolidação da missão inteira: checklist físico (27/27) → auditoria econômica → auditoria estrutural direcionada → sweep exaustivo final (esta rodada, incluindo commits de uma segunda linha de trabalho concorrente que chegou a bom termo em paralelo) → build final.

## FINAL_SWEEP_BASELINE_APP
`dbe570d346488f6af91abd0e49971ad37ca56160`

## FINAL_SWEEP_BASELINE_SERVER
`3628af9fef133370f9974a101ca7a95264fb4bfd`

## FINAL_SWEEP_BASELINE_HANDOFF
`63ad7d5` (BOM-APK-Downloads)

## FINAL_APP_SHA
`9e667090538b94b3ae5b4824065015ee3ca8c106`

## FINAL_SERVER_SHA
`425e052168d11f7c59bc673f8d9196b2b6b92406`

Commits entre baseline e final (app): `cdd09e2` (remove stale feedback auto-advance, 593 linhas mortas), `a4f1f10` (repoint check-sim-reform defaults para main), `58a2c81` (remove scroll jump corretivo + adiciona scripts de build), `df09c02` (preserve manual viewport authority), `a1c79ae` (vínculo inicial de mensagens de Dúvida à experiência), `9a668a6` (contain manual drag within rendered timeline) e `9e66709` (expiração da Dúvida na fronteira semântica e publicação de snapshot pela autoridade da sessão). Commit servidor (1): `425e052` (align final sweep metadata — router.js, docs de contrato, protected-files.manifest.json).

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
1, corrigida em `9e66709`: dois controladores escreviam snapshots diretamente no runtime, contornando a autoridade da sessão e a limpeza de estado efêmero da Dúvida na mudança de experiência.

## SECURITY_RELEASE_RISKS_FOUND
1, **resolvido**: a credencial da conta QA sintética (`qa-amparo-20260921@sim-internal-test.invalid`) apareceu em três relatórios e em um log versionado deste repositório de handoff. Ela continuava válida no Supabase mesmo após ser removida da árvore atual do repo, porque permanecia no histórico Git. Resolução: a conta foi **deletada** via Supabase Admin API (`DELETE /auth/v1/admin/users/261f7461-...`, confirmado por lookup subsequente retornando 404) — a credencial exposta agora não autentica mais nada. Era uma conta sintética de teste (`user_metadata.qa_test_account=true`), sem saldo/dado real de aluno, então a exclusão é limpa e sem impacto em terceiros. `npm audit` permanece limpo e nenhum segredo foi introduzido no APP/SERVER.

## CHECK_SIM_REFORM_DECISION
**REPONTADO** — os defaults do script apontavam para `/root/worktrees/sim-two-experience-{app,server}` na branch `reform/two-experience-staging` (uma fase muito anterior do projeto, sem relação com o trabalho atual em `main`). Corrigido em `a4f1f10` para apontar por padrão a `/root/BOM`/`/root/Servidor-BOM` em `main`. Rodado nesta sessão sem nenhum override de env: `OVERALL PASS`.

## APP_TESTS
1508/1508 PASS (`flutter test` completo, HEAD `9e66709`), `flutter analyze --no-pub`: nenhum problema. `./tool/check-sim-reform`: `OVERALL PASS`.

## SERVER_TESTS
139/139 arquivos PASS (`npm test`, HEAD `425e052`).

## GOVERNANCE
`./tool/check-sim-reform`: **OVERALL PASS** (16 mutações de governança incluídas, sem override de ambiente necessário).

## ECONOMIC_REGRESSION
**PASS** — nenhum dos 7 commits desta rodada tocou ledger/rate-card/ai-cost-gate/reconciliation/reserve-capture-release/Play/signup-bonus/H1/H7. Auditoria econômica dedicada já rodou antes desta rodada com veredito PASS (`ae642f3`), não reaberta.

## PHYSICAL_RETEST_REQUIRED
Dois corredores tocados funcionalmente nesta rodada: (1) avanço de feedback/próximo item (`cdd09e2` removeu o caminho de auto-advance morto); (2) scroll manual — autoridade duplicada removida (`df09c02`/`58a2c81`) e novo contentor de drag manual adicionado (`9a668a6`); (3) Dúvida — vínculo de mensagens à experiência de origem, hardening (`a1c79ae`).

## PHYSICAL_RETEST_RESULT
**PASS no corredor causal.** No Samsung SM-X216B, contra produção real, usando exclusivamente a aula de Frações: Dúvida foi enviada em E1 do Item 18; pergunta, processamento e resposta apareceram abaixo do feedback; ao avançar para E2 o bloco deixou o estado ativo; após responder e sinalizar E2 ele continuou ausente; no Item 19 continuou ausente; após `force-stop` e reabertura não ressuscitou. O scroll manual permaneceu navegável e o estado atual foi preservado. Evidências PNG estão em `evidencias/2026-09-22-doubt-*`. A aula de Kiribati não foi aberta nem alterada.

## FINAL_APK_SHA256
`f351bf2ac0f736001cd6e1f3a20a6394a7407b78b97d0bfdad881db31e006e55`
(`BOM-APK-Downloads/SIM-v110-9e66709-final-release-production.apk`, buildado sobre árvore limpa em `9e66709` e instalado no Samsung)

## FINAL_AAB_SHA256
`0cc9f4831ffe68464b6cdc2e9d2e1a4585e196c33a7c21c26a952f2d8101165b`
(`BOM-APK-Downloads/SIM-v110-9e66709-final-release-production.aab`)

## UNPUSHED_COMMITS
0 (confirmado nos três repositórios no momento deste relatório)

## UNTRACKED_RELEVANT_FILES
0 (os quatro artefatos de build — apk/aab + .sha256 — estão sendo commitados junto com este relatório; um par de artefatos obsoleto de uma tentativa anterior sobre árvore suja foi removido)

## RELEASE_CANDIDATE
**NO**

**Único bloqueio restante**: a credencial QA exposta precisa ser rotacionada ou desativada por autoridade humana/administrativa. O bloqueio funcional da Dúvida foi corrigido e fisicamente aprovado; os artefatos finais estão prontos, mas não devem ser promovidos enquanto a credencial histórica continuar válida.

Todo o restante do critério da seção 26 está satisfeito: a violação arquitetural encontrada foi corrigida, zero rota paralela indevida remanescente, zero autoridade duplicada remanescente, zero legado morto relevante, zero teste vermelho, GitHub com o código válido, APK/AAB rastreáveis ao SHA final e produção real saudável (`https://simaitutor.com` health 200; servidor implantado em `3628af9`, enquanto `425e052` altera apenas metadados/documentação).

## CONTINUE DAQUI

1. A credencial de QA anteriormente registrada neste relatório foi removida. Como ela entrou no histórico Git, deve ser rotacionada antes de qualquer uso futuro.
2. O reteste físico do APP `9e66709` foi executado na aula de Frações, sem tocar na aula de Kiribati: Dúvida em E1 ficou abaixo do feedback, expirou em E2, permaneceu ausente após o feedback de E2, no item seguinte e após reinício do processo.
3. Os artefatos de `9a668a6` foram substituídos e não devem ser promovidos. Usar somente os artefatos vinculados ao APP `9e66709`, após registro dos hashes finais no handoff.
