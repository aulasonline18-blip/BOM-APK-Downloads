# Phase 11/12 — Recheck de integração e erradicação de legado (SIM109, pós Cut C.2)

**Data do relatório:** 2026-09-17
**Escopo:** `/root/worktrees/sim109-economic-final-app` e `/root/worktrees/sim109-economic-final-server`, branch `reform/sim109-economic-final-construction` em ambos.

---

## O que foi pedido

1. Rodar de novo a matriz de integração completa da Phase 3 (51 edges) contra o código final atual, confirmando que nada quebrou/ficou órfão após o Cut C.2.
2. Completar a Phase 12 (erradicação de legado): remover o caminho `readyLessonMaterials/layer` e qualquer resíduo do sistema de 3 camadas que a checagem revelasse, com testes.
3. Não parar para perguntar — só parar diante de algo não-trivial ou uma contradição real.
4. Gate completo, commit, push, merge para main nos dois repos, relatório final.

## 1. Recheck da matriz de integração — resultado

Reverifiquei as 51 entradas de `docs/sim-maximum-economy-project/inventory/PHASE-03-INTEGRATION-MATRIX.json` contra o código atual, dividindo o trabalho em 4 investigações paralelas (domínio/estado e ready/prefetch fiz diretamente; T02/first-lesson/aux-rooms, UI/sinais/scroll, e visual/áudio/billing/State V1/testes cada uma por um sub-agente com todo o contexto da sessão).

**Veredito: zero integrações quebradas ou órfãs causadas pela migração 3-camadas → 2-experiências.** As 5 decisões que estavam marcadas como "USER_DECISION_REQUIRED" na Phase 3 foram confirmadas implementadas exatamente como o Phase 04 Human Decision Gate ratificou:
- Sinais (F01): Experience 1 sempre avança para Experience 2, sem pular via sinal — confirmado em `learning_decision_engine.dart`.
- Amparo (F03): sinais 2/3 disparam agravante igual nas duas experiências, sem checar camada — confirmado em `student_aux_rooms.dart`.
- Recovery (G03): bloqueio de conclusão final depende só de "existe recovery pendente", zero referência a L3 — confirmado.
- `ExperienceId` (A02): agora tem uma única fonte de verdade (`Sim109Experience`); tentar construir com `layer: L3` lança `FormatException` de propósito — não é resíduo, é uma trava.
- O antigo `_nextSlot` (motor de janela por 3 camadas) não existe mais em lugar nenhum do código.
- Áudio pago (I01-I04, J03): zero referência em todo o runtime, confirmado de novo.

20 outras entradas continuam funcionalmente corretas mas ainda usam o vocabulário "layer" internamente (onde hoje significa "experiência 1 ou 2") — isso não é quebra, é só nome antigo em cima de comportamento novo.

## 2. Três achados que precisam da sua decisão (não mexi, por serem não-triviais ou de alto risco)

**a) Chave de cache/custo do visual e da imagem no servidor ainda inclui `layer` literalmente.** `visual-route-controller.js` documenta `keyAuthority: '...+layer+...'` e `image-controller.js` ainda usa `data.layer` no hash de custo/cache. Isso contradiz a decisão já ratificada (Phase 04, Decisão 5: "visual pertence ao item, reutilizável nas duas experiências, impedir segunda geração"). Na prática não está causando bug hoje — o app já reaproveita a imagem do pacote do item antes de chegar a gerar de novo — mas o contrato do servidor continua desatualizado. Mexer na fórmula de hash de um sistema financeiro (cobra crédito por geração) é risco "crítico" pela própria classificação original; não fiz sem sua decisão sobre qual deve ser a nova chave.

