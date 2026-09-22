# Janela 10 — Integração global dos 9 sweeps — Rodada 1

## INTEGRATION_START_APP / SERVER / HANDOFF

- `INTEGRATION_START_APP=9e667090538b94b3ae5b4824065015ee3ca8c106`
- `INTEGRATION_START_SERVER=425e052168d11f7c59bc673f8d9196b2b6b92406`
- `INTEGRATION_START_HANDOFF=66703f1584b5fb45fce7c0b3ba965fd8b75ba739` (avançou depois para `1c7bde1` durante W9; não misturado nesta integração)

`origin/main` de `BOM` e `Servidor-BOM` continuavam exatamente nesses SHAs no início desta rodada — nenhum outro trabalho avançou `main` diretamente. Uma outra janela concorrente foi observada com edições **não commitadas** em `src/attachments/attachment-processor.js`/testes relacionados no checkout compartilhado `/root/Servidor-BOM` durante parte desta rodada — não tocado, resolvido operando em worktrees isoladas (`/root/worktrees/integration-w10-BOM`, `/root/worktrees/integration-w10-Servidor-BOM`) para não colidir com nenhuma das outras 8 janelas.

## Ambiente

**Achado operacional crítico, resolvido**: `/root/BOM` e `/root/Servidor-BOM` são checkouts únicos compartilhados por todas as 9 janelas (confirmado pela W9, reconfirmado aqui ao vivo). Toda a integração desta rodada foi feita em worktrees git isoladas, nunca nos diretórios compartilhados. Ambas as branches de integração foram pushadas para preservação:

- `BOM`: branch `integration/w10-main`, HEAD `5a998d31582be4d9cb29d7051ac332725bf30e1b`
- `Servidor-BOM`: branch `integration/w10-main`, HEAD `bb7c56f...` (ver `git log`)

**Achado operacional crítico #2, resolvido**: o disco raiz ficou 100% cheio (0 disponível) no meio da rodada, travando toda suíte de testes para todas as janelas. Causa: ~18GB acumulados em `build/`/`.dart_tool/` de worktrees antigas já finalizadas nesta sessão longa. Liberados 19GB removendo apenas esses caches regeneráveis (nenhum código, histórico git ou artefato de release tocado).

## W1 — Timeline/Scroll

**Integrado (parcial, conforme a própria recomendação do relatório W1)**: commits `b16e306` (identidade semântica de mensagem) e `fb51070` (remoção do bookmark de scroll persistido morto). Cherry-pick limpo, sem conflito.

**NEEDS_PROOF resolvidos por análise direta de código nesta rodada**:
- **9A (restauração inicial)**: classificado `VALID_INITIAL_RESTORE`, não `UNAUTHORIZED_AUTO_SCROLL`. `_scheduleInitialPositioning()` dispara exatamente uma vez (guardado por `_initialPositioningHandled`), antes de qualquer interação do usuário, não compete com os 3 intents pedagógicos, não dispara por rebuild, nunca move depois que o usuário começa a ler. Nenhuma mudança de código.
- **9B (jumpTo de contenção)**: confirmado como correção artificial da aplicação, não limitação nativa do framework. Porém: é 1 dos exatamente 3 sites de `jumpTo` já canônicos, testados e documentados em `canonical_pedagogical_scroll_test.dart` desde uma missão anterior desta mesma sessão (a reserva de espaço é necessária para o cálculo do anchor de 10% do `enterNextExperience`; o próprio relatório W1 já provou que remover reserva+clamp juntos quebra controles terminais reais). Classificado `VALID_CURRENT`. Nenhuma mudança de código.

**Status**: `INTEGRATED` (parcial, por decisão explícita do próprio relatório W1).

## W2 — Salas auxiliares

