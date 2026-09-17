# Fechamento do Cut C.2 (SIM109 Economic Final) — Media/Áudio/Econômico

**Data do relatório:** 2026-09-17
**Escopo:** `/root/worktrees/sim109-economic-final-app` e `/root/worktrees/sim109-economic-final-server`, branch `reform/sim109-economic-final-construction` em ambos.
**Método:** execução direta autorizada pelo usuário — implementação, testes e commits nos dois worktrees.

---

## Resumo

O Cut C.1 (relatado em 2026-09-16) tinha deixado 6 pendências explícitas antes de qualquer trabalho de Billing/Play/APK/AAB. As 6 foram fechadas nesta sessão, cada uma com commit próprio, testes rodados isoladamente antes de avançar para a próxima, e um gate completo de app+server ao final. Nenhuma mudança em Billing, staging, Play, AAB ou keystore de produção foi feita — isso continua sendo o próximo passo, não parte deste Cut.

## Os 6 itens

**Item 1 — Autoridade única de pacote atual/próximo.** Criada a classe `PackageAuthority` em `student_learning_state.dart` (app), substituindo buscas manuais em `sim109ItemPackages` espalhadas por 4 arquivos. Todos os 4 pontos de uso foram migrados.

**Item 2 — Limite de "next package" em voo sob restart/reconexão.** Criada uma intenção persistida com TTL de 45s (`Sim109NextPackageIntent`) dentro do próprio `StudentLearningState`, para que um restart do app ou um segundo dispositivo sincronizando a mesma conta não dispare a mesma requisição de novo. O servidor (lock Redis por usuário+item+marker+layer) continua sendo a autoridade real contra duplicidade de custo — isso aqui é só eficiência/latência do lado do cliente.

**Item 3 — Prova do limite do cache visual do servidor.** O cache (`src/app/media-cache.js`, arquivo protegido) já tinha limite (32) e TTL (30 min) corretos — não havia bug, só faltava prova via teste. Dois testes novos provam isso sem tocar no arquivo protegido.

**Item 4 — Remoção real do runtime de áudio de IA pago.** Este foi o item de maior raio de explosão. Removido de ponta a ponta, por autorização explícita sua (remoção real do código, não só desligar via config):
- **App:** cliente `SimServerGeneratedAudioClient`, toda a interface/cache de `GeneratedAudioClient` dentro de `AudioCore`, o método `playDataUrl`, a flag `SIM_ENABLE_AI_AUDIO`, e todo o pipeline de "pré-gerar áudio enquanto a aula carrega". O TTS local (`flutter_tts`) passou a ser o único caminho de áudio — sem mudança de comportamento observável em produção, já que a flag de áudio pago já vinha desligada por padrão.
- **Servidor:** deletado `src/media/audio-controller.js` inteiro, a rota `/api/generate-lesson-audio`, e toda referência ao organ `MEDIA_AUDIO` em `router.js`, `ai-provider-policy.js`, `ai-cost-protection-gate.js` e `env.js`. Isso exigiu criar e usar um arquivo de autorização formal (`docs/migracao-sim-nv/autorizacoes/SIM109-CUT-C2-PAID-AUDIO-REMOVAL-2026-09-16.json`) porque esses arquivos estão no grupo de proteção anti-loop.
- O modelo de estado "State V1" (o sistema de `ArtifactType.audio`/`audioReadiness` dentro do `StudentLearningState`) foi deliberadamente **não tocado** — está fora de escopo por decisão prévia registrada no Phase 04 decision gate. Um único ponto de chamada residual desse sistema (`lab_session_classroom_interaction_controller.dart`) foi ajustado para retornar falha imediata sem tentar gerar áudio pago, preservando o comportamento que já existia (a flag já estava desligada).
- Sobrou trabalho de limpeza de teste considerável: 7 arquivos de teste do app e 12 do servidor tinham testes ou fixtures presos ao áudio pago; todos foram ajustados (removendo o que não fazia mais sentido, mantendo a cobertura equivalente do lado de imagem).

**Item 5 — Taxonomia de falha econômica por provedor.** Criado `test/rwr001_economic_provider_failure_taxonomy_contract.test.js` no servidor, nomeando e provando a taxonomia que já existia implicitamente no código: falha antes do provider responder e respostas conhecidas (429/503) podem tentar outro provider com segurança; um timeout ou erro de rede ambíguo (pedido saiu, resultado desconhecido) nunca pode cair automaticamente para outro provider (risco de cobrança dupla); contrato inválido e limite mensal de gasto nunca caem para outro provider. Testado via `shouldFallback()` (a função real de decisão) e via `createGovernedTextClient` com provedores fake, sem tocar no arquivo protegido do gate de custo.

**Item 6 — Gate completo + modelo de escala.**
- **Servidor:** `npm test` (98/98 arquivos), `npm run check:protected` (limpo), `npm run check:server-minimum`, `npm audit --omit=dev` (0 vulnerabilidades), teste de identidade em escala (10.000 operações, todas únicas), `git diff --check` e `node --check` em todo `.js` alterado — tudo verde.
- **App:** `dart format` (corrigido o que este trabalho introduziu; 5 arquivos com formatação antiga e não relacionada a este Cut foram deixados de lado, são parte de uma limpeza geral futura), `flutter analyze` limpo, suíte completa `flutter test` (1423/1423), suíte M1 (11/11), suíte auth/Google Play/elétrica/Android (30/30). `./gradlew :app:assembleRelease` falhou com `BOM_RELEASE_SIGNING_INCOMPLETE` — **isso é o esperado e correto** (não existe keystore de release configurado ainda; é o gate negativo funcionando).
- Revisão manual de `docs/operations/AI_CAPACITY_AND_FINANCIAL_SAFETY_V2.md`: nenhuma menção a áudio pago para remover.

## Commits gerados

**App** (`sim109-economic-final-app`):
1. `9eb1e63` — Item 1 + Item 2 (PackageAuthority + intenção de next-package). Foram no mesmo commit porque o Item 2 depende diretamente da classe do Item 1, nos mesmos trechos dos mesmos arquivos — não dava para separar sem quebrar a compilação no meio.
2. `cb14f0b` — Item 4, metade app.
3. `3b04efb` — formatação (dart format) dos arquivos tocados nesta sessão.
4. `fd0687d` — relatório de fechamento do Cut C.2, no padrão dos relatórios anteriores (`docs/sim-maximum-economy-project/execution-reports/`).

**Servidor** (`sim109-economic-final-server`):
1. `cfe5792` — Item 3 (prova do cache visual).
2. `a564000` — Item 4, metade servidor.
3. `892004f` — Item 5 (taxonomia de falha de provedor).

Nenhum dos dois branches foi enviado ao remoto (`origin`) — os commits estão só localmente nos worktrees, como de costume neste projeto.

## O que fica faltando (fora do escopo deste Cut)

O Cut C está **fechado**. O que vem depois, na ordem já registrada:
1. Configurar Google Play Billing de verdade — falta service account/JWT no servidor e keystore de release assinada no app; isso não é um bug de código, é configuração/infra que precisa de decisão e credenciais suas.
2. Gerar APK de teste.
3. Gerar AAB de teste interno.
4. Publicar AAB de produção.
5. Uma segunda fase de limpeza/padronização de código (inclui os 5 arquivos com formatação antiga mencionados acima).
