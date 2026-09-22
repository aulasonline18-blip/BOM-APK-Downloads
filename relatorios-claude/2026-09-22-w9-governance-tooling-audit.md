# W9 — Auditoria de Governança / Tooling / Release Hygiene — 2026-09-22

**Papel**: janela W9 do sweep final distribuído (9 janelas). Não é integradora global.

## BASELINE_APP
`9e66709` (BOM, main)

## BASELINE_SERVER
`425e052` (Servidor-BOM, main)

BASELINE_HANDOFF = `66703f1` (BOM-APK-Downloads, main)

**Nota operacional crítica encontrada durante a auditoria**: no momento em que este baseline foi congelado, `Servidor-BOM` tinha mudanças não commitadas em `src/attachments/attachment-processor.js`+testes (outra janela, não tocado). Além disso, durante a execução desta auditoria, `/root/BOM` foi encontrado com o checkout ativo trocado para `audit/final-sweep-w1-timeline-scroll` (branch de outra janela, W1) — ver `ACHADOS_CONFIRMADOS` abaixo, é o achado mais importante deste relatório.

## ARQUIVOS_INSPECIONADOS
`.github/workflows/e0b-governance.yml`; `tool/check-sim-reform`; `Sim organizing/governance/**` (README, manifest, schema, checks/*.py); `docs/migracao-sim-nv/protected-files.manifest.json`; `docs/migracao-sim-nv/PROTOCOLO_DE_EXECUCAO_PROTEGIDA.md`; `scripts/check-protected-files.js`; `scripts/build-bom-production-apk.sh`/`build-bom-production-aab.sh`; `pubspec.yaml` (BOM) e `package.json` (Servidor-BOM) completos; `test/protected_files_gate.test.js`; `test/anti_loop_protection_contract_test.dart`; `test/final_proposition_c_contract.test.js`; docs de arquitetura tocados pelos commits do baseline (`docs/AUDIO_PARALLEL_CONTRACT.md`, `docs/migracao-sim-nv/LEI_PROTECAO_TRAVAS_ANTI_LOOP_SIM_NV.md`, `docs/migracao-sim-nv/M2A_CONTRATOS_DE_COMPATIBILIDADE_APP_SERVIDOR.md`, `docs/sim_nv_app_architecture_inventory.md`); varredura de segredos (`git grep` por padrões de chave/token/private-key) nas duas árvores; `build/`, `.dart_tool/` (espaço em disco).

## ACHADOS_CONFIRMADOS

1. **RELEASE_RISK — ambiente de múltiplas janelas compartilha um único checkout físico por repositório.** `/root/BOM` e `/root/Servidor-BOM` são diretórios de trabalho ÚNICOS nesta máquina. O protocolo de 9 janelas assume isolamento (cada janela em sua branch), mas aqui um `git checkout <branch>` de uma janela muda o branch ativo para TODAS as janelas que operam nesse mesmo diretório. Confirmado ao vivo: durante esta auditoria, `/root/BOM` estava com HEAD em `audit/final-sweep-w1-timeline-scroll` (branch da janela W1), não em `main`, embora sem divergência de conteúdo no momento (mesmo commit `9e66709`). Isso é uma condição de corrida latente séria: se duas janelas fizerem `git checkout` de branches diferentes quase simultaneamente, ou se uma janela editar arquivos enquanto outra troca o branch, há risco real de corrupção de working tree ou perda de edição não commitada. **Recomendação para a integração**: antes de qualquer fusão, confirmar que cada janela commitou tudo em SUA branch nomeada antes de outra janela tocar o checkout compartilhado; considerar `git worktree add` por janela em vez de checkout compartilhado nas próximas rodadas.

2. **ARCHITECTURAL_VIOLATION / RELEASE_RISK — "check que pode dar PASS sem olhar o código correto" confirmado em `docs/migracao-sim-nv/protected-files.manifest.json` (Servidor-BOM), seção `anti-loop-protection`.** O manifesto lista 7 caminhos absolutos com prefixo `/root/SIM-SCROL/...` (mais 1 no grupo `readOnlyRepositories`) como arquivos protegidos do lado APP:
   - `/root/SIM-SCROL/lib/sim/external_ai/sim_ai_server_config.dart`
   - `/root/SIM-SCROL/lib/sim/external_ai/sim_server_ai_clients.dart`
   - `/root/SIM-SCROL/lib/sim/lesson/ready_window_worker.dart`
   - `/root/SIM-SCROL/test/anti_loop_protection_contract_test.dart`
   - `/root/SIM-SCROL/lib/sim/lesson/dopamine_ready_window_engine.dart`
   - `/root/SIM-SCROL/lib/sim/media/student_lesson_media_service.dart`
   - `/root/SIM-SCROL/test/first_lesson_ready_window_test.dart`
   - `/root/SIM-SCROL/docs/LEI_PROTECAO_TRAVAS_ANTI_LOOP_SIM_NV.md`

   **Evidência de que é morto/inerte**: `/root/SIM-SCROL` existe no disco mas está **vazio** (0 arquivos) — era o checkout antigo do app antes de virar `/root/BOM` (confirmado por uma entrada irmã em `readOnlyRepositories`: `{"id":"sim-flutter-app","path":"/root/SIM-SCROL","role":"out-of-scope-for-server-governance-task"}`). Além disso, **7 dos 8 arquivos listados EXISTEM hoje em `/root/BOM`** sob caminho relativo equivalente (só `lib/sim/lesson/ready_window_worker.dart` não existe mais em nenhum lugar) — ou seja, o manifesto tenta proteger arquivos reais e importantes (proteção anti-loop), mas usa um caminho absoluto que nunca pode corresponder a uma mudança real detectada por `scripts/check-protected-files.js` (que resolve `gitChangedPaths()` relativo à raiz do próprio repo Servidor-BOM — um caminho absoluto `/root/SIM-SCROL/...` jamais é igual a um caminho de diff relativo a `/root/Servidor-BOM`). Rodei `npm run check:protected` no baseline: `Protected-files check passed: no protected files changed` — passa, mas essa parte específica da proteção NUNCA seria capaz de disparar, mesmo que alguém alterasse `lib/sim/lesson/dopamine_ready_window_engine.dart` deliberadamente sem autorização. É proteção de fachada para esses 7 arquivos.

   Achado irmão: `readOnlyRepositories` também tem `{"id":"sim-nv-reference","path":"/root/SIM-NV",...}` — `/root/SIM-NV` **não existe** no disco (nem vazio, inexistente).

   **Correção recomendada** (não apliquei — decisão de escopo cruzado com quem administra o mecanismo de proteção anti-loop do app, prefiro reportar com evidência completa a arriscar um fix incompleto): substituir os 7+1 caminhos absolutos `/root/SIM-SCROL/...` por uma referência relativa correta ao repo APP atual, OU confirmar (não tive tempo de provar 100%) se a proteção equivalente já existe via `Sim organizing/governance/checks/sim_governance_check.py --all` (que roda com `--app-root`/`--server-root` explícitos e cobre hashes normativos/pin APP-SERVER — rodei este check no baseline, **PASS em tudo**, incluindo hash de prompts e pin de compatibilidade; é plausível que ele já cubra esses arquivos por outro mecanismo, tornando o manifesto do Servidor-BOM redundante-e-morto em vez de ser a única linha de defesa). Marco como **NEEDS_PROOF** a extensão exata da redundância — não removi nada.

3. **OBSOLETE_CONFIG confirmado, já corrigido antes deste baseline**: `docs/migracao-sim-nv/protected-files.manifest.json` referenciava `src/media/audio-controller.js` (arquivo que não existe mais) — já corrigido no commit `425e052` (dentro do baseline) para `src/media/image-controller.js` (existe). Não é uma pendência, é uma correção real já aplicada por outra janela antes do meu baseline — cito para constar que verifiquei e está correta.

4. **CHECK_SIM_REFORM_DECISION — reauditado do zero, evidência fresca**: `tool/check-sim-reform` tem os *defaults* corretos desde o commit `a4f1f10` (`APP_ROOT=/root/BOM`, `SERVER_ROOT=/root/Servidor-BOM`, `BRANCH=main` — confirmei lendo o diff e o arquivo atual). Rodei os dois checks Python que ele invoca diretamente com `--app-root /root/BOM --server-root /root/Servidor-BOM`, sem nenhum artifício: `sim_governance_check.py --all` → **11/11 PASS** (schema, hashes normativos, pin APP/SERVER, 150 proposições, decisões humanas, autoridade/escritor, bridge/fallback, custo/chamadas, L3/áudio futuro, phase broom, gates E0-B); `run_mutation_tests.py` → **17/17 PASS MUTATION** (inclui `protected-hash-drift`, `break-app-server-pin`, `remove-proposition`, `unauthorized-writer`, `paid-call-growth`, `hash-cycle`, `pin-circular`, entre outros). O único jeito de fazer o script `tool/check-sim-reform` (o wrapper bash, não os checks Python) falhar agora é a condição de corrida do achado #1 (branch trocado por outra janela) — não é defeito do script. **Decisão: MANTER.** Aponta para o main real, valida conteúdo real (hashes/pin/proposições atuais), não duplica outro check (é a ÚNICA verificação combinada APP+SERVER desta natureza), não usa worktree obsoleto (isso já foi corrigido), e o manifesto de governança (`Sim organizing/governance/SIM_GOVERNANCE_E0B_MANIFEST.json`) corresponde ao código atual (schema válido, hashes batendo). Não reponte de novo sem evidência nova — o repontamento de `a4f1f10` já foi a correção certa.

5. **UNUSED_DEPENDENCY**: nenhuma confirmada. Spot-check de 18 dependências `pubspec.yaml` + 6 `dev_dependencies`: todas com uso real em `lib/`, exceto `sqlite3_flutter_libs` (0 ocorrências de import direto) — classificado **VALID_CURRENT**, não `UNUSED_DEPENDENCY`: é dependência companheira nativa padrão do `drift` (usado em 6 arquivos) para bundlear a lib SQLite nativa em mobile; não se importa por nome, se auto-registra. `package.json` do Servidor-BOM: 7 dependências (`@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`, `express`, `jose`, `redis`, `sharp`, `word-extractor`) mais 1 override (`qs@6.16.0`) — todas de uso óbvio e coerente com o domínio (S3/storage, JWT, Redis, imagem, extração de documento). Nenhuma removida.

6. **SECURITY_FINDINGS**: nenhum segredo real encontrado na árvore atual de nenhum dos dois repos (`git grep` por padrões de API key Google, chave OpenAI-like, cabeçalho de chave privada PEM — vazio nos dois). Não escaneei o histórico completo de commits (fora do escopo de tempo desta rodada) — apenas a árvore no baseline. **NEEDS_PROOF**: varredura completa de histórico Git (todos os commits, não só HEAD) não foi feita por mim nesta rodada.

7. **STALE_TESTS**: 0 confirmados no meu território. Os testes do caminho `finite_feedback_auto_advance` morto foram removidos junto com o código (não deixados como legado artificial) por outra janela antes do meu baseline — verifiquei que não sobrou nenhum arquivo de teste órfão apontando para símbolos inexistentes.

8. **Espaço/artefatos**: `build/` (1.6G) e `.dart_tool/` (2.2G) em `/root/BOM` são caches efêmeros, não rastreados no git (`.gitignore` cobre ambos), sem artefato de release commitado indevidamente na raiz do repo. Não apaguei nada (achado #1 torna arriscado mexer no working tree compartilhado agora).

## CROSS_DOMAIN_FINDINGS

Nenhum encontrado que pertença claramente a outra janela específica além do já sinalizado. O achado #2 (manifesto de proteção anti-loop) tangencia território de quem mantém a proteção anti-loop do app (`lib/sim/lesson/*`, `lib/sim/external_ai/*`) mas o ARQUIVO DEFEITUOSO em si (`protected-files.manifest.json`) é governança pura — do meu território — por isso tratei como achado próprio, não cross-domain.

## ARQUIVOS_MODIFICADOS
Nenhum.

## ARQUIVOS_REMOVIDOS
Nenhum.

## ROTAS_PARALELAS
0 (nenhuma rota paralela de execução funcional encontrada — não é território funcional de W9, mas nada saltou aos olhos durante a leitura de scripts de build/CI).

## AUTORIDADES_DUPLICADAS
0 confirmadas no meu território (governança tem uma única fonte: `Sim organizing/governance/checks/*.py`, invocada por `tool/check-sim-reform` e pelo workflow `.github/workflows/e0b-governance.yml` — mesmo par de scripts, sem duplicação).

## LEGADO_MORTO
1 confirmado e não removido por mim (achado #2 — proteção morta por caminho absoluto stale); 1 já removido por outra janela antes do meu baseline (achado #3, `audio-controller.js` → `image-controller.js`, não é pendência).

## TESTES_EXECUTADOS
`npm run check:protected` (Servidor-BOM); `node test/protected_files_gate.test.js`; `Sim organizing/governance/checks/sim_governance_check.py --all` (--app-root /root/BOM --server-root /root/Servidor-BOM); `Sim organizing/governance/checks/run_mutation_tests.py` (idem); `flutter test test/anti_loop_protection_contract_test.dart`; `node --check scripts/check-protected-files.js`.

## TESTES_RESULTADO
Todos PASS: `check:protected` (passou, sem falso-positivo, mas ver achado #2 sobre falso-negativo estrutural); `protected_files_gate.test.js` PASS; `sim_governance_check.py --all` 11/11 PASS; `run_mutation_tests.py` 17/17 PASS MUTATION; `anti_loop_protection_contract_test.dart` 3/3 PASS; sintaxe de `check-protected-files.js` OK.

## BRANCH
Nenhuma criada — nenhuma correção foi aplicada nesta rodada (decidi reportar o achado #2 com evidência completa em vez de editar um manifesto de proteção com risco de regressão sem confirmação cruzada, e o achado #1 é ambiental, não corrigível por edição de arquivo).

## COMMITS
Nenhum em BOM/Servidor-BOM. Este relatório: commitado em `BOM-APK-Downloads` (main, ver hash no final).

## READY_FOR_INTEGRATION
**YES**, com duas ressalvas explícitas para a etapa integradora: (a) achado #1 (checkout compartilhado entre janelas) deve ser resolvido/mitigado antes de qualquer merge simultâneo de múltiplas branches `audit/final-sweep-wN-*`; (b) achado #2 (proteção anti-loop morta por caminho stale) precisa de uma decisão consciente — corrigir o caminho, ou confirmar redundância via `sim_governance_check.py` e então decidir se remove ou mantém como defesa-em-profundidade.

---

## GOVERNANCE_CHECKS
4 (`check:protected`, `sim_governance_check.py --all`, `run_mutation_tests.py`, `protected_files_gate.test.js`) — todos executados, todos PASS.

## STALE_GOVERNANCE_PATHS
2 confirmados: (1) 7 caminhos absolutos `/root/SIM-SCROL/...` em `protected-files.manifest.json` (achado #2); (2) `readOnlyRepositories.sim-nv-reference` → `/root/SIM-NV` (não existe no disco).

## OBSOLETE_CONFIG
2 (os mesmos do item acima — `readOnlyRepositories` com 1 path inexistente e 1 path vazio/abandonado).

## UNUSED_DEPENDENCIES
0 confirmadas.

## SECURITY_FINDINGS
0 confirmados na árvore atual (histórico completo não escaneado — NEEDS_PROOF).

## STALE_DOCS
0 confirmados como induzindo operação errada. Docs tocados no baseline (`AUDIO_PARALLEL_CONTRACT.md`, `LEI_PROTECAO_TRAVAS_ANTI_LOOP_SIM_NV.md`, `M2A_CONTRATOS_DE_COMPATIBILIDADE_APP_SERVIDOR.md`) foram atualizados coerentemente pela mesma janela que fez a limpeza de código morto — li os diffs, refletem o estado atual.

## CHECK_SIM_REFORM_DECISION
**MANTER** (reauditado do zero nesta rodada — ver achado #4 para evidência completa).

## BUILD_CONFIGURATION
PASS — `scripts/build-bom-production-apk.sh`/`build-bom-production-aab.sh` existem, foram usados com sucesso por outra janela nesta mesma sessão (hashes já registrados em relatórios anteriores desta missão), sem flag de build morta ou endpoint de produção incorreto encontrado nos scripts.

## RELEASE_HYGIENE
PASS, com as 2 ressalvas documentadas em `READY_FOR_INTEGRATION` acima (checkout compartilhado entre janelas; proteção anti-loop com caminho morto).
