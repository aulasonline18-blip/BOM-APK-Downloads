# Incidente: `sim-api.service` real rodando com `IS_PRODUCTION=false` — créditos vivem em arquivo JSON local, não no Supabase

**Data:** 2026-09-19
**Severidade:** Alta (integridade/durabilidade de dados financeiros), mas impacto real limitado (ver seção "Por que a gravidade prática é menor do que parece").
**Status:** Investigado e reconciliado por completo. Nenhuma mudança feita em produção. Aguardando decisão do usuário sobre quando/como executar o cutover.

## O que foi encontrado

Durante a implementação da precificação metrificada de microcréditos (branch `feature/metered-microcredit-pricing`), ao investigar se o `credits-store.js` distribuído (Redis) é alcançável em produção, descobri que o serviço `sim-api.service` — o processo Node real e ativo (`PID 1183316`, rodando `/root/Servidor-BOM/server.js`, no ar há mais de 1 dia servindo tráfego real) — **não está usando o ledger durável do Supabase**. Ele está usando o **armazenamento local em arquivo JSON** (`.data/credits-ledger.json`), que é explicitamente marcado no código como `productionAllowed: false` (uso previsto apenas para desenvolvimento).

### Cadeia causal (verificada, não inferida)

1. `systemd` (`/etc/systemd/system/sim-api.service`) define apenas `Environment=NODE_ENV=production` — nenhum `EnvironmentFile`.
2. `src/app/router.js` chama `loadEnv()` antes de `readConfig()`. `loadEnv()` só define uma variável **se ela ainda não existir** em `process.env`.
3. `/root/Servidor-BOM/.env` (arquivo do working tree, carregado em todo boot) tem `APP_MODE=development` na linha 37. Como o systemd nunca define `APP_MODE`, `loadEnv()` a define a partir do `.env`.
4. `appMode = process.env.APP_MODE || process.env.NODE_ENV || 'development'` — **`APP_MODE` é verificado antes de `NODE_ENV`** — então o processo real resolve `appMode = 'development'`, `IS_PRODUCTION = false`.
5. Reproduzido exatamente (mesma sequência de chamadas do `router.js`, mesmo `.env`, mesmo `NODE_ENV` inicial) via `node -e`: resultado `IS_PRODUCTION: false`, `DURABLE_LEDGER_MODE: 'disabled'`. Confirmado por **duas** funções de detecção de produção independentes no código (`env.js` e `media-cache.js`), que chegam à mesma conclusão pelo mesmo motivo.
6. Com isso, `credits-store.js` cai no armazenamento local em arquivo (não no Supabase, não no Redis).
7. **Causa adicional, independente:** o `/root/Servidor-BOM/.env` ao vivo **não contém `SUPABASE_SERVICE_ROLE_KEY`** (nem `DURABLE_LEDGER_SERVICE_ROLE_KEY`) — só `SUPABASE_ANON_KEY` e `SUPABASE_JWT_SECRET`. Ou seja, mesmo corrigindo o bug do `APP_MODE`, o ledger durável ainda não funcionaria hoje, porque a credencial de serviço nunca foi copiada para o `.env` ao vivo. Essa credencial real existe, mas está apenas em `/etc/sim/phase-b-real-supabase.env` (arquivo mais antigo que o `.env` atual, nunca referenciado pelo `sim-api.service`).

## Reconciliação completa (538 contas, 100% cobertas)

Fiz backup imediato e imutável do arquivo local (`chmod 400`, hash SHA-256 conferido) antes de qualquer investigação adicional, em `/root/incident-2026-09-19-app-mode-credits/`.

Em seguida, consultei o Supabase real (somente leitura, via a RPC já sancionada `bom_durable_credit_snapshot` — as tabelas em si têm `revoke all` mesmo para `service_role`, por design da migração B2, então não há forma de ler/gravar fora das RPCs oficiais) para cada uma das 538 contas do arquivo local:

| Categoria | Contas | Significado |
|---|---|---|
| Sem nenhuma atividade no Supabase (`balance=0, revision=0, transactions=0`) | **533 / 538 (99%)** | Supabase nunca recebeu nenhuma operação para essas contas. O arquivo local é a **única** história existente. Migração de mão única seria segura (não há nada para sobrescrever). |
| Com atividade divergente no Supabase | **5 / 538** | Ver abaixo. |
| Erro de consulta | 0 | — |

### As 5 contas divergentes são contas de QA/desenvolvedor, não de estudantes reais

