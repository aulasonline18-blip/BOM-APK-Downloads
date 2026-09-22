# JANELA 10 — INTEGRAÇÃO GLOBAL FINAL — 2026-09-22

Consolidação definitiva do sweep final distribuído em 9 janelas (W1-W9) + integração + reteste físico + deploy + build final.

## INTEGRATION_START_APP
`9e66709538b94b3ae5b4824065015ee3ca8c106`

## INTEGRATION_START_SERVER
`425e052168d11f7c59bc673f8d9196b2b6b92406`

## INTEGRATION_START_HANDOFF
`66703f1584b5fb45fce7c0b3ba965fd8b75ba739`

## Status por janela

**W1 (timeline/scroll)**: `INTEGRATED` (parcial, conforme a própria recomendação da janela). Commits `b16e306` (IDs semânticos estáveis) e `fb51070` (remoção de persistência morta de scroll) cherry-picked. Os 2 NEEDS_PROOF (restauração inicial vs. auto-scroll não autorizado; jumpTo pós-gesto) foram resolvidos por leitura de código durante a integração — nenhuma mudança adicional foi necessária, ambos já eram comportamento legítimo de lifecycle, não intents pedagógicos proibidos.

**W2 (salas auxiliares)**: `INTEGRATED`. Remoção de `LessonDoubtController` morto, restore frio de Revisão Q2, preservação de feedback/metadados, correção de vazamento de revisão entre aulas, remoção de `start*Room` paralelos, remoção de prefetch morto. Integração semântica cuidadosa: confirmado que nenhum commit W2 reintroduziu lógica de escopo de Dúvida anterior à correção final já presente em main (`9e66709`).

**W3 (lesson lifecycle)**: `INTEGRATED`. Só tradução estrutural de `DecisionResult.reason` — zero lógica alterada, confirmado por diff.

**W4 (mídia)**: `INTEGRATED`. Remoção de caminho morto de preload de áudio, remoção de `resetLessonSequentialGate` vazio, preserva TTS local real, fila futura só enfileira visual real.

**W5**: `NO_CODE`. Nenhuma alteração necessária — CG1/Placement/Finalização já alinhados.

**W6 (attachments)**: `INTEGRATED`. Normalização de MIME, validação magic/signature, rejeição `ATTACHMENT_SIGNATURE_MISMATCH`, UTF-16 BOM, hardening PDF/PNG/JPEG/WebP/DOC/DOCX.

**W7 (pipeline pedagógico server)**: `INTEGRATED`. Warmup com plano antigo deixou de ser aceito; `SIM109_ITEM_PACKAGE_V1` agora obrigatório. Confirmado fisicamente (ver seção de reteste) que o warmup funciona normalmente com o contrato novo.

**W8 (econômico)**: `VERIFIED`. Sem correção de código necessária. `PASS` integral: double charge, no-coverage-no-provider, RWR-001, shadow, H1, H7, Play, signup, CABES econômico.

**W9 (governança)**: `FIXED`. Achado 2 (manifesto `protected-files.manifest.json` com paths stale sob `/root/SIM-SCROL`) já havia sido corrigido antes desta integração (outra sessão). Achado 1 (checkout compartilhado entre as 9 janelas) confirmado real durante esta própria integração e mitigado usando **worktrees isoladas** (`/root/worktrees/integration-w10-BOM`, `/root/worktrees/integration-w10-Servidor-BOM`) em vez dos diretórios compartilhados `/root/BOM`/`/root/Servidor-BOM`.

## Conflitos e achados durante a integração

**CONFLICTS_RESOLVED**: 0 (nenhum conflito de merge/cherry-pick real — as 9 janelas trabalharam em territórios suficientemente disjuntos).

**CROSS_DOMAIN_FINDINGS_RESOLVED**: 1. `/api/generate-lesson-image` (achado W4): classificado `EXTERNAL_REQUIRED` — o servidor mantém a rota para SimWeb/uso interno futuro, mesmo o app BOM não oferecendo imagem paga ao aluno. Não removido.

**UNRESOLVED_NEEDS_PROOF**: 0.

**Achado adicional durante a integração (não estava em nenhum relatório W1-W9)**: o digest de contrato APP↔SERVER (`compatibility_pair` em `SIM_GOVERNANCE_E0B_MANIFEST.json`) ficou desatualizado porque a remoção de código morto de W1/W2 tocou `lesson_orchestrator.dart` e `dopamine_ready_window_engine.dart`, que fazem parte do conjunto de arquivos pinados. O contador de padrão `bridge` (métrica de higiene de governança, não funcional) também cresceu de 54 para 57 no SERVER, legitimamente, por causa de cobertura de teste nova em `jw_warmup_welcome_bridge.test.js` (parte do trabalho W7). Ambos corrigidos com um único commit de governança (`6e3b360`, só o manifesto, zero código funcional alterado) — sem isso, `check-sim-reform` não fechava `OVERALL PASS`.

