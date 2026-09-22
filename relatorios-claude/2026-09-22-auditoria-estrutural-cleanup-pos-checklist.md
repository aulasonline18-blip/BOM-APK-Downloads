# Auditoria estrutural / cleanup completa — pós-checklist, pós-auditoria econômica

**Baseline auditado**: APP `b51df222` / SERVER `3628af9f` (o mesmo par confirmado na auditoria econômica anterior, `2026-09-22-auditoria-integridade-microcredito-pos-checklist.md`, commit `ae642f3`).

## FASE 0 — Reconciliação documental (feita primeiro, antes de qualquer alteração)

Cruzando timestamps reais de commit (`git log --format=%ci`) para resolver a ambiguidade entre as duas linhas de trabalho paralelas que editaram o handoff `1108f0e`:

- **#120 CG-1, #122 Amparo, #123 Dúvida/Revisão/Recuperação, #124 Finalização, #126 Restart/Offline, #127 Account/Microcrédito/Billing**: confirmados `OK_PRODUCTION` de verdade — as ATUALIZAÇÕES 6-23 do handoff são cronologicamente as mais recentes e completas. A "lista numerada histórica" no fim do arquivo (menciona `0d422ad`/"terceiro stall do Amparo", `2026-09-21 09:43`) é anterior e foi genuinamente superseded pela ATUALIZAÇÃO 13 (`7665118`, `2026-09-22 00:47`), que fechou o Amparo por completo. TaskList atualizada, `#120/#122/#123/#124/#126/#127 → completed`.
- **#121 Placement — gap real encontrado**: só há prova física de placement em escala normal (aula de 20 itens). Nenhuma evidência de placement especificamente dentro de currículo CG-1 (60-150 itens). Marcado `RETEST_REQUIRED` nesse escopo específico.
- **#125 Rename — contradição real resolvida a favor da evidência mais forte**: a matriz do handoff cita `OK_PRODUCTION` (ATUALIZAÇÃO 7, aula recém-criada, fluxo limpo), mas a Seção H do mesmo documento (mais recente) reproduziu 2x, deterministicamente, rename falhando silenciosamente em aulas com histórico de múltiplas sessões de teste. Candidato de causa raiz (`student_state_store.dart:2569`, guarda `rename_expected_objective_mismatch`) não confirmado ao vivo. Revertido para `RETEST_REQUIRED`; Menu/Drawer permanecem `OK_PRODUCTION`.

Handoff (`2026-09-21-HANDOFF-CHECKLIST-FISICO-PARA-CODEX.md`) e TaskList atualizados e commitados separadamente (`251f07e`) antes desta auditoria estrutural começar.

## FASE 1 — Auditoria estrutural

Áreas de atenção especial auditadas: APP (Dúvida/Revisão/Recuperação/Amparo/advance/prefetch/restore-rehydration/scroll/finalization/CG1), SERVER (T00/T02/ai-cost-protection-gate/reconciliation/durable-ledger/credits/economic-context/result-recovery/billing-identity).

### Achados

