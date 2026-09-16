# Status da Fase C (Cut C) e Google Play Billing — SIM109 Economic Final

**Data do relatório:** 2026-09-16
**Escopo investigado:** `/root/worktrees/sim109-economic-final-app` (branch `reform/sim109-economic-final-construction`) e `/root/worktrees/sim109-economic-final-server`
**Método:** somente leitura (git status/diff/log, busca e leitura de documentação e código). Nenhuma edição foi feita nos worktrees do SIM.

---

## 1. Git status / pendências sem commit

- **App:** working tree limpo, branch sincronizada com `origin/reform/sim109-economic-final-construction`.
- **Server:** branch sincronizada, mas com **7 arquivos untracked** (nunca commitados), todos relacionados a uma revisão do contrato T02:
  - `SIM109_T02_PACOTE_DE_PROVAS_BASE_IMUTAVEL.txt`
  - `SIM109_T02_PACOTE_DE_PROVAS_PARA_AUDITORIA.txt`
  - `SIM109_T02_PACOTE_DE_PROVAS_REVISAO_2_PARA_AUDITORIA.txt`
  - `SIM109_T02_REVISAO_2_DIFF_PARA_AUDITORIA.txt`
  - `T02_SIM109_CANDIDATO_BASE_IMUTAVEL.txt`
  - `T02_SIM109_CANDIDATO_PARA_AUDITORIA.txt`
  - `T02_SIM109_CANDIDATO_REVISAO_2_PARA_AUDITORIA.txt`

São pacotes de prova/auditoria gerados mas nunca `git add`-ados. Vale decidir se serão comitados, movidos para outro lugar, ou descartados antes de seguir.

---

## 2. Onde a Fase C ("Cut C") parou

O projeto não usa literalmente "Fase C" — a nomenclatura real do plano de economia é **"Cut A / Cut B / Cut C"**, documentada em `docs/sim-maximum-economy-project/execution-reports/` (no app). Os arquivos `CODEX_FASE0..15_REVIEW.md` na raiz são um esquema numérico diferente (revisões de código do Codex), não correspondem ao "Cut C" econômico.

### Relatório mais recente (maior mtime de todo o projeto)
`docs/sim-maximum-economy-project/execution-reports/SIM109-APP-CUT-C1-JUST-IN-TIME-NEXT-PACKAGE-REPORT.md` — **2026-09-15 22:30**, parte do commit `ccc6a69`.

**Veredito literal do documento:**
> "SIM109 CUT C.1 PARTIAL — JUST-IN-TIME SUCCESSOR TRIGGER IMPLEMENTED AND FOCUSED TESTED; C.2 MEDIA/AUDIO/ECONOMIC CLOSURE REMAINS."

### O que foi feito (Cut C.1)
Escopo declarado: implementa apenas "a bounded successor package request when a real normal transition enters Experience 2."

Mudança técnica: `LabSessionClassroomAdvanceController` grava a experiência visível antes do advance; após uma transição bem-sucedida E1→E2, exige que o pacote SIM109 atual esteja aceito e invoca `SimOrganism.prepareNextItemPackageIfAuthorized`. O método é guardado por: chave semântica item/currículo/marker, mapa in-flight, checagem de fim-de-currículo e checagem de pacote já existente. O successor fica como projeção preparada — não é promovido a `currentLessonMaterial` — e sua mídia é deferida (áudio/visual não iniciam antes da hora).

**Arquivos de runtime alterados:**
- `lib/features/session/lab_session_classroom_advance_controller.dart`
- `lib/sim/lesson/student_lesson_material_service.dart`
- `lib/sim/organism/sim_organism.dart`

**Teste alterado:** `test/classroom_phase_test.dart`

**Verificação:** `flutter analyze --no-pub` PASS; casos focados E2/N+1 e E1→E2 PASS; suíte de 186 testes (`classroom_phase_test.dart`, `sim109_item_package_orchestrator_test.dart`, `first_lesson_ready_window_test.dart`) PASS. Chamadas de provider usadas nos testes são fakes — nenhuma chamada paga real foi exercitada. Bytes de prompt e runtime do servidor não mudaram.

### O que falta (Cut C.2 em diante) — texto literal do documento
> - package-level current/next authority must replace remaining layer-shaped readiness projections;
> - enforce and test the maximum-one-next frontier across restart, reconnect, and multi-instance execution;
> - bound visual cache/decode/eviction and prove one visual effect;
> - remove the paid AI audio runtime path while preserving local/TTS accessibility behavior;
> - execute causal economic failure tests and classify provider effects;
> - run full APP gate and scale model after those changes.

O próprio documento também é explícito sobre o que **não** foi tocado: *"It does not implement media cache/eviction, paid-audio retirement, provider reconciliation, Billing, staging, Play, AAB, or production."*

### Conclusão sobre a Fase C
Parou no meio: **C.1 concluído e testado** (gatilho just-in-time do próximo pacote). **C.2 não foi iniciado** — e é pré-requisito explícito antes de qualquer trabalho de Billing, staging, Play, AAB ou produção.