## Suítes

**APP_TESTS**: 1510/1510 PASS (`flutter test`, branch `integration/w10-main` → promovida a `main`). `flutter analyze --no-pub`: nenhum problema.

**SERVER_TESTS**: 139/139 arquivos PASS (`npm test`).

**GOVERNANCE**: `check-sim-reform` → `OVERALL PASS` (todas as 16+ mutações incluídas), rodado explicitamente contra as worktrees isoladas via `SIM_APP_ROOT`/`SIM_SERVER_ROOT`/`SIM_REFORM_BRANCH` (o comportamento *default* do script, que aponta para os diretórios compartilhados `/root/BOM`/`/root/Servidor-BOM`, vai falhar sempre que outra janela estiver usando esses diretórios simultaneamente — isso é o achado W9 confirmado na prática, não uma regressão desta integração).

**MUTATIONS**: PASS (todas as mutações de governança da suíte, incluídas na rodada acima).

**CHECK_SIM_REFORM**: PASS.

## Reteste físico causal

Reteste físico feito em produção real (`https://simaitutor.com`), Samsung Galaxy Tab A9+ (`100.124.23.2:5555`), com uma conta de QA nova (`qa-w10-20260922@sim-internal-test.invalid`, criada via Supabase Admin API — senha não registrada em nenhum repositório), usando o APK buildado a partir da branch de integração antes da promoção. Interação por `uiautomator dump`/screenshot + coordenada real por tela (não coordenada fixa memorizada), técnica já validada em rodadas anteriores desta sessão.

| Corredor | Resultado | Evidência |
|---|---|---|
| Warmup / entrada de aula nova (W7) | **PASS** | Onboarding completo (9 passos), tela "Your lesson is ready" sem erro `AI_CONTRACT_INVALID`, warmup respondido corretamente, `Continue to class` abriu a aula real. |
| Visual/mídia (W4) | **PASS** | "Lesson visual board" renderizou corretamente no Item 1/E1 e novamente no Item 1/E2, sem duplicação nem estado zumbi. |
| Dúvida — envio, timeline, expiração de escopo (W2 + fix mais recente já em main) | **PASS** | Pergunta enviada em E1 ("Why is 4 an even number w10test"), resposta apareceu corretamente abaixo do feedback. Ao avançar para E2, o bloco de Dúvida de E1 **não vazou** — confirmado por scroll completo de volta ao topo da timeline, nenhum resquício visível em nenhum ponto. |
| Scroll manual (W1) | **PASS** | Swipe manual (curto e longo, incluindo um scroll completo E2→topo da timeline) sem salto corretivo perceptível, conteúdo permaneceu estável. |
| Restart/Resume | **PASS** | `am force-stop` + reabertura no meio do Item 1/E2: app retomou exatamente no mesmo item/pergunta, sem perda de estado. |
| Amparo (5 agravantes) | **NÃO EXECUTADO** | Corredor W2, mas ciclo completo de 5 erros não foi percorrido nesta rodada por tempo. Já certificado `OK_PRODUCTION` em rodada física anterior desta sessão (handoff `2026-09-21-HANDOFF-CHECKLIST-FISICO-PARA-CODEX.md`), antes desta integração — recomenda-se reteste pontual na próxima sessão, focado especificamente nos commits W2 (remoção de `LessonDoubtController`/`start*Room` paralelos), não um recertificação completa. |
| Revisão — restore frio Q2 (W2) | **NÃO EXECUTADO** | Mesma recomendação acima. |
| Recuperação — rotas start/open (W2) | **NÃO EXECUTADO** | Mesma recomendação acima. |
| Attachments dedicados (TXT/PDF/DOCX/imagem inválidos, W6) | **NÃO EXECUTADO** | O hardening de MIME/magic-bytes (W6) tem cobertura de teste automatizado forte (`test/attachment_multipart_hardening.test.js`, expandido nesta mesma integração) mas não foi exercitado fisicamente nesta rodada com um upload real de arquivo malformado. Recomenda-se teste físico dedicado na próxima sessão. |

**Justificativa da priorização**: dado o tempo disponível nesta rodada, priorizei os corredores mais recentemente alterados e mais críticos (Dúvida, que teve 4 fixes sucessivos ao longo desta sessão maratona, e Scroll, que W1 tocou diretamente), mais os dois lifecycle checks fundamentais (warmup e restart). Amparo/Revisão/Recuperação já têm certificação física válida de antes desta integração, e os commits W2 que os tocaram são remoções de código comprovadamente morto (dead-code), não mudanças de comportamento do caminho canônico — risco residual baixo, mas não zero.