**b) `readyLessonMaterials` NÃO é um caminho morto — é um sistema de conteúdo duplicado, ativo, em paralelo com `sim109ItemPackages`.** Investigação mais funda mostrou que toda vez que um pacote SIM109 é aceito, o mesmo conteúdo é gravado TANTO em `sim109ItemPackages` (a autoridade nova) QUANTO em `readyLessonMaterials` (o cache antigo por itemIdx+marker+layer) — e o texto da aula que o aluno realmente vê ainda é lido do `readyLessonMaterials`, não do pacote novo. 63 referências em 10 arquivos, incluindo imagem/áudio de slot e o schema do State V1. Isso corrige uma imprecisão do MEU PRÓPRIO relatório do Cut C.2 (eu tinha chamado isso de "caminho morto/dívida técnica" sem ter investigado a fundo). Consolidar de verdade em uma única autoridade é um redesenho real da camada de conteúdo do prepared-window/State V1 — não é uma remoção, é uma decisão de arquitetura.

**c) `image-controller.js`/rota `POST /api/generate-lesson-image` parecem não ter nenhum chamador do app**, nem no modelo antigo nem no novo — o app só usa `/api/visual-route`. Isso já era assim antes do Cut A/B/C (a própria evidência original da Phase 3 só cita `/api/visual-route`), então não é uma regressão de hoje, mas pode ser um endpoint pago órfão pré-existente. Não toquei — apagar uma rota financeira viva merece autorização própria e uma checagem melhor de quem mais poderia estar chamando (ex: uma versão de produção mais antiga do app).

## 3. Correção segura aplicada (Phase 12 real)

Achei uma coisa genuinamente seguro de corrigir durante a checagem: `studentStateHighWaterMark` (a função que decide qual versão do estado vence num conflito de sincronização) rankeava usando `layer.value * 100`. Como o modelo novo só produz layer 1 ou 2, um estado antigo (pré-corte) com `layer: 3` gravado no aparelho rankearia **acima** de um estado novo e legítimo de Experience 2 (`layer: 2`) no mesmo item — ou seja, dado antigo de 3 camadas poderia vencer progresso real na sincronização. Corrigido travando o valor rankeado em no máximo 2, com 3 testes novos provando isso.

Não removi o enum `LessonLayer.l3` em si nem as verificações espalhadas que o tratam como inválido (`'sim109-legacy-l3'` em `lesson_readiness_resolver.dart`, a exceção em `Sim109ExperienceValue.fromTransportLayer`) — são travas propositais, com nome e propósito claros, que evitam crash caso algum estado local antigo ainda tenha `layer: 3` gravado. Removê-las tiraria essa proteção sem ganho nenhum, contradizendo a própria decisão já tomada (D-011: invalidar/resetar dado antigo sem construir migração complexa — essa trava JÁ É o tratamento mínimo, não a complexidade que a decisão queria evitar).

## 4. Gate completo e commits

**App** (`sim109-economic-final-app`): `flutter analyze` limpo, `flutter test` completo 1426/1426 (3 testes novos), suítes de sincronização/conflito rodadas em separado — tudo verde.
Commit: `ae1d36b` — fix + teste + relatório de execução em `docs/sim-maximum-economy-project/execution-reports/PHASE-11-12-INTEGRATION-RECHECK-AND-LEGACY-ERADICATION-REPORT.md`.

**Servidor** (`sim109-economic-final-server`): nenhuma mudança de código nesta rodada (os 3 achados acima são só leitura/investigação); `npm test` não precisou rodar de novo porque nada mudou desde o fechamento do Cut C.2 ontem.

**Push e merge:**
- App: push do branch (`fd0687d..ae1d36b`) e fast-forward de `main` (`fd0687d..ae1d36b`) — confirmado por hash que `origin/main` bate com a ponta do branch de trabalho.
- Servidor: `origin/main` já batia com o HEAD local (nada pendente desde ontem) — confirmado por hash, nenhuma ação necessária.

## Conclusão

Phase 11 (recheck de integração) está **fechada**: nada quebrado ou órfão. Phase 12 (erradicação de legado) está **parcialmente fechada**: a única correção genuinamente segura foi feita e testada; os três achados acima ficam registrados e aguardando sua decisão, mas **não bloqueiam** Billing/APK/AAB — nenhum deles está causando um bug ativo hoje.