---

## 3. Google Play Billing — situação real (não é bug, é configuração/operação pendente)

Não há log de erro, stack trace ou relatório de incidente documentado em nenhum arquivo lido. O código de integração existe e está implementado:

- **App:** `lib/sim/billing/play_billing_functions.dart` — fluxo completo via `in_app_purchase`/`in_app_purchase_android` (compra, listener de purchase stream, consumo, timeout de 2 min). Sem TODO/FIXME visível.
- **Server:** `src/play-billing/play-billing-controller.js` — valida `purchaseToken` contra a Android Publisher API; exige service account (JWT) ou token estático (só em dev); grava crédito de forma idempotente.

### Gate de produção (provável causa raiz do "problema")
```js
// src/app/router.js:191-193
if (!config.GOOGLE_PLAY_PACKAGE_NAME) throw new Error('Production Google Play billing requires GOOGLE_PLAY_PACKAGE_NAME.');
if (config.GOOGLE_PLAY_ACCESS_TOKEN) throw new Error('Production Google Play billing requires service account/JWT; GOOGLE_PLAY_ACCESS_TOKEN is lab-only.');
if (!config.GOOGLE_PLAY_SERVICE_ACCOUNT_FILE && !config.GOOGLE_PLAY_SERVICE_ACCOUNT_JSON && (!config.GOOGLE_PLAY_SERVICE_ACCOUNT_EMAIL || !config.GOOGLE_PLAY_SERVICE_ACCOUNT_PRIVATE_KEY)) throw new Error('Production Google Play billing requires a service account/JWT.');
```
- Não há `.env` com `GOOGLE_PLAY_SERVICE_ACCOUNT_*` preenchido no worktree do server.
- Não há keystore/`key.properties` de assinatura de release no app — `build.gradle.kts` cai no signing config `"debug"` quando `simReleaseSigningReady` é falso, o que bloquearia a geração de um AAB assinado corretamente.
- O Cut C.1 confirma que nenhum teste real de Billing/Play/APK/AAB foi executado até agora — só fakes.

### `docs/GOOGLE-PLAY-SERVER-READINESS.md` (server, 2026-07-13, lido na íntegra)
Descreve o desenho de código já implementado (não um runbook operacional):
- `POST /api/play-billing/consume-credit-pack` valida `purchaseToken`.
- Servidor aceita `GOOGLE_PLAY_ACCESS_TOKEN` (dev) OU service account (produção) e renova OAuth internamente; o app nunca recebe a chave de service account.
- Produtos oficiais definidos: `sim_credits_100`, `sim_credits_200`, `sim_credits_500`.
- Concessão de créditos idempotente por compra.
- Em produção, o servidor **falha ao iniciar** se a credencial real do Google Play estiver ausente (mesmo gate de `router.js`).
- Prova obrigatória via `npm test`, cobrindo validação de token no Android Publisher e não vazamento de `purchaseToken`/chave privada em respostas de erro.

**O documento não contém nenhum checklist de configuração real do Google Play Console** (criar produto X, gerar service account Y, etc.) — é só o contrato de código.

### Plano/roadmap explícito para depois do Cut C.1
Não encontrado nenhum `NEXT_STEPS`/`TODO`/`ROADMAP` mais recente que o próprio relatório do Cut C.1 (confirmado por mtime e por `git log` na pasta `docs/sim-maximum-economy-project`, tanto no app quanto no server — nenhum commit mais novo). O documento de decisão arquitetural anterior `docs/sim-maximum-economy-project/decisions/PHASE-04-HUMAN-DECISION-GATE.md` trata billing apenas como invariante de design a preservar ("sem chamada extra se pacote do item for único"), não como pendência operacional, e explicita que nenhuma opção do documento autoriza mudança de billing ou áudio.

### Conclusão sobre Billing
O código de Billing parece completo e coberto por testes com fakes, mas **nunca foi exercitado contra o Play Console real nem contra um device físico**. O bloqueio real para produção é a ausência de: (1) credenciais reais de service account no server, e (2) keystore de assinatura de release no app — ambos requisitos de configuração/operação, não defeitos de código. Se existe um erro específico observado em algum log ou tela (fora dos documentos lidos), ele não está registrado em nenhum relatório encontrado nesta investigação.

---

## 4. Ordem recomendada de trabalho (dedução a partir da documentação, não uma decisão já tomada)

1. Decidir o destino dos 7 arquivos untracked no server (commitar, arquivar ou descartar).
2. Concluir Cut C.2 (fechamento de mídia/áudio/economia — é pré-requisito explícito antes de Billing/Play/AAB segundo o próprio relatório do Cut C.1).
3. Só então configurar credenciais reais de Google Play (service account/JWT) e keystore de release, e exercitar o fluxo de Billing de ponta a ponta.
4. Gerar APK de teste → validar em tablet físico → gerar AAB de teste interno → publicar AAB de produção.
5. Segunda fase: limpeza/padronização geral do código.
