# W8 — Auditoria do corredor econômico protegido — 2026-09-22

**Papel**: janela de auditoria de território (1 de 9), não integradora. Território exclusivo: `ai-cost-protection-gate`, `meter`, `economic context`, `durable ledger`, `credits store`, `reserve/capture/release`, `reconciliation`, `financial identity`, `rate card`, `FX/H1/H7`, `shadow`, `signup bonus`, `Play grants`, `provider billability`, testes econômicos correspondentes.

## Baseline

- BASELINE_APP = `9e66709` (BOM)
- BASELINE_SERVER = `425e052` (Servidor-BOM)
- BASELINE_HANDOFF = `66703f1` (BOM-APK-Downloads)

Não fiz pull/merge de outras janelas depois deste congelamento. Detectei mudanças não commitadas de outra janela em `src/attachments/attachment-processor.js` + 2 testes (fora do meu território) — não toquei nesses arquivos.

## Delta do território desde a última auditoria econômica conhecida

Auditoria anterior desta sessão (`ae642f3`) cobriu `1f3172b..3628af9f`. Reconfirmei do zero, não assumi validade: apenas 2 commits tocaram o território entre `1f3172b` e o baseline atual (`425e052`):

- `42d541d` fix(cost): recover captured T02 material on clean reinstall
- `6885043` fix(cost): keep paid identity stable across restart

Ambos relidos linha a linha nesta rodada (não só stat):

1. **`42d541d`** — corredor de recovery para material T02 já capturado. Fluxo: (a) se a operação comercial original já foi aceita/capturada, faz replay do resultado durável sem tocar provider/ledger; (b) só se o RESULTADO estiver faltando (cobrança já feita), faz UMA chamada de provider adicional sob `effectKey` própria e namespaced (`{creditOperationKey}:result-recovery:v1`), marcada explicitamente `financialDisposition:'absorbed'`, `charged:false`; (c) ambíguo continua fail-closed; (d) reusa 100% dos primitivos existentes (durableLedger, economicContext, assertBudgets, assertCircuit) — nenhum motor paralelo. Teste dedicado (`rwr001_economic_saga_baseline_contract.test.js`) prova com stubs que lançam exceção se hold/take/free forem chamados de novo, e prova idempotência numa segunda chamada.
2. **`6885043`** — remove `accountGeneration` (contador client-side instável entre restarts) da identidade econômica T02/comercial/visual, adiciona `legacyAccountGenerationOperationIdentity` como ponte ESTRITAMENTE DE LEITURA (só replay de resultado durável pré-existente, nunca provider/débito). Autorizado por arquivo de governança formal (`docs/migracao-sim-nv/autorizacoes/SIM-RWR001-RESTART-ECONOMIC-IDENTITY-2026-09-22.json`).

Nenhum dos dois introduz segunda autoridade, rota paralela, ou bypass. Classificação: **VALID_CURRENT** para ambos.

## Verificações de invariante (re-executadas nesta rodada, não herdadas)

- **Reserve/capture/release**: única fonte — `src/credits/credits-store.js` (grep confirma zero outra definição de `reserveCredit`/`captureCredit`/`releaseCredit` em `src/`).
- **Exactly-once estrutural (efeitos durÁveis)**: `unique (account_id, effect_key)` em `sim_durable_operations` (`202608300002_phase_b2_durable_ledger.sql:37`) — backstop de banco, não só disciplina de código.
- **Exactly-once de compra/grant (signup bonus + Play)**: `bom_durable_grant_credits` faz um check-then-act (`exists(...idempotency_key=p_purchase_key...)` antes de creditar), mas isso sozinho seria uma race condition teórica sob concorrência real — **fechada por um segundo backstop de banco** que eu confirmei agora: `unique (account_id, idempotency_key, movement_kind)` em `sim_credit_ledger` (linha 55 da mesma migration). Uma segunda chamada concorrente com a mesma `purchase_key` falha no `insert` do ledger com violação de unicidade, revertendo toda a função (mesma transação implícita do RPC) — sem double-grant possível mesmo em corrida.
- **Reconciliation**: `markEffectAmbiguous` tem exatamente 3 chamadores — `ai-cost-protection-gate.js` (motor ativo), `play-billing-rtdn.js` (RTDN), e a própria definição em `durable-ledger.js`. Um único caminho canônico; a fila própria do motor metered (`sim_metered_operations`/`bom_metered_pending_reconciliation_queue`) opera sobre tabela distinta porque é um sistema observacional inerte, não um segundo caminho concorrente para o motor ativo.
- **Billability**: exatamente 3 estados em `src/economics/billability.js` (`BILLABLE_CONFIRMED`, `NOT_BILLABLE_CONFIRMED`, `INDETERMINATE`) — nenhum quarto estado implícito, `classify()` é a única função exportada de decisão.
- **`MICROCREDIT_PRICING_ENFORCE`**: zero consumidores reais em `src/` (só menções em comentário explicando por que ainda não é consumido) — reconfirmado.
- **Shadow**: `MICROCREDIT_PRICING_SHADOW` gate condiciona `metered-shadow-ledger.js`; sem essa flag, todo método é no-op — reconfirmado por leitura do construtor `createMeteredShadowLedger`.
- **H1/H7**: `margin-policy.js`/`fx-policy.js` intocados desde a auditoria anterior; nenhum commit do delta os referencia.

## Testes executados (território, isolado dos arquivos em edição de outra janela)

39 arquivos de teste rodados individualmente (evitando a suíte completa por causa da interferência não commitada de outra janela em `test/attachment_multipart_hardening.test.js`/`test/server-contract.test.js`, fora do meu território): `economics_*`, `credits_*`, `rwr001_*`, `metered_*`, `fx_policy_contract`, `margin_policy_contract`, `rate_card_*`, `play_billing_*`, `microcredit_*`, `billability_*`, `phase1_ai_economic_meter_contract`, `legacy_credits_routes_removed_contract`.

**Resultado: 39/39 PASS.**

## Saída

```
BASELINE_APP=9e66709
BASELINE_SERVER=425e052
ARQUIVOS_INSPECIONADOS=9 (src/ai/*.js relevantes) + 2 (src/credits/*.js) + 1 (src/durable/durable-ledger.js) + 8 (src/economics/*.js) + 2 (src/play-billing/*.js) + 39 arquivos de teste + 2 migrations SQL relevantes
ACHADOS_CONFIRMADOS=0 (nenhuma correção necessária; 2 commits do delta classificados VALID_CURRENT)
CROSS_DOMAIN_FINDINGS=0
ARQUIVOS_MODIFICADOS=0
ARQUIVOS_REMOVIDOS=0
ROTAS_PARALELAS=0
AUTORIDADES_DUPLICADAS=0
LEGADO_MORTO=0
TESTES_EXECUTADOS=39
TESTES_RESULTADO=39/39 PASS
BRANCH=(nenhuma criada — sem correção comprovada necessária)
COMMITS=(nenhum em BOM/Servidor-BOM; este relatório em BOM-APK-Downloads)
READY_FOR_INTEGRATION=YES

ECONOMIC_AUTHORITIES=1 (credits-store.js — reserve/capture/release; durable-ledger.js — efeitos/reconciliation; ambos únicos em seu papel)
RESERVE_PATHS=1
CAPTURE_PATHS=1
RELEASE_PATHS=1
RECONCILIATION_PATHS=1
DOUBLE_CHARGE=PASS
NO_COVERAGE_NO_PROVIDER=PASS
RWR001=PASS
SHADOW=PASS
H1=PASS
H7=PASS
PLAY=PASS
SIGNUP=PASS
CABES_ECONOMIC=PASS
```