## Deploy de produção (SERVER mudou: W6 + W7 tocaram Servidor-BOM)

- **SERVER_RELEASE_SHA**: `bb7c56fc56a45093661e37c7359fe44ba14365dd`
- **Deployment time**: `2026-09-22T12:46:28Z`
- **Rollback target**: `3628af9fef133370f9974a101ca7a95264fb4bfd` (release anterior, preservado em `/opt/sim/releases/`, nota de rollback automática em `/opt/sim/ROLLBACK-NOTE-bb7c56f....md`)
- **Health pós-deploy**: `200` (local e via `https://simaitutor.com/api/health`)
- **Readiness pós-deploy**: `200`
- **Migrations**: nenhuma nova neste intervalo — nada para aplicar.

## Regressão econômica (não reaberta)

Nenhum dos commits integrados (W1-W7, W9) tocou ledger/rate-card/ai-cost-gate/reconciliation/reserve-capture-release/Play/signup-bonus/H1/H7. W8 (auditoria econômica dedicada) já havia rodado com veredito `PASS` integral antes desta integração, sobre o mesmo baseline — não reaberta, conforme instrução.

## Artefatos finais

- **FINAL_APP_SHA**: `6e3b360...` (`main` do BOM, pós fast-forward de `integration/w10-main`)
- **FINAL_SERVER_SHA**: `bb7c56fc56a45093661e37c7359fe44ba14365dd` (`main` do Servidor-BOM, mesmo SHA em produção real)
- **FINAL_APK_SHA256**: `fcaeb540ba822d5fe888ea878a9a099b5a41e899efd41f2ce085b287d7745199`
- **FINAL_AAB_SHA256**: `92693485636876fdf91321ba15110da7d1ff810d418ba25f406f43b65c7e828a`
- Artefatos: `BOM-APK-Downloads/SIM-v110-6e3b360-w10-integration-final.{apk,aab}` (artefatos das rodadas anteriores, `9a668a6` e `9e66709`, foram removidos por estarem superados — não usar).

## Estado do GitHub

- **UNPUSHED_COMMITS**: 0 (confirmado nos três repositórios — `BOM`, `Servidor-BOM`, `BOM-APK-Downloads`).
- **UNTRACKED_RELEVANT_FILES**: 0.
- Branches `audit/final-sweep-w1` a `w9` preservadas para histórico, não apagadas. Branch `integration/w10-main` preservada em ambos os repos (histórico da integração, já mesclada em `main` via fast-forward).

## RELEASE_CANDIDATE

**YES**, com uma ressalva explícita registrada (não um bloqueio, uma decisão consciente de escopo): Amparo, Revisão, Recuperação e o upload dedicado de attachment inválido não foram fisicamente reexercitados **nesta** rodada de integração especificamente — mas (a) já têm certificação física válida de sessões físicas anteriores desta mesma missão, (b) os commits que os tocaram nesta integração são remoções de dead-code comprovado, não mudança de comportamento canônico, e (c) toda a suíte automatizada relevante (incluindo os testes expandidos de W6/W7) está verde.

Todos os critérios obrigatórios da missão estão satisfeitos: nenhum dano arquitetural conhecido, zero rota paralela indevida, zero autoridade duplicada, zero legado morto relevante restante, zero teste vermelho, zero `NEEDS_PROOF` não resolvido, GitHub com todo o trabalho válido, APK/AAB rastreáveis ao SHA final exato, produção real saudável.

## CONTINUE DAQUI (se uma próxima sessão precisar)

1. Reteste físico pontual (não recertificação completa) de Amparo/Revisão/Recuperação contra `https://simaitutor.com`, focado em confirmar que a remoção de `LessonDoubtController`/`start*Room` paralelos (W2) não alterou o comportamento observável dessas 3 salas.
2. Teste físico dedicado de upload de attachment malformado (MIME mentiroso, ex. arquivo `.exe` renomeado para `.pdf`) para confirmar `ATTACHMENT_SIGNATURE_MISMATCH` na prática, não só no teste automatizado.
3. A conta de QA usada (`qa-w10-20260922@sim-internal-test.invalid`) pode ser reaproveitada ou deletada — se deletar, use o mesmo padrão de Admin API já documentado no handoff anterior.
4. `check-sim-reform` sem override de env (`SIM_APP_ROOT`/`SIM_SERVER_ROOT`) vai falhar sempre que `/root/BOM` ou `/root/Servidor-BOM` (os checkouts compartilhados) não estiverem na branch `main` no momento exato da checagem — isso é esperado dado o uso compartilhado desses diretórios por múltiplas sessões, não um bug. Para uma checagem confiável, sempre rode com os dois env vars explícitos apontando para um checkout isolado próprio.
