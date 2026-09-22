# Sweep exaustivo final — Rodada 1 (parcial) + ALERTA CRÍTICO DE CONCORRÊNCIA — 2026-09-22

## ALERTA CRÍTICO — LEIA PRIMEIRO

Durante esta rodada, confirmei via `ps aux` que **outro processo/agente está editando `/root/Servidor-BOM` e `/root/BOM` concomitantemente a esta auditoria**: há um `npm test`/`flutter test` rodando em paralelo, e um script de shell (`sleep 30; git status --short; git log -4 --oneline --decorate; ps -eo pid,etime,cmd | rg 'claude --continue|flutter test|npm test|check-sim-reform'`) ativo, cujo padrão indica outro agente monitorando o mesmo estado que eu.

Encontrei `git status` **não limpo** em `Servidor-BOM` (working tree com mudanças não commitadas em `src/app/router.js`, `test/protected_files_gate.test.js`, `docs/migracao-sim-nv/protected-files.manifest.json`, `docs/AUDIO_PARALLEL_CONTRACT.md`, `docs/migracao-sim-nv/LEI_PROTECAO_TRAVAS_ANTI_LOOP_SIM_NV.md`, `docs/migracao-sim-nv/M2A_CONTRATOS_DE_COMPATIBILIDADE_APP_SERVIDOR.md`) e em `BOM` (`.github/workflows/e0b-governance.yml`, vários arquivos de `Sim organizing/governance/`, `tool/check-sim-reform`, `tool/generate_sim_nv_app_architecture_inventory.js`, `tool/sim_nv_app_architecture_inventory.json`, `tool/sim_nv_app_connections.json`, `test/anti_loop_protection_contract_test.dart`), com timestamps de poucos minutos atrás — ativamente sendo escritos por outro processo, não por mim.

**Eu NÃO toquei nesses arquivos** (só usei Bash/Read/grep nesta rodada, nenhum Edit/Write neles) e **NÃO commitei nada em `BOM` ou `Servidor-BOM`** por causa disso — risco real de corromper trabalho em andamento de outro agente. O diff observado (remoção de referências a uma rota/route-class `audio` já morta desde o commit `a564000` "remove paid AI audio runtime end to end", e atualização de `src/media/audio-controller.js` → `src/media/image-controller.js` no manifest de proteção) **parece ser exatamente o tipo de limpeza de resíduo morto que este próprio sweep deveria encontrar** — plausivelmente outro agente já a está fazendo em paralelo. Recomendo ao coordenador: **não rodar dois sweeps simultâneos nos mesmos repositórios** — confirme com o outro agente/fork antes de prosseguir, para não haver commits conflitantes.

## Baseline confirmado (antes de qualquer coisa)

- FINAL_SWEEP_BASELINE_APP = `dbe570d346488f6af91abd0e49971ad37ca56160`
- FINAL_SWEEP_BASELINE_SERVER = `3628af9fef133370f9974a101ca7a95264fb4bfd`
- FINAL_SWEEP_BASELINE_HANDOFF = `63ad7d5` (commitei aqui um relatório órfão que estava só na VM, referenciado pelo handoff mas nunca commitado: `2026-09-21-fix-warmup-coordinator-nao-resetado-nova-aula.md`)

## Correção importante a um achado anterior

A rodada anterior desta missão reportou `GOVERNANCE = FAIL` no `tool/check-sim-reform`, atribuindo a uma suposta auditoria de worktrees congelados (`/root/worktrees/sim-two-experience-*`) na branch `reform/two-experience-staging`. **Isso estava incorreto.** Rodei `tool/check-sim-reform` do zero, com ambiente limpo (`env -u SIM_APP_ROOT -u SIM_SERVER_ROOT -u SIM_REFORM_BRANCH`) e também no shell normal: em ambos os casos o script usa corretamente seus defaults (`/root/BOM`, `/root/Servidor-BOM`, branch `main`) e retorna **`OVERALL PASS`** limpo, incluindo todos os 16 mutation tests de governança. O `FAIL` anterior foi quase certamente causado por variáveis de ambiente (`SIM_APP_ROOT`/`SIM_SERVER_ROOT`/`SIM_REFORM_BRANCH`) deixadas exportadas na sessão daquele fork específico, não um problema real do script ou do repositório.

**Decisão da seção 19: `check-sim-reform` = MANTIDO_COM_JUSTIFICATIVA.** Já valida corretamente o `main` atual dos dois repositórios reais, sem necessidade de reponte ou aposentadoria.

## Achados do sweep (classificados)