1. **Nenhum marcador de dívida técnica ativo**: `grep -rniE "// TODO|// FIXME|@deprecated"` em `src/` (server) e `lib/` (app) não retornou nenhuma ocorrência real fora de um comentário de documentação inofensivo (`signal_tracker.dart:56`, é prosa, não marcador).
2. **Pricing legado × motor metered — uma autoridade só, confirmado de novo**: `T00_PART_CREDIT_COST`/`T02_ITEM_CREDIT_COST`/`IMAGE_CREDIT_COST` são a autoridade ATIVA de cobrança hoje (usados diretamente em `native-bootstrap-controller.js`, `complete-lesson-controller.js`, `image-controller.js`). `MICROCREDIT_PRICING_ENFORCE` continua com zero consumidores funcionais (só declaração de config + comentários) — não é uma segunda autoridade concorrente, é um motor futuro genuinamente inerte.
3. **Reconciliation — caminho único confirmado**: `markEffectAmbiguous` tem exatamente 3 chamadores (`durable-ledger.js` definição, `ai-cost-protection-gate.js` gate canônico, `play-billing-rtdn.js`), todos convergindo no mesmo primitivo. A fila própria do motor metered (`sim_metered_operations`/`bom_metered_pending_reconciliation_queue`) opera sobre tabela isolada e inerte — não é um segundo caminho concorrente para o motor ativo.
4. **App: `nextAdvanceReady`/`ensureNextAulaAdvancePrepared`/`WarmupBridgeCoordinator`/`_hydrateActiveLessonFromCloud`** — nomes parecidos, responsabilidades genuinamente distintas (checagem de prontidão vs. agendamento de retry vs. gate de transição aquecimento→aula vs. sincronização de estado remoto). Os três bugs corrigidos nesta sessão em torno desses símbolos tinham causas raiz totalmente diferentes e não sobrepostas — não há autoridade duplicada aqui, é separação de responsabilidades legítima.
5. **`legacyAccountGenerationOperationIdentity`** (ponte de leitura do fix `6885043`, restart-identity): classificado `LEGACY_REQUIRED` por enquanto — necessário até que operações pré-migração deixem de ser relevantes (janela de transição), estritamente somente-leitura, namespaced, sem chance de colisão. Não é uma violação; é um candidato honesto a `LEGACY_REMOVABLE` numa auditoria futura, quando se puder provar que nenhuma operação pré-migração ainda precisa dele.
6. **`flutter analyze --no-pub`**: limpo, sem warnings de código morto/não utilizado.
7. **Suítes completas, rodadas agora, ao vivo**: `flutter test` → **1506/1506 PASS**. `npm test` (server) → **139/139 arquivos PASS**.
8. **Governança (`./tool/check-sim-reform`)**: `FAIL` — mas por um motivo pré-existente e não-arquitetural, já documentado antes desta auditoria: o script espera o checkout do SERVER estar na branch `reform/two-experience-staging`; o checkout real está em `main` (que é o fluxo de trabalho atual desde que a disciplina de merge direto para `main` foi estabelecida nesta sessão). Isso é uma expectativa desatualizada do PRÓPRIO script de governança (nome de branch obsoleto), não um dano estrutural no código auditado.

### Não auditado em profundidade suficiente para PASS/FAIL definitivo (NEEDS_PROOF)

- Varredura arquivo-a-arquivo exaustiva de TODO o restante dos dois repositórios em busca de helpers/classes sem nenhum consumidor (esta auditoria foi direcionada às áreas de atenção especial nomeadas pelo usuário e ao diff recente, não um sweep total linha-a-linha de tudo).
- Fronteira de compatibilidade legada do CG-1 de 20 itens (isolada como somente-leitura nas tasks #84-85 de uma sessão anterior) — não reverificada nesta rodada especificamente.

## Entrega

```
CLEANUP_AUDIT = PASS
PARALLEL_PATHS_FOUND = 0
DEAD_LEGACY_FOUND = 0
DUPLICATE_AUTHORITIES_FOUND = 0
FILES_REMOVED = 0
PATHS_CONSOLIDATED = 0
ARCHITECTURAL_VIOLATIONS = 0
APP_TESTS = 1506/1506 PASS
SERVER_TESTS = 139/139 arquivos PASS
GOVERNANCE = FAIL (mismatch de nome de branch pré-existente no próprio script `check-sim-reform`, não-arquitetural, documentado antes desta auditoria)
PHYSICAL_RETEST_REQUIRED = nenhum item novo (nenhum código foi alterado nesta auditoria). Os dois RETEST_REQUIRED já existentes e reconciliados na FASE 0 continuam abertos, sem relação com esta auditoria: (1) Placement dentro de currículo CG-1 (60-150 itens) — nunca testado fisicamente; (2) Rename de aula em lições com histórico de múltiplas sessões de teste — causa raiz candidata não confirmada ao vivo.
```

Nenhuma remoção ou consolidação foi aplicada porque nenhuma autoridade duplicada, rota paralela ou legado morto com dano real foi encontrado dentro do escopo auditado — consistente com a regra do usuário de não fazer refactor cosmético sem prova de dano.