Os 5 IDs divergentes (`a03e00cc-...`, `3d927dc5-...`, `e6b42f7c-...`, `5ed88baa-...`, `7cf6000c-...`) aparecem em artefatos de teste físico em tablet já existentes no VM (`FASE2B-*`, `CUT04-*`, `JOEL-SYNC-PHYSICAL-*`, `DRAWER-SYNC-FIX-*`, `TRADING_LESSON_*`) — ou seja, são contas de teste/validação usadas em sessões de auditoria anteriores, provavelmente batendo ora numa instância apontada para o Supabase real (canário Fase B), ora nesta instância de produção com armazenamento local. Não são estudantes anônimos reais. Uma delas (`7cf6000c-...`) tem 2148 revisões no Supabase contra apenas 44 entradas no arquivo local — claramente duas histórias de teste independentes, não a mesma sequência de eventos vista de dois ângulos.

### Por que a gravidade prática é menor do que parece

- **Nenhuma das 538 contas tem qualquer entrada em `purchaseIndex`** — ou seja, **nenhuma compra real (Google Play Billing) está em jogo** nesse arquivo. É saldo promocional/de teste, não dinheiro real de aluno.
- Das 538 contas, apenas **6 têm saldo diferente dos dois valores "padrão"** (a maioria está em `999999`, um valor claramente de semeadura/QA, não uso orgânico). Isso sugere fortemente que a base ainda está em fase de pré-lançamento/QA, consistente com a preocupação do usuário sobre "sobreviver à primeira semana" — a primeira semana real ainda não começou com essa massa de contas.
- O arquivo local É persistido em disco de verdade (não está em memória) e o serviço tem `Restart=always` — um simples restart do processo não perde dados. O risco real é: (a) ausência de backup/redundância automatizada, (b) ausência de garantias ACID sob escrita concorrente, (c) divergência de todo o desenho desta migração, que assumia Supabase como autoridade.

## Plano de remediação recomendado (não executado — aguardando decisão)

Sequência seguindo o mesmo método `EXPAND → BACKFILL → VERIFY → CUTOVER` já usado nas migrações deste projeto, adaptado a este caso específico:

1. **Backup contínuo do arquivo local** até o cutover (feito uma vez; recomendo um cron leve de snapshot enquanto o incidente estiver aberto).
2. **Provisionar a credencial de serviço corretamente**: copiar `SUPABASE_SERVICE_ROLE_KEY` de `/etc/sim/phase-b-real-supabase.env` para o `.env` real (ou, melhor, migrar `sim-api.service` para usar `EnvironmentFile=/etc/sim/production-runtime.env` com essa chave adicionada, em vez de depender do `.env` do working tree, que fica sujeito a edições de desenvolvimento como a que causou este incidente).
3. **Corrigir a precedência do `APP_MODE`**: remover a linha `APP_MODE=development` do `.env` real, ou (mais robusto) alterar `loadEnv()`/`readConfig()` para nunca deixar um `.env` de working tree sobrescrever um `NODE_ENV`/`APP_MODE` já injetado explicitamente pelo systemd — essa é a causa estrutural, e sem corrigi-la o mesmo incidente pode se repetir a cada novo commit no `.env`.
4. **Migrar as 533 contas "limpas"** (zero atividade no Supabase) via uma RPC de importação dedicada (nos moldes de `bom_durable_grant_credits`, mas para saldo bruto + histórico resumido), com o mesmo portão de verificação zero-loss já usado em `202608300002` (`new_balance == old_balance`, tolerância zero).
5. **Tratar as 5 contas de QA separadamente**: não fundir automaticamente; ou (a) resetar a conta de teste no Supabase e reimportar do arquivo local (mais simples, sem risco pois não é dinheiro real), ou (b) simplesmente excluir essas 5 contas da migração automática e deixá-las para o desenvolvedor revisar manualmente.
6. **Só então** cortar `sim-api.service` para `IS_PRODUCTION=true` de fato (com a correção do item 3 já aplicada) — nessa ordem, o corte não causa uma janela em que estudantes reais veem saldo zerado.
7. **Observabilidade pós-corte**: manter o arquivo local montado como leitura, sem escrita, por um período de observação, para detectar qualquer caminho de código que ainda tente escrever nele.

## Evidência bruta

Preservado em `/root/incident-2026-09-19-app-mode-credits/` (todos os arquivos `chmod 400`, somente leitura):
- `credits-ledger.json.backup-20260919T223003Z` — cópia imutável do arquivo local no momento da descoberta (SHA-256 conferido idêntico ao arquivo ao vivo).
- `reconcile_full_results.json` — resultado bruto da reconciliação das 538 contas (saldo local vs. `bom_durable_credit_snapshot` do Supabase, por conta).
- `reconcile_summary.json` — resumo agregado (533 sem atividade no Supabase, 5 divergentes, 0 erros de consulta).
