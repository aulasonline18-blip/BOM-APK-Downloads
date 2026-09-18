# Auditoria: servidor SIM vs. padrões ideais de retry/resiliência

**Data:** 2026-09-18
**Segue:** investigação do bug "Preparando próximo passo" travado (item 11), relatado anteriormente
**Escopo:** `/root/Servidor-BOM/src` — comparação das estratégias de retry, timeout, idempotência e proteção econômica contra padrões consolidados de engenharia (AWS "Exponential Backoff and Jitter", idempotency keys no estilo Stripe, Google Cloud client retry guidelines, RFC 7231 Retry-After, circuit breaker de Fowler). Auditoria **somente leitura** — nenhum arquivo de app ou servidor foi alterado.

## O que foi pedido

Comparar o servidor do SIM com um "servidor ideal" em todas as suas funcionalidades relacionadas a resiliência (retry, timeout, bloqueio de custo), listando o que está certo e o que diverge do que a indústria já considera padrão, com localização exata no código.

## Achado principal: dois padrões diferentes convivem no mesmo servidor

O servidor **já contém** uma implementação correta e idiomática de backoff exponencial com jitter — mas ela não é usada em todos os caminhos que deveriam usá-la.

### ✅ Caminho de geração de imagem — ideal, segue o padrão AWS

- `src/app/media-cache.js:107` — `retryDelayMs()`: backoff exponencial padrão (`base * 2^(attempt-1)`, capado em `maxMs`) + jitter proporcional. É exatamente a fórmula do artigo "Exponential Backoff and Jitter" da AWS.
- `src/app/media-cache.js:113` — `retryWithBudget()`: função genérica de retry com número máximo de tentativas (`maxAttempts`), delay calculado por `retryDelayMs`, e um classificador `isRetryable` injetável — decide reexecutar só se o erro for de fato transitório.
- `src/media/image-controller.js:93` — `isTransientProviderError()`: retryable apenas para `429, 502, 503, 504` (erros de infraestrutura/limite), nunca para erros de conteúdo/contrato.
- `src/media/image-controller.js:98-113` — `callImageProviderWithRetry()`: usa os dois acima com `maxAttempts: 3`, `baseMs/maxMs` configuráveis. **3 tentativas automáticas, poucos segundos de espera, depois desiste** — igual ao que apps "ideais" fazem.
- `src/media/visual-route-controller.js:203` — cache de falha recente (`recentFailures`) com TTL padrão de **apenas 1000ms** (`VISUAL_ROUTE_RECENT_FAILURE_TTL_MS`), suficiente só para deduplicar requisições concorrentes — não é um bloqueio punitivo longo.

### ❌ Caminho de conteúdo de aula (texto/N+1) — não segue o mesmo padrão

- `src/ai/ai-cost-protection-gate.js:167-197` — `providerFailureDisposition()`: para erros genéricos/transitórios (não é limite de cota, não é 429 explícito do provedor), o fallback final (linha 196) retorna `{ recoverable: true, retryAt: null }`. O código **sabe** que o erro é recuperável mas nunca calcula uma janela de retry.
- `src/ai/ai-cost-protection-gate.js:1547` — a falha é gravada com `expiresAt: Date.now() + Math.min(resultTtlMs, 10 * 60 * 1000)` — até **10 minutos** de bloqueio, mesmo quando `recoverable: true` e nenhuma cobrança ocorreu.
- `src/ai/ai-cost-protection-gate.js:1140` (`failedOperationRetryReady`) e `:1381-1385` — enquanto esse prazo não vence, qualquer nova tentativa com a mesma chave é **rejeitada ativamente** (`previousProviderAttemptError`), não apenas ignorada — dá exatamente o sintoma relatado: item preso, sem novo pedido ao provedor, sem nova cobrança, mas também sem chance de sucesso.

**Resumo do achado principal:** o servidor não "inventou uma lógica errada do zero" — ele tem duas famílias de código para o mesmo problema (retry após falha de IA), uma seguindo o padrão da indústria (imagem) e outra não (texto/conteúdo de aula, que é o caminho que trava a aula). É inconsistência interna, não desconhecimento do padrão correto.

## Achados positivos (para não haver viés — partes do servidor já ideais)

- `src/distributed/redis-authority.js` (base de toda a camada distribuída, 201 linhas, lida por completo):
  - `rateLimit()` (INCR+PEXPIRE atômico via Lua) e `tokenBucket()` (leaky bucket via Lua) calculam `retryAfter` real a partir do TTL/estado do Redis — não usam número fixo arbitrário.
  - `acquireLock`/`releaseLock` seguem o padrão seguro padrão da indústria (SET NX PX + compare-token-e-delete via Lua), evitando que um processo libere o lock de outro.
  - Todo `command()` tem timeout via `Promise.race`.
- `src/ai/ai-cost-meter.js`: idempotência de custo bem desenhada — `beginAttempt`/`completeAttempt` via RPC no Supabase com hash de correlação (`idempotencyBasis`), limites diários por usuário/aula/global, e status explícitos (`succeeded/failed/indeterminate`) em vez de heurística.
- Caminho de sucesso do gate de custo (`ai-cost-protection-gate.js:1490-1517`): resultado bem-sucedido é salvo com TTL cheio, então se a resposta se perder na entrega (internet caiu depois do servidor responder), o cliente que tentar de novo com a **mesma chave** recebe o resultado em cache, sem gerar nova cobrança. Esse caso ("pedi comida, comida chegou, mas eu não vi") já está resolvido de graça — não precisa de protocolo de confirmação novo.

## O que um servidor ideal faria diferente aqui

1. Timeout/erro de rede transitório → retry automático em segundos, com backoff+jitter, poucas tentativas (2-3), como o próprio servidor já faz em `media-cache.js`.
2. Limite de taxa explícito do provedor (429 com `Retry-After`) → respeitar o valor real informado pelo provedor.
3. Estouro de orçamento/cota (`monthly_spending_cap`) → bloqueio longo e intencional (correto manter aqui).
4. O bug está em tratar o caso 1 como se fosse o caso 3: erro genérico recuperável ganha o mesmo bloqueio de até 10 minutos do estouro de orçamento, sem nunca reclassificar nem reoferecer retry automático limitado.

## Recomendação (não implementada — aguardando decisão)

Aplicar ao caminho de texto/N+1 (`ai-cost-protection-gate.js`) o mesmo padrão já existente em `media-cache.js`: quando `providerFailureDisposition` retornar `recoverable: true` sem `retryAt` explícito de provedor, usar uma janela curta (segundos, não minutos) com poucas tentativas automáticas, preservando a chave de idempotência determinística já existente (`_t02IdempotencyKey`) para não gerar cobrança duplicada. Escopo de mudança pequeno e localizado; não requer nova infraestrutura.

## Conclusão

O servidor não é "todo errado". Ele tem uma peça do padrão ideal (backoff+jitter+tentativas limitadas) já implementada e correta, mas aplicada só na geração de imagem — não no caminho de texto que trava a aula. O restante dos módulos auditados (Redis, medidor de custo, cache de sucesso) segue boas práticas de mercado. O problema é pontual e já mapeado linha a linha.

**Pendente:** decisão sobre implementar o retry curto no caminho de texto (usando a mesma chave de idempotência, sem risco de cobrança duplicada).
