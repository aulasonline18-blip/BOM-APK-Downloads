# CG-1 — Currículo Grande Canônico: Fase 1 (mapeamento, contrato, prefetch, cobrança por parte)

**Data:** 2026-09-19
**Status:** Em andamento. Este relatório cobre as fatias 1, 2 e 3 (mapeamento/contrato/prefetch/cobrança; consolidação de mapeamento e tabela de decisão; correção de bug real de exclusão/renomeação + matriz de testes de fronteira) da missão CG-1. Pende ainda: novo APK e validação física — ver seção "Pendências" no final.

## Atualização (fatia 3, mesma data): bug real de exclusão/renomeação + matriz de testes de fronteira

Ao investigar o requisito de teste "renomear/excluir uma trilha de currículo" (parte da Fase 10), foi encontrado um **bug funcional real**, não apenas uma lacuna de teste: `deleteLocalLesson`/`renameCloudLesson`/`deleteCloudLesson` (`lab_session_drawer_controller.dart`) sempre operaram sobre um único `lessonLocalId`. Como o menu colapsa um currículo de múltiplas partes em UMA linha representativa (`groupCurriculumLessonSummaries`), excluir ou renomear pelo menu só afetava a parte representativa — as demais partes do mesmo root ficavam órfãs (não excluídas, ou com nome antigo), violando diretamente a invariante da missão ("delete tombstones the whole trail as one entity", "rename renames the root projection... all parts keep belonging to the same root").

**Correção:** as três funções agora aceitam `relatedLessonLocalIds` (a lista completa `curriculumPartLessonIds` que a linha do menu já carrega) e cascateiam a mesma operação para todas as partes, com falha de uma parte relacionada tratada como best-effort (nunca desfaz a operação primária já aceita; será repetida no próximo open/sync). Wiring completo: `lab_session.dart` → `shared_widgets.dart` (handlers `onRename`/`onDelete` do drawer). Dois novos testes dedicados confirmam exclusão e renomeação cascateando para todas as partes. Commit `6d16c41`.

**Matriz de fronteira de particionamento (servidor):** novo `test/cg1_partition_boundary_matrix_contract.test.js` cobre exatamente os totais exigidos pela missão (1, 79, 80, 81, 85, 100, 159, 160, 161, 180, 239, 240, 241, 300, 320, 321, 400): contagem correta de partes, toda parte não-final com exatamente 80 itens, parte final nunca preenchida artificialmente além do total, formato canônico de id, currículo ≤80 nunca dividido, e rejeição de partes artificialmente encolhidas/preenchidas em cada fronteira. Commit `aa5b30f`.

**Menu com 4 partes (app):** novo teste em `lab_session_drawer_controller_test.dart` confirmando que um root canônico de 4 partes (80/80/80/60 = 300 itens) produz exatamente 1 card, com total e progresso globais corretos. Commit `caaf604`.

**Itens da matriz de testes já cobertos por construção (não duplicados):** idempotência econômica multi-dispositivo/concorrência já é garantida genericamente por `test/credits_concurrency_contract.test.js` (duas chamadas concorrentes de `reserveCredit` com a mesma chave convergem para uma única reserva) — como a nova cobrança de T00 usa exatamente esse mesmo mecanismo com uma chave determinística, essa garantia se aplica automaticamente, sem necessidade de um teste duplicado específico de T00. O teste "P2 does not mark next curriculum part ready while it is still partial" já simula dois adaptadores concorrentes (efeito multi-dispositivo) pedindo a mesma continuação e confirma exatamente uma chamada. O round-trip `StudentLearningState.fromJson(part2.toJson())` no teste de continuação já cobre persistência/restart.

**Ainda não escrito:** testes de integração ponta-a-ponta dedicados para currículos de 300/400 itens com provedor fake simulando múltiplas partes reais em sequência (a matemática de particionamento para esses totais já está coberta pela matriz acima, mas não o fluxo completo de geração sequencial de todas as partes de um currículo de 400 itens).

Suítes completas rodadas após esta fatia: servidor (mandatória, 106 arquivos, mesma falha pré-existente não relacionada) e app (1439 testes) — todos passam. `flutter analyze` limpo.

## Atualização (fatia 2, mesma data): consolidação de mapeamento + tabela de decisão