**Integrado**: commits `a4c36b1`, `b448425`. Verificação semântica específica: confirmei que `LessonDoubtController` (removida pela W2 como morta) e `DoubtRequestScope`/`defaultDoubtError` (preservados) são distintos — o fix final de Dúvida já em `main` (`9e66709`, via `lab_session_warmup_flows.dart`) usa `LabSessionDoubtController` (outra classe) e só importa `DoubtRequestScope` do arquivo que a W2 tocou. Cherry-pick limpo, sem reintrodução de lógica superada.

**Status**: `INTEGRATED`.

## W3 — Lesson lifecycle

**Integrado**: commit `375df13`. Tradução estrutural pura de strings de diagnóstico interno (PT→EN), zero lógica alterada, confirmado por grep — nenhum teste referenciava as strings antigas.

**Status**: `INTEGRATED`.

## W4 — Mídia

**Integrado**: commit `4d17958`. **Regressão real encontrada e corrigida nesta rodada**: `test/anti_loop_protection_contract_test.dart` fazia checagem literal de string por `_slotMediaAlreadyRequested`, símbolo que a W4 renomeou para `_slotImageAlreadyRequested` (guarda preservada, só simplificada porque o branch de áudio morreu). Fix: atualizar a expectativa do teste de governança para o novo nome (commit `d3f763f`), não reverter o refactor válido. Confirmado via bisect: baseline+W3+W2 = 1509/1509 limpo; +W4 sem o fix = 1 falha real e reproduzível; +W4 com o fix = 1510/1510 limpo.

**Status**: `INTEGRATED`.

## W5 — Sem código

`NO_CODE`, nada a integrar.

## W6 — Attachments

**Integrado**: commit `e6e7073`. Cherry-pick limpo no servidor.

**Status**: `INTEGRATED`.

## W7 — Pipeline pedagógico do servidor

**Integrado**: commit `9f0922d`. Cherry-pick limpo no servidor.

**Status**: `INTEGRATED`.

## W8 — Corredor econômico

`NO_CODE / VERIFIED`. Nenhuma correção necessária (relatório `8f3e716`, PASS integral). Regressão econômica reconfirmada nesta rodada com o estado integrado atual (RWR-001, concorrência de créditos, billability, phase1 meter) — todos passando.

## W9 — Governança

`FIXED / VERIFIED`. `protected-files.manifest.json` apontava 7 arquivos de proteção anti-loop para `/root/SIM-SCROL` (worktree morto). `check-protected-files.js` resolvia esse caminho, não encontrava, e pulava silenciosamente toda a checagem do lado do app — proteção de fachada confirmada ao vivo.

Fix aplicado (commit `bb7c56f`, autorizado via `docs/migracao-sim-nv/autorizacoes/SIM-W10-PROTECTED-MANIFEST-APP-ROOT-2026-09-22.json`): path repo-relative com root configurável por `SIM_APP_ROOT` (mesma convenção já usada por `tool/check-sim-reform`), em vez de reintroduzir outro hardcode absoluto. **Provado ao vivo**: uma edição real em `dopamine_ready_window_engine.dart` (worktree app isolada) agora é corretamente detectada pelo gate, onde antes era silenciosamente ignorada.

Achado adicional fora de escopo, não corrigido: `/root/SIM-NV` (outra entrada de `readOnlyRepositories`, `sim-nv-reference`) também está com path morto — não foi o achado original da W9, registrado aqui para uma próxima rodada.

## Seção 11 — `/api/generate-lesson-image`

Investigado do zero: **zero consumidor no runtime atual do app BOM** (`grep` em `lib/` não encontra nenhuma chamada real; só aparece em testes). Porém documentação normativa pré-existente (`docs/IMAGE_PROVIDER_DECISION.md`, `docs/migracao-sim-nv/M1_ADAPTADORES_MOTORES_HERDADOS_E_ROTAS_VELHAS.md`) já registra decisão explícita anterior: rota mantida para SimWeb/uso interno, classificação `ADIAR`, exige prova antes de mexer. Classificação final: **`EXTERNAL_REQUIRED`**, não `DEAD_CONFIRMED`. Nenhuma remoção — ausência de consumidor no BOM não é a mesma coisa que ausência de consumidor real.

