# Auditoria de integridade do microcrédito e do corredor econômico — pós-checklist físico

**Contexto**: o checklist físico fechou 27/27 `OK_PRODUCTION` (APP `b51df222`, SERVER `3628af9f`, HANDOFF `1108f0e4`, APK v110). Antes de liberar a próxima fase (cleanup estrutural global), esta auditoria prova que os fixes aplicados depois do fechamento técnico original do microcrédito não reabriram a arquitetura econômica.

## PROTECTED_MICROCREDIT_BASELINE
`1f3172b` (Servidor-BOM main) — estado logo após o shadow-isolation fix, H1 (margem 40%), H7 (FX PTAX/BCB), BACKFILL real de `credit_units` em produção, e a CABES-SIM. Commits-chave: `22c7bf2`, `043d791`, `6ff8370`, `4809115`, `5f7e0cf`, `1f3172b`.

## CURRENT_SERVER_MAIN
`3628af9fef133370f9974a101ca7a95264fb4bfd`

## ECONOMIC_COMMITS_POST_BASELINE
8 commits entre `1f3172b` e `3628af9f`, todos lidos na íntegra:

| commit | classificação | resumo |
|---|---|---|
| `ee203bf` | SAFE_PARSER | Projeção de conteúdo SIM109 para review/recovery/support — zero toque em custo/reserva/ledger |
| `42d541d` | **ECONOMIC_SENSITIVE** | Recovery de material T02 já pago em reinstalação limpa — ver detalhe abaixo |
| `d0b32d6`, `68d7b87`, `519f3c8` | SAFE_PARSER | Parsing/diagnóstico de currículo T00 (t00-parser.js). Grep por credit/cost/charge/reserve/capture/release/ledger/price/balance/fee: zero ocorrências |
| `72c7d20` | SAFE_PARSER | Placement passa a consumir a projeção do pacote canônico. Autorização formal proíbe explicitamente tocar reserva/captura/ledger/custo |
| `6885043` | **ECONOMIC_SENSITIVE / ARCHITECTURAL_SENSITIVE** | Identidade econômica estável entre restarts — ver detalhe abaixo |
| `3628af9` | SAFE_FUNCTIONAL | Só adiciona uma chamada de log (`safeError`) em falha de visual-route |

### Detalhe: `42d541d` (recovery de material T02 em clean reinstall)
Corredor novo em `ai-cost-protection-gate.js`: se a operação comercial (`creditOperationKey`) já está `accepted`/`captured` (ou seja, **o aluno já foi cobrado**) mas o resultado durável não existe mais (ex.: reinstalação limpa apagou o cache local), o servidor:
1. Primeiro tenta replay do resultado já aceito — **sem provider, sem débito**.
2. Só se o resultado em si estiver faltando, faz **UMA** chamada de provider adicional, sob uma `effectKey` própria e namespaced (`${creditOperationKey}:result-recovery:v1`, sem colisão possível com a chave real), explicitamente marcada `financialDisposition:'absorbed'`, `charged:false`.
3. Teste automatizado (`rwr001_economic_saga_baseline_contract.test.js`) prova com stubs que **lançam exceção se hold/take/free forem chamados de novo** — passou. Replay subsequente não gera novo provider call (`cleanInstallProviderStarts` permanece 1).
4. Estado ambíguo continua fail-closed.
5. Reusa 100% dos primitivos existentes (durableLedger, economicContext, assertBudgets, assertCircuit) — não é motor paralelo.
6. Tem arquivo de autorização/governança formal (`docs/migracao-sim-nv/autorizacoes/`), no padrão de protected-path da CABES-SIM.

**Veredito: PASS.** Nenhuma cobrança nova, nenhuma segunda reserva, disposição econômica explícita e rastreável.

### Detalhe: `6885043` (identidade paga estável entre restarts)
Bug real documentado na própria autorização: reiniciar o app no mesmo item mudava o `accountGeneration` (contador client-side), criando nova identidade econômica → novo provider → novo débito. Fix: remove `accountGeneration` da base de identidade para T02/VISUAL_ROUTER (mantém userId, aula, marker, itemIdx, conteúdo, locale, versões de contrato). Adiciona `legacyAccountGenerationOperationIdentity` como **ponte estritamente de leitura**: só consultada se a nova identidade não tiver efeito durável ainda; se a identidade antiga tiver resultado aceito/capturado, faz replay (sem provider, sem débito), loga `legacy_account_generation_result_recovered`/`duplicateSuppressed:true`.

**Veredito: PASS.** Ponte só lê e reproduz; nunca chama provider nem cobra, exatamente como a autorização exige.

## MICROCREDIT_INVARIANTS_PRESERVED: **YES**

