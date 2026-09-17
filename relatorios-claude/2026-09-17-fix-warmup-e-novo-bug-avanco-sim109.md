# Correção do bug de warmup + novo bug encontrado no avanço de item — SIM109

**Data:** 2026-09-17
**Segue:** `2026-09-17-teste-funcional-apk-device-sim109.md`
**Escopo:** `sim109-economic-final-server`, branch `reform/sim109-economic-final-construction` (main atualizado). `sim109-economic-final-app` sem alteração de código nesta rodada.

## O que foi corrigido

O bug relatado no teste anterior (`/api/warmup` sempre rejeitado com `AI_CONTRACT_INVALID`, travando o app logo após o primeiro item) tinha causa raiz confirmada: a IA responde ao modo `warmup` do T02 com o mesmo contrato `SIM109_ITEM_PACKAGE_V1` usado pelo modo `lesson`, mas o servidor só aceitava esse formato para `lesson` — `warmup` era validado por um parser antigo, achatado, incompatível.

**Correção aplicada** (commit `90fe89b`, servidor):
- `src/t02/complete-lesson-controller.js`: `normalizeLessonJson` agora aceita `item_package` também para `mode === "warmup"`, chamando o mesmo `normalizeSim109ItemPackage` usado por `lesson`.
- `src/entry-warmup/entry-warmup-controller.js`: `validateWarmup` agora extrai a pergunta-ponte de `item_package.experience_1`, com fallback para o formato achatado antigo (mantendo compatibilidade com qualquer chamador/teste que ainda entregue esse formato direto).
- Nenhuma mudança de contrato para doubt/review/recovery/support/placement, nem em `ai-cost-protection-gate.js`.
- Autorização formal registrada: `docs/migracao-sim-nv/autorizacoes/SIM109-T02-WARMUP-ITEM-PACKAGE-CONTRACT-FIX-2026-09-17.json` (grupo protegido `t02-runtime-contract`).
- Testes novos: `t02_sim109_item_package_contract.test.js` (warmup aceita item_package) e `jw_warmup_welcome_bridge.test.js` (validateWarmup lê de `item_package.experience_1`, incluindo através do controller completo). `npm test`: **98/98 arquivos**. `npm run check:protected`: passa com a autorização.

## Reteste funcional em device físico

Rebuild do APK debug (mesmo `SIM_SERVER_URL`), reinstalado no tablet físico (Galaxy Tab A9 5G) via ADB/Tailscale, com `pm clear` para estado limpo. Login (Google, `aulasonline18@gmail.com`), onboarding completo, criação de aula:

- **`/api/warmup` agora responde 200** (antes: 502 sempre, 3/3 tentativas). A tela "Enquanto sua aula fica pronta" (welcome bridge) renderizou corretamente, com pergunta real e feedback de acerto — **isso não acontecia antes da correção**.
- `/api/complete-lesson` (modo lesson) e `/api/visual-route`: 200, aula criada e exibida (Item 1/20, Experiência 1/2).
- Respondi o Item 1 (alternativa correta) e o sinal de confiança ("Tenho certeza"): ambos persistidos com 200 no servidor.

## Novo bug real encontrado (diferente do anterior, não corrigido)

Depois de responder o Item 1 e o sinal de confiança, o app trava novamente em "Preparando próximo passo" — desta vez a transição de **Experiência 1 para Experiência 2 do mesmo item** (não mais o warmup, que já está corrigido e confirmado funcionando). Reproduzido com o app permanecendo em primeiro plano o tempo todo (sem qualquer interferência minha), sem nenhuma chamada de rede nova por mais de 20 segundos após o servidor confirmar o registro do sinal.

**Causa provável, já documentada anteriormente como achado aberto:** no relatório `2026-09-17-phase11-12-integracao-legado-sim109.md` eu já havia identificado que `readyLessonMaterials` é uma autoridade de conteúdo duplicada e ainda ativa, escrita e lida em paralelo com `sim109ItemPackages` (a autoridade canônica). A rotina que decide se o material da Experiência 2 já está pronto para exibir localmente (`carregarRapidoSePronto` → `readReadyLessonMaterialFromStudentState`, em `lib/sim/classroom/lesson_material_controller.dart` e `lib/sim/lesson/student_lesson_material_service.dart`) consulta `readyLessonMaterials`, não `sim109ItemPackages` — mesmo a Experiência 2 já tendo chegado completa na mesma resposta de `/api/complete-lesson` que trouxe a Experiência 1. Se o espelhamento (`_mirrorCurrentLessonMaterial`/`_mirrorPreparedAndCurrentLessonMaterial`) não grava a Experiência 2 em `readyLessonMaterials` no momento certo, a checagem de "pronto" falha silenciosamente e a tela fica presa sem erro visível — o mesmo padrão de falha silenciosa do bug do warmup, só que num ponto diferente do fluxo.

Esse achado **já estava em aberto, aguardando uma decisão de produto** (não é uma correção pontual de contrato como a do warmup — é uma consolidação/redesenho de qual autoridade de conteúdo o app deve confiar). Por isso não toquei nele agora: a autorização desta rodada foi especificamente para o contrato de warmup do T02, e esse é um problema arquitetural maior, do lado do app, que precisa da sua decisão sobre como consolidar `readyLessonMaterials` e `sim109ItemPackages` antes de qualquer correção.

## Estado do ambiente

- `git status` limpo em ambos os repositórios.
- Servidor: commit `90fe89b`, `main` atualizado (push feito).
- App: sem alteração de código, HEAD inalterado (`f51dc95`).
- Servidor de teste standalone (porta 3012) encerrado ao final.

## Conclusão

O bug relatado (`/api/warmup` travando o app) está **corrigido, testado e confirmado no device físico**. Durante o reteste, o fluxo avançou mais do que antes (chegou a exibir a aula real, Item 1/20) mas encontrou um **segundo bug real, diferente e já documentado como achado em aberto**: a transição de experiência dentro do mesmo item trava pelo mesmo padrão (checagem de prontidão contra uma autoridade de conteúdo que pode não ter sido atualizada). Recomendo decidirmos juntos o caminho de consolidação de `readyLessonMaterials`/`sim109ItemPackages` antes da próxima rodada de correção, já que esse é o bloqueador restante para o fluxo completo funcionar ponta-a-ponta.