Após a fatia 1 (commits `0d3ac0e`, `4cbc6fe`, `ac5405a`), foi feita uma segunda fatia:

- **Consolidação de autoridade de mapeamento (Fase 4/7):** `lab_session_curriculum_menu.dart` tinha cópias privadas do regex `::part-N` e da fórmula `batchStart = (partNumber-1)*80+1`, duplicando lógica equivalente já existente em `curriculum_utils.dart` (para um tipo de dado diferente — `StudentStateSummaryRow` vs `StudentLearningState`). Extraídas para `curriculum_utils.dart` como funções puras (`rootLessonIdFromRawId`, `partNumberFromRawId`, `batchStartForPartNumber`, `curriculumMaxBatchItems`), reutilizáveis por qualquer tipo. O menu agora chama essas funções em vez de reimplementá-las. Sem mudança de comportamento — mesmas fórmulas, um único lugar. Commit `15a21f0` (app), suíte completa (1436 testes) e `flutter analyze` verificados após a mudança.
- **Investigação de código morto (Fase 8):** confirmado via grep direcionado que os aliases legados (`splitPartNumber`, `global_plan` em snake_case) **nunca são escritos** em nenhum lugar do app ou do servidor atual — só existem como fallback de leitura em `student-state-controller.js`'s `summary()` (servidor) e em `_planMapFrom`/`CurriculumGlobalPlan.fromJson` (app). Não há, portanto, "código morto" para apagar nesse cluster: é código de leitura legado genuinamente vivo (serve linhas antigas persistidas), não um caminho paralelo ativo concorrendo com o canônico. Apagar quebraria o menu de lições antigas reais — não foi feito.
- **Fronteira legada de 20 itens (Fase 9):** confirmado (nesta e nas duas investigações por fork anteriores) que não existe nenhuma constante literal de "lote de 20 itens" no código atual. A única evidência de compatibilidade legada é o sufixo de id `::part-N`/`-pN`, que já é tratado como leitura (nunca escrita) pelas funções acima. O cenário de currículos históricos de 20 itens, se existir em dados reais de produção, é coberto pelos mesmos fallbacks — não foi encontrado um caminho de código que crie novos lotes de 20 itens hoje.
- **`student-state-controller.js`'s `summary()`** (cadeia de 8 fontes para `partNumber`, 6 para `rootLessonLocalId`): deliberadamente **não modificado** nesta missão. É uma função pura de leitura/projeção (não decide billing, não cria partes, não escreve estado) — já satisfaz o requisito de "fronteira legada somente-leitura" estruturalmente. Reescrevê-la é uma mudança de alto risco (alimenta o menu de todos os usuários existentes em produção) para um ganho de manutenibilidade, não de correção. Registrado como decisão consciente de não-alteração, não como pendência esquecida.

### Tabela KEEP/ADJUST/DELETE

