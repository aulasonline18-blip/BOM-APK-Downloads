# Fix: preparação do próximo item travava até 10 minutos

Data: 2026-09-18

## Causa raiz

Encontrada durante teste ao vivo do scroll: o item "Preparando próximo passo" ficava travado indefinidamente.

Dois problemas, um em cada lado:

1. **App**: a única tentativa de preparar o próximo item acontece uma vez, na transição E1→E2. O loop que fica checando o status a cada ~2s (`ensureNextAulaAdvancePrepared` em `lib/features/session/lab_session.dart`) só relia uma flag local — nunca tentava de novo pela rede.
2. **Servidor**: em `src/ai/ai-cost-protection-gate.js` existem dois gates (um baseado em arquivo, usado localmente/dev; um baseado em Redis, usado em produção). Nos dois, uma falha genérica do provedor (sem ser rate-limit explícito, ex: timeout, erro de rede, 5xx inespecífico) era classificada como "recuperável" mas nunca recebia um horário de retry (`retryAfterAt` ficava `null`). Resultado: a operação ficava bloqueada até o TTL cheio do registro de falha (até 10 minutos), sem nenhuma forma de destravar antes disso.

Juntos: o app nunca pedia de novo, e mesmo que pedisse, o servidor recusaria por até 10 minutos.

## Correção

**Servidor** (`src/ai/ai-cost-protection-gate.js`):
- Falhas genéricas transitórias (não rate-limit) agora recebem até 3 tentativas automáticas dentro do mesmo pedido, com espera curta e crescente (250ms–5s, com jitter) — reaproveitando o helper `retryWithBudget` que já existia e já era usado no lado de geração de imagem.
- Se mesmo assim continuar falhando, o registro de falha passa a liberar novo retry em ~15 segundos (config `AI_COST_PROVIDER_LIMIT_RECHECK_MS`), em vez de só depois do TTL de 10 minutos.
- Rate-limit explícito do provedor (429) continua sem retry automático apertado — isso hoje já usa seu próprio tempo de espera (respeitando o `Retry-After` do provedor), e não deve ser tocado por essa mudança.
- Mesma chave de idempotência em todas as tentativas — nenhum risco de cobrança duplicada.

**App** (`lib/features/session/lab_session.dart`):
- `ensureNextAulaAdvancePrepared` agora tenta de novo preparar o próximo item quando não está pronto, mas de forma limitada: no máximo 4 tentativas extras, com pelo menos 20 segundos entre elas, reiniciando a cada item novo. Evita repetir o loop apertado que já causou um pico de erro 429 numa tentativa anterior nesta mesma sessão de trabalho.

## Testes

- Servidor: **101/101** arquivos de teste passando (rodado duas vezes). Inclui teste novo (`test/generic_transient_ai_failure_recovery_contract.test.js`) provando: (1) falha transitória se autocura dentro do mesmo pedido; (2) falha persistente libera retry em segundos, não em 10 minutos; (3) rate-limit explícito continua intocado.
- App: `flutter analyze` limpo, **1431/1431** testes passando (rodado do zero, sem cache).

## Arquivos alterados

Servidor (`/root/Servidor-BOM`):
- `src/ai/ai-cost-protection-gate.js`
- `test/finance_p0_alignment_contract.test.js`
- `test/play16_financial_anti_leakage_contract.test.js`
- `test/t02_reservation_release_on_provider_failure_contract.test.js`
- `test/mandatory-tests.manifest`
- `test/generic_transient_ai_failure_recovery_contract.test.js` (novo)

App (`/root/worktrees/sim109-nplus1-scroll`):
- `lib/features/session/lab_session.dart`

## Status

Commitado e publicado:
- Servidor: `7fd99e8` em https://github.com/aulasonline18-blip/Servidor-BOM (branch `main`)
- App: `89ca488` em https://github.com/aulasonline18-blip/BOM (branch `feature/nplus1-image-and-canonical-scroll`)

Auditoria completa do servidor (comparação com padrões ideais de engenharia): https://raw.githubusercontent.com/aulasonline18-blip/BOM-APK-Downloads/main/relatorios-claude/2026-09-18-auditoria-servidor-retry-resiliencia-sim.md