## Suítes (estado integrado atual)

- `APP_TESTS`: **1510/1510 PASS** (`flutter analyze --no-pub`: limpo).
- `SERVER_TESTS`: **139/139 PASS**.
- `GOVERNANCE` (`npm run check:protected`): PASS com autorização explícita.
- Regressão econômica dedicada: PASS (RWR-001, concorrência, billability, phase1 meter).
- `npm audit --omit=dev`: 0 vulnerabilidades.

`./tool/check-sim-reform` **não foi re-executado nesta rodada** (não é meu território direto; W9 já confirmou PASS com os parâmetros corretos sobre um estado anterior — deve ser reconfirmado na próxima rodada sobre o estado integrado final).

## CONFLICTS_RESOLVED

2: (1) rename `_slotMediaAlreadyRequested`→`_slotImageAlreadyRequested` da W4 vs. checagem literal do teste de governança; (2) path stale `/root/SIM-SCROL` da W9 vs. fixture de teste (`protected_files_gate.test.js`) que também hardcodava o mesmo path morto.

## CROSS_DOMAIN_FINDINGS_RESOLVED

0 novos nesta rodada (nenhuma das 9 janelas reportou cross-domain finding).

## UNRESOLVED_NEEDS_PROOF

0. Ambos os NEEDS_PROOF da W1 foram resolvidos por classificação com evidência (nenhuma mudança de código necessária em nenhum dos dois).

## O QUE NÃO FOI FEITO NESTA RODADA (escopo real restante)

1. **Reteste físico causal** (seção 16-19 da missão): nenhum reteste no tablet foi feito nesta rodada. Corredores que precisam de confirmação física antes de `RELEASE_CANDIDATE=YES`: Dúvida (W2), Revisão/Recuperação (W2), Amparo (W2, mesmo domínio), Scroll (W1), Attachments (W6), Warmup/entrada de aula (W7).
2. **Deploy do servidor**: nenhum deploy feito. O droplet real continua em `3628af9` (mais antigo que a integração). Não deployar antes do reteste físico.
3. **Build final APK/AAB**: não gerado nesta rodada.
4. **`./tool/check-sim-reform`**: não reconfirmado sobre o estado final integrado.
5. **`/root/SIM-NV`** stale path: achado, não corrigido (fora do escopo original da W9).

## RELEASE_CANDIDATE

**NO** — por decisão de rigor, não por problema encontrado. Todo o código integrado está limpo, testado e sem regressão conhecida, mas nenhum corredor tocado foi confirmado fisicamente em produção real ainda, e não há build final rastreável ao SHA integrado.

## CONTINUE DAQUI

1. `cd /root/worktrees/integration-w10-BOM && git pull` (branch `integration/w10-main`, HEAD `5a998d3`) — ou recriar a worktree a partir de `origin/integration/w10-main` se ela não existir mais.
2. `cd /root/worktrees/integration-w10-Servidor-BOM && git pull` (branch `integration/w10-main`, HEAD `bb7c56f`).
3. Confirmar `git status` limpo em ambas antes de prosseguir.
4. Reteste físico dos corredores listados acima, usando a técnica de `uiautomator dump` → achar coordenada real → tocar (não usar coordenadas fixas memorizadas).
5. Se tudo passar: gerar build final APK/AAB, registrar hashes, fazer deploy controlado do servidor (se aplicável), promover as branches `integration/w10-main` para `main` em ambos os repos (fast-forward, já que partem exatamente do baseline atual de `main`).
6. Reconfirmar `./tool/check-sim-reform` sobre o estado final.
7. Produzir o relatório final definitivo com `RELEASE_CANDIDATE` reavaliado.
