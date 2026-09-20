# Reconfirmação pré-migração + decisão individual das 5 contas de QA (incidente 2026-09-19)

**Data:** 2026-09-20
**Contexto:** addendum ao relatório `2026-09-19-incidente-app-mode-credit-store-split-brain.md`, antes de qualquer execução real de remediação em produção.

## Reconfirmação (passo 2 do plano aprovado)

Re-executado agora, não reaproveitado do snapshot de ontem:

- Hash do arquivo local ao vivo (`/root/Servidor-BOM/.data/credits-ledger.json`) comparado byte-a-byte contra o backup imutável de 2026-09-19T22:30:03Z: **idêntico** (`sha256: 59bd5ccce1208...`). Nenhuma escrita ocorreu no arquivo desde o backup.
- Contagem de contas: **538** (inalterado).
- Contas com qualquer `purchaseIndex` (compra real via Google Play): **0** (inalterado).
- Reconciliação Supabase (via `bom_durable_credit_snapshot`, somente leitura, mesma via sancionada): **533 contas sem nenhuma atividade no Supabase, 5 divergentes, 0 erros de consulta** — exatamente os mesmos 5 IDs e os mesmos valores de ontem.

Nenhuma das reconfirmações do passo 2 diverge do relatório original. Seguro prosseguir com a preparação dos passos seguintes.

## Decisão individual das 5 contas de QA/dev (passo 6)

Todas as 5 aparecem espalhadas por **meses** de artefatos de auditoria com dispositivo físico (`/root/BOM-audit-artifacts/`), de 2026-08-03 até 2026-08-27 — cobrindo praticamente todo o histórico de desenvolvimento deste projeto. Nenhuma tem qualquer `purchaseIndex` (nenhum dinheiro real envolvido).

| Conta | Saldo local | Saldo Supabase | Revisão Supabase | Evidência de origem | Decisão |
|---|---|---|---|---|---|
| `a03e00cc-efae-43c9-8a54-b179a343f719` | 999894 | 999797 | 422 | Aparece em `CURSOR-WINDOW-ABC-FINAL`, `CUT04-PHYSICAL-RC`, `FASE2B` (ago/2026) | **Excluir** da importação automática — conta de teste ativa em ambos os lados, históricos independentes e não reconciliáveis sem apagar um dos dois |
| `3d927dc5-078f-4b8b-b221-04417640f0c7` | 999967 | 999918 | 176 | Aparece em `AI-COST-BURST-AUDIT`, `AI-COST-REPAIR`, `AMPARO-FALLBACK-RC-115` (ago/2026) | **Excluir** — mesma razão |
| `5ed88baa-371d-4890-b28e-d2c1f1d7b7a4` | 999985 | 999901 | 197 | Aparece em `KIRIBATI-FICHA-EDIT`, `LESSON_OPEN_REPAIR_TRADING_KIRIBATI`, `LESSON_OPEN_RUNTIME_STATE_REPAIR` (ago/2026) | **Excluir** — mesma razão |
| `7cf6000c-4795-475a-bd2b-8b5e05d822c7` | 999955 | 999332 | 2148 | Aparece em `DRAWER-SYNC-FIX-PHYSICAL`, `DRAWER-SYNC-PHYSICAL-FIX-APK-203`, `KIRIBATI-FICHA-EDIT` (ago/2026); 2148 revisões no Supabase contra apenas 44 operações locais — claramente duas histórias de teste diferentes, não a mesma sequência vista de dois ângulos | **Excluir** — mesma razão, com atenção especial: o volume de revisões no Supabase (2148) sugere uso intenso via uma instância apontada para o Supabase real em algum momento, distinto do uso local registrado aqui |
| `e6b42f7c-ccdb-455c-ac53-9f7d14cde21b` | 0 | 1986 | 60 | **Não encontrada** nos artefatos de auditoria por este ID específico; local mostra atividade ZERO (`localOpCount: 0, localBalance: 0`) enquanto o Supabase mostra 60 transações e saldo 1986 | **Excluir** — esta conta nunca teve atividade no armazenamento local ao vivo; toda sua história existe apenas no Supabase (provavelmente criada/usada via uma instância de canário testada diretamente contra o Supabase real, nunca contra este `sim-api.service`). Importá-la do arquivo local seria criar uma conta vazia por cima de uma que já tem história real no destino — o oposto do que a migração deve fazer |

**Decisão consolidada:** as 5 contas ficam **fora** da importação automática (script do passo 5). Cada uma delas já existe no Supabase com sua própria história — a importação automática, que serve para trazer contas que hoje só existem localmente, não deve tocá-las. Ficam para revisão manual do usuário/desenvolvedor numa missão futura dedicada, sem nenhuma urgência (não são estudantes reais, não envolvem dinheiro real).