## DOUBLE_CHARGE: **PASS**
- Constraint real de banco `unique (account_id, effect_key)` em `sim_durable_operations` (`202608300002_phase_b2_durable_ledger.sql:37`) — exactly-once estrutural, não só disciplina de código.
- Relatório econômico real de produção (últimos ~2 dias, RPC `bom_ai_cost_report_v2`): **`chargedCallsWithoutReservation: 0`**, **`capturesWithoutAcceptedMaterial: 0`**, **`callsWithoutCorrelation: 0`**, `negativeMargin: false`.
- Créditos: 144 reservados → 143 capturados + 1 liberado = fecha exatamente, zero reserva órfã.

## NO_COVERAGE_NO_PROVIDER: **PASS**
Reexecutei agora, ao vivo, `test/coverage_before_provider_call_contract.test.js` contra o código atual: **"0 provider calls under insufficient balance, in both main and aux-room modes, single and concurrent"**. Este teste conta chamadas reais de provider (não infere por status HTTP), contra o wiring de produção real (`createCompleteLessonController` + `createCreditsStore` + `createAiCostProtectionGate`).

## REPLAY / RESTART / CLEAN_INSTALL: **PASS**
Provados pelos testes automatizados dos commits `42d541d` e `6885043` (stubs que lançam exceção se reserva/captura/release forem chamados de novo; segunda chamada não gera novo provider call). Não fiz um novo teste manual ao vivo no tablet além do que os forks anteriores já confirmaram fisicamente (o botão "Continuar para aula" e o fluxo de restart já foram validados fisicamente em produção real nesta mesma missão, tasks #126).

## MULTI_DEVICE: **PASS (por construção, não testado ao vivo nesta rodada)**
A chave de efeito é 100% derivada de fatos server-side (userId, aula, marker, itemIdx, conteúdo) — não depende de qual dispositivo fez a chamada. Não há campo de dispositivo na identidade econômica em nenhum dos dois commits revisados. Não executei um teste físico multi-device dedicado nesta rodada (fora do escopo de tempo disponível) — marco como **RETEST_REQUIRED** para um teste físico dedicado futuro, embora a construção não deixe margem estrutural para colisão.

## RECONCILIATION: **PASS**
`markEffectAmbiguous` tem 3 chamadores: `durable-ledger.js` (definição), `ai-cost-protection-gate.js` (gate canônico), `play-billing-rtdn.js` (RTDN) — todos convergindo no MESMO primitivo de `durable-ledger.js`, não há reconciliation paralela para o motor ativo (T00/T02/imagem). O motor metered (inerte) tem sua própria fila (`sim_metered_operations`/`bom_metered_pending_reconciliation_queue`) porque opera sobre tabela própria — não é um segundo caminho concorrente, é o mecanismo do sistema observacional que ainda não está ligado.

## SHADOW: **PASS**
`MICROCREDIT_PRICING_SHADOW` e `MICROCREDIT_PRICING_ENFORCE` confirmados **ausentes** no ambiente real do droplet agora (reconfirmado ao vivo via `/proc/<pid>/environ`). `MICROCREDIT_PRICING_ENFORCE` continua com zero consumidores reais no código (só comentários).

## RATE_CARD / H1 / H7: **PASS**
Nenhum dos 8 commits toca `src/economics/rate-card.js`, `margin-policy.js` ou `fx-policy.js`. H1/H7 permanecem exatamente como fechados na sessão anterior.

## LEGACY PRICING (`T00_PART_CREDIT_COST`/`T02_ITEM_CREDIT_COST`/`IMAGE_CREDIT_COST`): **ACTIVE_AUTHORITY (esperado, não é dano)**
Estas constantes SÃO o motor de cobrança ativo atual em produção (o motor metered/credit_units ainda é só observacional). Como `MICROCREDIT_PRICING_ENFORCE` tem zero consumidores, não há conflito de duas autoridades ativas simultâneas — apenas uma autoridade real (a legada) e um observador inerte (o metered).

## SIGNUP BONUS: **PASS**
`purchaseKey: 'signup-bonus:v1'` fixo por conta → `ledger.grantCredits` → RPC `bom_durable_grant_credits`, que tem checagem de idempotência explícita (`if exists (... idempotency_key=p_purchase_key and movement_kind='purchase') then return`) **e** constraint real de banco `unique (account_id, idempotency_key, movement_kind)` em `sim_credit_ledger`. Exactly-once garantido em dois níveis independentes.

## PLAY GRANTS: **PASS**
`sessionId: google_play:${productId}:${stablePurchaseId}` (orderId do Google ou hash do purchaseToken) — mesmo caminho `ledger.grantCredits`/`bom_durable_grant_credits`, mesma proteção dupla (app + DB constraint). RTDN (`src/play-billing/play-billing-rtdn.js`) dedupa por `effectKey = play-rtdn:${messageId}`, protegido pela mesma unique constraint de `sim_durable_operations`.

## CABES_SIM_CONFORMANCE: **PASS**
Nenhuma violação encontrada nos 8 commits: uma autoridade por verdade preservada (durableLedger é a única fonte), dinheiro fora do processo (Postgres/Supabase), Redis não vira cofre (nenhum dos 8 commits grava saldo em Redis), servidor substituível (nada hardcoded ao processo), idempotência reforçada (não enfraquecida), "acknowledged means durable" preservado (recovery só responde depois de `acceptEffect`/`persistDurableResult`), fail-closed em ambíguo preservado.

## PARALLEL_ECONOMIC_PATH: **NONE**
Os dois commits econômico-sensíveis reusam 100% dos primitivos existentes (`durableLedger`, `economicContext`, `assertBudgets`, `assertCircuit`, `credits-store`). Nenhum novo módulo, helper ou rota de cobrança paralela foi criado.

## DEAD_LEGACY: **NONE encontrado nos 8 commits**
Grep por `TODO`/`FIXME`/`deprecated` nos arquivos econômicos centrais (`ai-cost-protection-gate.js`, `src/economics/*.js`, `src/credits/*.js`, `src/durable/*.js`): zero ocorrências.

## UNEXPLAINED_PROVIDER_COST: **R$ 0,00**
## TOTAL_PROVIDER_COST_AUDITED: **R$ 5,0268** (295 chamadas, últimos ~2 dias, produção real, RPC `bom_ai_cost_report_v2`)

Breakdown que soma exatamente ao total:
- `t02`: 177 chamadas, R$2,7294
- `visual_router`: 91 chamadas, R$1,6085
- `t00`: 24 chamadas, R$0,6802
- `attachment`: 3 chamadas, R$0,0087
- 100% modelo único: `gemini-3.1-flash-lite`. 0 falhas. 9 retries (~3%, normal). p50=R$0,016, p95=R$0,026, p99=R$0,032 — sem outlier caro.

Nota honesta: o valor real medido (**R$5,03**) é um pouco maior que a estimativa do usuário ("~R$4") — não forcei o número para bater; é o valor real da janela de ~2,5 dias consultada (desde `2026-09-20T00:00 UTC`). O volume (295 chamadas, majoritariamente T02+visual_router) é plenamente consistente com a extensão física do checklist desta mesma sessão (dezenas de aulas completas, imagens geradas, múltiplas tentativas de Amparo/freeze).

### Respostas às 10 perguntas obrigatórias (seção 16)
1. **Custo esperado de testes?** Sim, majoritariamente — 177+91 chamadas T02/visual_router batem com o volume de QA física desta sessão.
2. **Chamadas duplicadas?** Não há evidência (`chargedCallsWithoutReservation:0`); os dois fixes revisados (`42d541d`, `6885043`) existem exatamente para PREVENIR duplicação em restart/reinstall, e ambos passaram nos testes que provam isso.
3. **Retries em excesso?** 9/295 (~3%) — normal.
4. **Fallback desnecessário?** Sem sinal no agregado (0 indeterminate, 0 failed); não medido por chamada individual nesta rodada.
5. **Imagem/provider caro?** `visual_router` é 32% do custo total, mas p95/p99 não mostram outlier — proporcional ao volume.
6. **Recovery chamando provider de novo?** O corredor de recovery só absorve UMA chamada por operação comercial já cobrada, e o agregado não mostra sinal de disparo em massa (`capturesWithoutAcceptedMaterial:0` no total). Não consegui isolar quantas das 295 chamadas foram especificamente `financialDisposition:absorbed` nesta consulta agregada — limitação reconhecida.
7. **Chamadas sem cobertura?** Não — `chargedCallsWithoutReservation:0`, e reconfirmado ao vivo agora pelo teste `coverage_before_provider_call_contract`.
8. **Custo absorvido pelo sistema?** Possivelmente algum (recovery corridor), mas não quantificável separadamente nesta consulta — limitação reconhecida.
9. **Cobrança correta do aluno?** Sim — 143 capturas de 144 reservas, 1 liberada corretamente, sem órfãs.
10. **Chamada não atribuível?** Não — `callsWithoutCorrelation:0`.

## ROOT CAUSES
Ambos os bugs econômico-sensíveis corrigidos nos 8 commits (`42d541d`, `6885043`) tinham causa raiz real e documentada (identidade econômica instável / material durável ausente após reinstalação), com autorização formal e teste que prova a correção — não foram "consertos às cegas".

## FIXES_REQUIRED
Nenhum. Nenhuma violação foi encontrada nos 8 commits pós-baseline.

## READY_FOR_STRUCTURAL_CLEANUP_AUDIT: **YES**

---
**Suíte completa do servidor**: 139/139 arquivos de teste passaram (rodado agora, ao vivo, incluindo o teste de no-coverage-no-provider). Nenhum código foi alterado nesta auditoria — só leitura, consultas de produção real (via RPCs já existentes, sem tocar segredo) e reexecução de testes já existentes.
