# CG-1 — Currículo Grande Canônico: Fase 1 (mapeamento, contrato, prefetch, cobrança por parte)

**Data:** 2026-09-19
**Status:** Em andamento. Este relatório cobre a primeira fatia entregue, testada, commitada e pushada da missão CG-1. As fases restantes (consolidação de menu, remoção de código morto, fronteira legada, matriz de testes completa, novo APK e validação física) **não** estão cobertas aqui — ver seção "Pendências" no final.

## Base (antes desta fatia)

- APP HEAD (antes): `c13f7f8096112dab426354e471ffc850128669e7`
- SERVER CODE HEAD (antes): `3630a7db25cf35df76d35f3dd2c4c9fd441097c8`
- SERVER DEPLOYED HEAD (produção, droplet): `3630a7d...` (mesmo commit do código antes desta fatia — não mudou nesta fatia)

## Entrega desta fatia

### 1. Mapeamento (Fase 1 da missão)

Investigação direta (leitura de código, não só relatórios de fork) confirmou que a arquitetura CG-1 já existente (documentada em 2026-07-12) está estruturalmente madura:

- `src/t00/t00-contract.js`: `MAX_BATCH_ITEMS=80`, `expectedPartLessonLocalId`, `validateCg1CurriculumPlan` já implementam corretamente particionamento, faixa contígua, anti-repetição, anti-buraco — **mantidos como autoridade única**, sem necessidade de reconstrução.
- `src/t00/native-bootstrap-controller.js`: único caminho oficial para bootstrap T00 (`src/web-startup-engine/**` já removido em missão anterior).
- App: `CurriculumGlobalPlan` e `CurriculumContinuationState` (`student_learning_state.dart`) já minimalistas e corretos — mantidos como estão.
- `curriculum_utils.dart`: autoridade de fato do domínio de currículo no app; continha o único gap concreto confirmado (gatilho de prefetch tardio) — corrigido nesta fatia.
- **Confirmado via grep exaustivo em `src/`:** nenhuma cobrança de crédito pré-existente tocava T00/bootstrap/criação de currículo antes desta fatia (os únicos pontos de cobrança eram T02/item, salas auxiliares, geração de imagem e concessões de compra). A nova regra é, portanto, puramente aditiva — sem risco de cobrança dupla.

### 2. Contrato normativo atualizado (Fase 3)

`Servidor-BOM/ADENDO_CG_1_CURRICULOS_GRANDES.md` passou de v1.0 para v1.1:

- **Seção 16.2 reescrita:** a versão 1.0 permitia o menu mostrar "Parte 1: 80/80, Parte 2: 3/80..." — isso está agora **proibido**. Menu deve mostrar exatamente uma entrada por root, com progresso global (`Matemática financeira — 83 de 300`), nunca a palavra "parte" ou contagem por-parte.
- **Nova Seção 24 (Regra Econômica de Cobrança por Parte):** 1 crédito por parte gerada (incluindo a primeira), identidade econômica determinística (`rootLessonLocalId + partNumber`, nunca timestamp/UUID), fluxo obrigatório reserva→gera→valida→persiste→captura, idempotência, zero cobrança retroativa, zero prefetch em cadeia.
- Invariantes (Seção 17) e casos proibidos (Seção 18) estendidos para refletir as novas regras.

### 3. Prefetch: gatilho corrigido (Fase 5)

`curriculum_utils.dart`: `shouldPrepareNextCurriculumPartAtCurrentPosition` disparava apenas no último item da parte. Agora dispara ao entrar nos últimos 20 itens (`curriculumPartPrefetchWindow = 20`) — para uma parte de 80 itens, dispara a partir do item 61. Guarda de single-flight existente (cliente + `canonicalT00Identity` no servidor) preservada e reverificada; nenhum prefetch em cadeia introduzido (uma parte à frente, nunca duas).

### 4. Cobrança de 1 crédito por parte curricular (Fase 6)

Implementado em `src/t00/native-bootstrap-controller.js`:

- Antes de chamar o provedor de IA: reserva 1 crédito com chave `t00-part:{rootLessonLocalId}:{partNumber}` (resolvida da ficha da requisição, nunca de um id de requisição transitório).
- Em saldo insuficiente: o provedor de IA **nunca** é chamado; evento SSE `fatal` traz `code=INSUFFICIENT_CREDITS`, `statusCode=402`, `humanError.action='buy_credits'` (em vez do "tentar novamente" genérico).
- Após validação completa (`validateCg1CurriculumPlan`) e persistência do contexto privado: captura o crédito.
- Em qualquer falha após a reserva (provedor, validação): libera a reserva antes de propagar o erro.
- Reaproveita o ledger durável existente (`credits.reserveCredit/captureCredit/releaseCredit`, exatamente o mesmo mecanismo já usado por T02 e geração de imagem) — nenhuma infraestrutura financeira nova.
- Aplica-se igualmente à primeira parte de qualquer aula nova e a cada parte de continuação (aulas ≤80 itens também passam a custar 1 crédito).