| Arquivo/Símbolo | Antes | Decisão | Por quê | Depois |
|---|---|---|---|---|
| `src/t00/t00-contract.js`: `MAX_BATCH_ITEMS`, `expectedPartLessonLocalId`, `validateCg1CurriculumPlan` | Autoridade única de particionamento, já correta | **KEEP** | Já implementa faixa contígua, anti-buraco, anti-repetição, formato de id de parte corretamente | Inalterado; ganhou `resolveCurriculumPartEconomicIdentity` como export adicional |
| `src/t00/t00-contract.js`: `normalizePublicT00Event` (evento `fatal`) | Mensagem/ação fixas ("tentar novamente") para todo erro | **ADJUST** | Precisava distinguir saldo insuficiente (402) de erro transitório para acionar UX de compra em vez de retry | Agora expõe `statusCode`, mensagem e `action` corretos por tipo de erro |
| `src/t00/native-bootstrap-controller.js` | Sem nenhuma cobrança | **ADJUST** | Nova regra de 1 crédito por parte precisa reservar antes do provedor e capturar após validar+persistir | Reserva/captura/libera crédito via `credits`, chave `t00-part:{root}:{partNumber}` |
| `src/app/router.js` (wiring do T00) | `createNativeBootstrapController` sem `credits` | **ADJUST** | Necessário para a nova cobrança | Passa `credits` (já existente no módulo) para o controller |
| `src/config/env.js` | Sem constante de custo de parte | **ADJUST** | Preço configurável, padrão 1, mesmo padrão de `T02_ITEM_CREDIT_COST` | Nova `T00_PART_CREDIT_COST` |
| `test/rwr001_economic_boundary_static_contract.test.js` (allowlist) | Não incluía `native-bootstrap-controller.js` | **ADJUST** | Teste de governança que impede pontos de cobrança dispersos; novo chamador legítimo precisa ser reconhecido | Allowlist inclui o novo arquivo |
| `ADENDO_CG_1_CURRICULOS_GRANDES.md` §16.2 | Permitia mostrar "Parte 1: 80/80, Parte 2: 3/80..." no menu | **ADJUST** | Conflitava com a exigência de invisibilidade total de partes | Proíbe explicitamente qualquer exibição por-parte; exige uma entrada por root |
| `ADENDO_CG_1_CURRICULOS_GRANDES.md` (nova §24) | Nenhuma regra econômica documentada | **ADJUST (adição)** | Regra de cobrança por parte precisa de contrato normativo | Nova seção completa (identidade econômica, fluxo, idempotência, zero retroativo, zero cadeia) |
| `curriculum_utils.dart`: `shouldPrepareNextCurriculumPartAtCurrentPosition` | Disparava só no último item da parte | **ADJUST** | Missão exige gatilho nos últimos 20 itens, não só no último | Dispara a partir de `items.length - 20` |
| `curriculum_utils.dart`: `curriculumPlanRootLessonId`, nova extração de funções puras | Regex de sufixo inline dentro de uma função ligada a `StudentLearningState` | **ADJUST** | Lógica de mapeamento de id precisava ser reutilizável fora do tipo de estado completo | Extraídas `rootLessonIdFromRawId`/`partNumberFromRawId`/`batchStartForPartNumber` como autoridade única |
| `lab_session_curriculum_menu.dart`: `_summaryRootId`/`_summaryPartNumber`/`_summaryBatchStart` | Cópias privadas do regex e da fórmula de batch-start | **ADJUST** | Duplicava a lógica de mapeamento agora centralizada | Chama as funções extraídas de `curriculum_utils.dart` |
| `lab_session_curriculum_menu.dart`: `groupCurriculumLessonSummaries` (agrupamento por root, seleção de representante) | Já produzia uma linha por root corretamente | **KEEP** | Funcionalmente correto; só a matemática de id/parte estava duplicada (já corrigido acima) | Inalterado |
| `CurriculumGlobalPlan`, `CurriculumContinuationState` (`student_learning_state.dart`) | Já minimalistas, sem campos supérfluos | **KEEP** | Já correspondem exatamente ao que a missão exige (nenhum campo de fluxo/decisão indevido) | Inalterado |
| `student-state-controller.js`: `summary()` (cadeia de 8/6 fontes de fallback) | Fallback amplo para nomes de campo legados | **KEEP (fronteira legada)** | Função pura de leitura, não decide billing/criação; reescrever é alto risco para todo o menu de produção por ganho só de manutenibilidade | Inalterado; documentado como fronteira legada somente-leitura já conforme |
| `splitPartNumber`, `global_plan` (snake_case) — aliases legados | Lidos em vários pontos como fallback | **KEEP (somente leitura, confirmado sem escrita)** | Grep confirmou zero escritores atuais; servem dados históricos reais | Inalterado |
| `src/web-startup-engine/**` | Caminho paralelo de bootstrap | **DELETE (já feito em missão anterior)** | Sem consumidores, `native-bootstrap-controller.js` é o único caminho oficial (protegido por teste de governança) | Já removido antes desta missão |
| Rotas de crédito públicas legadas | Caminho paralelo de billing | **DELETE (já feito em missão anterior, task #73)** | Substituídas pelo ledger durável | Já removidas antes desta missão |

### Autoridades finais (após esta missão até aqui)

| Autoridade | Implementação única |
|---|---|
| GLOBAL CURRICULUM AUTHORITY | `CurriculumGlobalPlan` (app) + `CurriculumPlan`/`validateCg1CurriculumPlan` (servidor) |
| PARTITION MAPPING AUTHORITY | `t00-contract.js` (servidor); `curriculum_utils.dart`'s `rootLessonIdFromRawId`/`partNumberFromRawId`/`batchStartForPartNumber`/`curriculumPartLessonId` (app) |
| CONTINUATION GENERATION AUTHORITY | `curriculum_utils.dart`'s `buildCurriculumContinuationRequest`/`shouldPrepareNextCurriculumPartAtCurrentPosition` + `student_experience_t00_adapter.dart`'s `prepareNextCurriculumPartOnDemand` |
| CURRICULUM PART BILLING AUTHORITY | `native-bootstrap-controller.js` + `credits-store.js`/ledger durável (servidor) |
| MENU GLOBAL PROJECTION AUTHORITY | `groupCurriculumLessonSummaries` (app), agora usando a autoridade de mapeamento compartilhada |
| SYNC AUTHORITY | Inalterada (já existente antes desta missão) |

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
| BOM (app) | `15a21f0` | refactor(cg1): single partition-mapping authority for root/part id math |
| BOM (app) | `6d16c41` | fix(cg1): cascade delete/rename to every part of a curriculum trail |
| Servidor-BOM | `aa5b30f` | test(cg1): partition boundary matrix at every 80-item seam |
| BOM (app) | `caaf604` | test(cg1): menu produces exactly one card for a clean 4-part 300-item root |

Push confirmado em todos os commits (branch `feature/nplus1-image-and-canonical-scroll` no BOM, `main` no Servidor-BOM).

## HEADs após estas três fatias

- APP HEAD: `caaf604`
- SERVER CODE HEAD: `aa5b30f`
- SERVER DEPLOYED HEAD (produção): ainda `3630a7d` — **nada desta missão (`4cbc6fe`/`ac5405a`/`aa5b30f`) foi implantado no droplet de produção.**
- SERVER USADO NO TESTE FÍSICO: nenhum teste físico foi feito ainda (sem novo APK gerado nesta missão).

## Pendências explícitas (não resolvidas até aqui)

Estas fazem parte da missão CG-1 completa e continuam pendentes:

1. **Testes de integração 300/400 itens ponta-a-ponta** (Fase 10, parcial): a matemática de particionamento para esses totais está coberta pela matriz de fronteira; falta um teste de fluxo completo com provedor fake gerando sequencialmente todas as partes de um currículo de 400 itens (5 partes), provando zero prefetch em cadeia e uma cobrança por parte em um cenário realista de ponta a ponta.
2. **Novo APK, validação física e relatório integrado final** (missão + adendo): nenhum novo APK foi gerado; nenhuma validação física na tablet (fronteira 80→81, saldo antes/depois, menu, restart) foi feita. Isso requer uma fatia dedicada subsequente, incluindo instalação via ADB e confirmação de versão instalada.
3. **Deploy do servidor**: nada desta missão está no droplet de produção — a regra de cobrança e a matriz de validação só existem no código, não em produção, até o deploy ser feito.
4. **Pendências de missões anteriores permanecem em aberto** (não escondidas): RTDN para compras PENDING, política de refund/revoke, guarda de propriedade em `/api/student-state/persist`, storage seguro, R8, staleness incidental de Revisão/Recuperação.

## Veredito até aqui

O que foi entregue está **verificado automaticamente** (servidor + app, suítes completas verdes, `flutter analyze` limpo) e **commitado/pushado**. Não há validação física nem AAB ainda. A missão CG-1 completa (94 seções + adendo de 25 seções) permanece em andamento; até aqui foram concluídas as Fases 1 (mapeamento), 2 (tabela KEEP/ADJUST/DELETE), 3 (contrato normativo), 4 (mapeamento de particionamento consolidado), 5 (prefetch), 6 (cobrança por parte), 7 (projeção de menu consolidada — incluindo a correção do bug real de exclusão/renomeação órfã), 8 (investigação de código morto — nada a apagar com segurança) e 9 (fronteira legada confirmada somente-leitura). A Fase 10 está majoritariamente coberta (matriz de fronteira completa, menu de 4 partes, idempotência econômica por construção, multi-dispositivo, restart); falta apenas o teste de integração 300/400 ponta-a-ponta. Resta a validação final integrada com novo APK físico na tablet — este é o item de maior porte ainda pendente, e não foi tentado nesta sessão por exigir acesso físico ao dispositivo/ADB.
