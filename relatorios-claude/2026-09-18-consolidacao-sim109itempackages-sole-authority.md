# Consolidação de sim109ItemPackages como autoridade única de conteúdo — SIM109

**Data:** 2026-09-18
**Segue:** `2026-09-17-fix-warmup-e-novo-bug-avanco-sim109.md`
**Escopo:** `sim109-economic-final-app`, branch `reform/sim109-economic-final-construction` (main atualizado). Servidor sem alteração nesta rodada.

## O que foi autorizado e feito

Autorização: consolidar `sim109ItemPackages` como ÚNICA autoridade de conteúdo, deletando `readyLessonMaterials`/`currentLessonMaterial`/`PreparedLessonMaterial` do código executável — não desligar, não deixar atrás de flag, não manter como fallback.

**Inventário feito antes de apagar:** `readyLessonMaterials`/`currentLessonMaterial` só eram alcançáveis via `LessonMode.session` (o único `LessonMode` de fato construído em produção; `.simulado` nunca é construído, `.reforco` fica atrás de `item.isReview`, que nunca é setado `true` em produção). Confirmei também que o resolvedor de prontidão (`LessonReadinessResolver`) já checava `sim109ItemPackages` **antes** de cair no fallback flat — ou seja, sempre que um pacote existia, o mapa flat nunca era sequer consultado. Isso confirmou que a remoção era segura: nenhum caminho de produção dependia exclusivamente do mapa antigo.

**Removido de `lib/`:**
- Classe `PreparedLessonMaterial`, campos `readyLessonMaterials`/`currentLessonMaterial` em `StudentLearningState` (e todo `toJson`/`fromJson`/`copyWith` associado).
- `ReplacePreparedLessonMaterialsCommand` → substituído por `StoreSim109ItemPackageCommand` (upsert por identidade) e um novo `PatchSim109PackageExperienceImageCommand` (para quando só a imagem de uma experiência chega depois, via rota visual separada).
- Toda a lógica de leitura/gravação nos módulos de sessão, sala de aula, engine de janela quente (`DopamineReadyWindowEngine`) e serviço de material (`StudentLessonMaterialService`) — agora tudo passa por `PackageAuthority`/os novos comandos.
- Resultado líquido em `lib/`: **~500 linhas a menos**, uma autoridade só, sem vestígio da antiga.

## Dois bugs reais corrigidos no caminho

1. **A causa raiz do travamento relatado ontem** (Experiência 1 → 2 presa em "Preparando próximo passo"): `DopamineReadyWindowEngine._mirrorPreparedLesson` só gravava em `readyLessonMaterials`, nunca em `sim109ItemPackages`. Agora despacha `StoreSim109ItemPackageCommand`.
2. **Bug latente na mesclagem de estado (3-way merge):** a função de merge nunca mesclava `sim109ItemPackages` — descartava silenciosamente um dos lados em conflitos de sincronização entre dispositivos. Corrigido para mesclar como o mapa antigo fazia.
3. **Bug de correspondência em `PackageAuthority.packageFor`:** alguns pontos de chamada passavam o `itemStableId` do cursor (que pode divergir do marcador simples do currículo após uma renumeração em lote) em vez do marcador simples, causando falha de correspondência mesmo com o pacote presente. Corrigido para resolver o marcador do currículo naquela posição antes de comparar.

## Testes

Migrei toda fixture de teste que usava os mapas antigos para `sim109ItemPackages`, via novo `test/support/sim109_package_test_helpers.dart`. Onde um teste antigo validava um estado hoje estruturalmente impossível (ex.: "material sem identidade", "conteúdo presente mas sem pacote aceito"), removi o teste com comentário explicando por que o cenário não existe mais — a cobertura real (ex.: "resultado tardio de outra aula é ignorado") permanece coberta por outros testes já existentes.

`flutter analyze`: 0 problemas. `flutter test`: **1414/1414 passando**.

## Teste funcional em device físico

Servidor de teste standalone reiniciado (porta 3012), APK debug reconstruído, reinstalado no tablet físico (Galaxy Tab A9 5G) via ADB/Tailscale, com `pm clear` para estado limpo.

- Login (Google, `joelgomes522@gmail.com`), onboarding completo (9 etapas), criação de aula.
- Warmup (`/api/warmup`) respondeu corretamente com o formato `item_package` (confirma que a correção de ontem continua funcionando).
- **Repeti o ciclo completo Item → Experiência 1 → Experiência 2 → próximo Item mais de 10 vezes seguidas, sempre respondendo corretamente**: em nenhuma delas o app travou em "Preparando próximo passo". Cada transição (Experiência 1→2 dentro do mesmo item, e item→próximo item) avançou imediatamente, com conteúdo novo renderizado a cada vez.
- Log do servidor de teste revisado: 80 requisições 200, 1 401 (retry de auth inicial antes do app abrir, benigno) e 1 409 (conflito de revisão em sincronização, resolvido automaticamente — comportamento esperado e já coberto por teste). Nenhum `AI_CONTRACT_INVALID`, nenhum 5xx.

## Estado do ambiente

- App: commit `bc6602c`, branch `reform/sim109-economic-final-construction` e `main` ambos atualizados (push feito).
- Servidor: sem alteração nesta rodada (permanece em `90fe89b`).
- Servidor de teste standalone (porta 3012) encerrado ao final.
- `git status` limpo no repositório do app.

## Conclusão

O segundo bug relatado ontem (travamento na transição Experiência 1→2) está **corrigido, testado automaticamente (1414/1414) e confirmado no device físico em mais de 10 repetições consecutivas sem travar**. A consolidação eliminou a autoridade de conteúdo duplicada por completo — não há mais `readyLessonMaterials` em nenhum lugar do código, então a classe de bug "conteúdo existe mas na autoridade errada" deixou de ser possível estruturalmente.