## Testes

**Servidor** — novo arquivo `test/t00_curriculum_part_billing_contract.test.js` (6 cenários: sucesso parte 1, continuação parte 2 com chave correta, saldo insuficiente sem chamar provedor, falha de provedor libera reserva, falha de validação libera reserva, modo sem `credits` inalterado) — adicionado ao `test/mandatory-tests.manifest`. Suíte mandatória completa (105 arquivos) rodada integralmente: **todos passam**, exceto uma falha pré-existente e não relacionada (`test/m4_classroom_depot_removed.test.js`, confirmada presente no HEAD anterior a qualquer mudança desta fatia via `git stash`, provável poluição de dado de teste local — não é regressão desta fatia). O teste de governança econômica (`rwr001_economic_boundary_static_contract.test.js`, que impede pontos de cobrança dispersos) foi atualizado para reconhecer `native-bootstrap-controller.js` como chamador legítimo.

**App** — novo teste em `test/p2_curriculum_continuation_test.dart` (`CG-1 prefetch fires when entering the last 20 items, not only the last item`) verificando: não dispara a 21 itens do fim, dispara exatamente a 20 itens do fim, prepara exatamente uma parte à frente (sem cadeia). Suíte completa do app (**1436 testes**) rodada: **todos passam**. `flutter analyze`: sem problemas. `dart format`: aplicado.

## Commits e push

| Repo | Commit | Descrição |
|---|---|---|
| BOM (app) | `0d3ac0e` | fix(cg1): trigger next-part prefetch at last-20-items, not last item |
| Servidor-BOM | `4cbc6fe` | docs(cg1): mandate full part-invisibility in menu, add per-part billing rule |
| Servidor-BOM | `ac5405a` | feat(cg1): charge 1 credit per curriculum part, including the first |

Push confirmado nos três commits (branches `feature/nplus1-image-and-canonical-scroll` no BOM, `main` no Servidor-BOM).

## HEADs após esta fatia

- APP HEAD: `0d3ac0e`
- SERVER CODE HEAD: `ac5405a`
- SERVER DEPLOYED HEAD (produção): ainda `3630a7d` — **`ac5405a` ainda não foi implantado no droplet de produção.**
- SERVER USADO NO TESTE FÍSICO: nenhum teste físico foi feito nesta fatia (sem novo APK ainda).

## Pendências explícitas (não resolvidas nesta fatia)

Estas fazem parte da missão CG-1 completa e continuam pendentes:

1. **Tabela KEEP/ADJUST/DELETE formal** por arquivo/símbolo (Fase 2) — mapeamento já feito, tabela escrita ainda não formalizada em documento separado.
2. **Consolidação de autoridade de menu/projeção** (Fase 7): `lab_session_curriculum_menu.dart` ainda contém lógica de inferência de domínio (regex de número de parte, fallback de batch-start) que deveria migrar para `curriculum_utils.dart`. Hoje já produz uma linha por root corretamente — o problema é onde a lógica mora, não um bug funcional confirmado.
3. **Remoção de código morto/aliases** (Fase 8): cadeia de fallback de 8 fontes para `partNumber` em `student-state-controller.js`, aliases `curriculumId/curriculumStableId/curriculumRevisionId/planId` no app, duas implementações independentes de strip de sufixo `::part-N`.
4. **Fronteira legada explícita de 20 itens** (Fase 9): nenhuma constante literal de "lote de 20 itens" foi encontrada no código atual (server ou app); só evidência de compatibilidade é o regex de sufixo `-p\d+`/`::part-\d+`. Precisa de investigação adicional para confirmar se o cenário histórico de 20 itens existe em dados reais.
5. **Matriz de testes adversariais completa** (Fase 10): testes dedicados de fronteira em 85/100/180/300/400 itens, teste de menu com 1 card para root de 4 partes, teste de rename/delete, teste multi-dispositivo, teste de restart — ainda não escritos.
6. **Novo APK, validação física e relatório integrado final** (missão + adendo): nenhum novo APK foi gerado nesta fatia; nenhuma validação física na tablet (fronteira 80→81, saldo antes/depois, menu, restart) foi feita. Isso requer uma fatia dedicada subsequente.
7. **Pendências de missões anteriores permanecem em aberto** (não escondidas): RTDN para compras PENDING, política de refund/revoke, guarda de propriedade em `/api/student-state/persist`, storage seguro, R8, staleness incidental de Revisão/Recuperação.

## Veredito desta fatia

O que foi entregue está **verificado automaticamente** (servidor + app, suítes completas verdes, `flutter analyze` limpo) e **commitado/pushado**. Não há validação física nem AAB nesta fatia. A missão CG-1 completa (94 seções + adendo de 25 seções) permanece em andamento; esta fatia cobre as Fases 1, 3, 5 e 6 integralmente e deixa registradas, sem disfarce, as Fases 2, 4 (parcial), 7, 8, 9, 10 e a validação final como pendentes.