| Item | Classe | Evidência | Ação |
|---|---|---|---|
| `src/auth/migrate-legacy-resource-owners.js` (server) | VALID_LEGACY_REQUIRED | Zero consumidor em `src/`/`scripts/`, só testado por `test/phase6_saturation_contract.test.js`. Criado no commit `5d1f4cc`, tocado em `196a5d5` ("retire request-time compatibility paths") — ferramenta de migração de uma fase já encerrada, mantida por rastreabilidade, no mesmo padrão de nunca apagar migrations antigas usado no resto do projeto. | Não remover. |
| `src/media/local-object-storage.js` (server) | VALID_CURRENT (test infra) | Inicialmente parecia órfão (zero require em `src/`), mas é consumido por 2 testes **mandatórios** (`test/mandatory-tests.manifest`): `phase_b3_local_storage_contract.test.js` e `phase_b_final_local_integration_contract.test.js`. Confirmado que `createObjectStorage()` (`src/media/object-storage.js:295-297`) só despacha para `'memory'`/`'s3'` — este módulo nunca é runtime de produção, é infraestrutura de teste local para simular armazenamento durável em disco sem precisar de S3/Supabase reais na suíte. | Não remover — reclassificação correta evitou uma remoção que quebraria 2 testes mandatórios. |
| `src/economics/rate-card.js`, `src/economics/fx-policy.js` (server) | VALID_LEGACY_REQUIRED / aguardando ativação | Zero `require()` real fora dos próprios testes dedicados (`rate_card_structure_contract.test.js`, `fx_policy_contract.test.js`) — não estão sequer conectados ao `shadow-observer.js`/`meter-pricing.js` ainda. São a implementação das decisões H1/H7 já auditada e aprovada (PASS na auditoria econômica, commit `ae642f3`), aguardando a decisão de negócio de ativar `MICROCREDIT_PRICING_ENFORCE`. | **Não tocado** — mexer em arquivos econômicos sem motivo concreto é proibido pela seção 13 da missão; "ainda não ativado" não é motivo concreto para remover trabalho de H1/H7 já aprovado. |
| Dependências `pubspec.yaml` (app, 17 pacotes) e `package.json` (server, 7 pacotes) | VALID_CURRENT | Cada pacote confirmado com uso real via grep em `lib/`/`src/`+`scripts/`. Zero `UNUSED_DEPENDENCY`. | Nenhuma ação. |
| Arquivos `.dart` em `lib/` (193 arquivos) e `.js` em `src/` (61 arquivos) | VALID_CURRENT (heurística de alcançabilidade) | Busca por basename referenciado em `lib/`+`test/` (app) e `src/`+`server.js` (server): **zero arquivos totalmente órfãos** encontrados (além dos 2 já analisados individualmente acima). | Nenhuma remoção. |
| `flutter analyze --no-pub` | limpo | `No issues found!` | — |

## O que NÃO foi coberto nesta rodada (seções 7-12, 14-18 da missão)

Por causa do alerta de concorrência (parei de tocar em `BOM`/`Servidor-BOM` assim que descobri edições concorrentes) e do orçamento de uma rodada única, **não completei**:
- Auditoria linha-a-linha de duplicação de autoridade em Dúvida/Revisão/Recuperação/Amparo/Finalização/scroll/CG1 (seções 7-8, 11) — nota: essas áreas já passaram por consolidação dedicada e testada extensivamente em missões anteriores desta mesma sessão (scroll: C1-C7; completion: autoridade única Final-1 a Final-10; organismo das 5 salas: Org-1 a Org-9) — risco residual considerado baixo, mas não re-verificado arquivo-a-arquivo nesta rodada.
- Seções 9-10 (image/media/attachments profundo).
- Seções 14-18 (CABES-SIM revalidação formal, env/config sweep, testes obsoletos, docs desatualizados) além do já coberto na auditoria estrutural anterior (`a4567a1`).
- Seção 22 (suítes finais) — **fiz isso**: `npm test` server = **139/139 PASS**; `flutter test` app = **1508/1508 PASS**. Mas essas suítes rodaram contra o working tree com as edições concorrentes de outro agente presentes (não commitadas) — os números são válidos para o estado atual do disco, não necessariamente para o `HEAD` commitado.

## Por que não avancei para build final (seção 24)

A missão do usuário é explícita: build final só depois de `CLEANUP_SWEEP = PASS` completo (seções 1-23). Dado o escopo não coberto acima e, principalmente, o estado não resolvido de concorrência com outro agente, **não é seguro nem apropriado declarar o sweep completo nesta rodada**.

## Resumo por contagem

- `DEAD_CODE_FOUND` = 0 confirmado e removível com segurança (os 2 candidatos investigados foram reclassificados como legítimos)
- `DEAD_CODE_REMOVED` = 0
- `DUPLICATE_AUTHORITIES_FOUND` = 0 (nesta rodada, escopo parcial)
- `PARALLEL_PATHS_FOUND` = 0 (nesta rodada, escopo parcial)
- `UNUSED_DEPENDENCIES_FOUND` = 0
- `CHECK_SIM_REFORM_DECISION` = MANTIDO_COM_JUSTIFICATIVA (corrigindo o achado anterior de FAIL, que era falso-positivo por variável de ambiente)
- `APP_TESTS` = 1508/1508 PASS
- `SERVER_TESTS` = 139/139 PASS
- `RELEASE_CANDIDATE` = **NÃO** ainda — sweep incompleto (seções 7-10, 14-18 pendentes) e concorrência não resolvida.

## Continue daqui

1. **Primeiro**: resolver a concorrência — confirmar se outro agente está mesmo editando `Servidor-BOM`/`BOM` agora; se sim, deixá-lo terminar e commitar antes de qualquer nova rodada de sweep tocar esses arquivos.
2. Revisar o diff não commitado encontrado (remoção de resíduo `audio` route-class) — se for trabalho legítimo de outro agente, deixá-lo commitar; se for lixo residual de uma investigação anterior, decidir descartar ou finalizar deliberadamente (não foi commitado por mim para evitar decidir por outro agente).
3. Completar as seções 7-10 e 14-18 do sweep nas próximas rodadas.
4. Só depois: suítes finais reconfirmadas contra HEAD limpo, retest causal se algo mudou, e então build final APK/AAB.
