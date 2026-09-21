# Bug: rename de aula não persiste (task #125) — 2026-09-21

## Reprodução

Contra produção real (`https://simaitutor.com`), conta de QA `qa-amparo-20260921@sim-internal-test.invalid`, APP SHA `f85bcfa`, SERVER SHA `5f7e0cf`.

1. Menu (☰) → aula listada → "⋮" → "Rename lesson".
2. Editar o nome, tocar "Save".

**Resultado, reproduzido 2x em 2 aulas diferentes** ("Divisao de fracoes para o 6 ano" e "Fracoes equivalentes para o 6 ano"): o nome nunca muda (reabrir o diálogo mostra sempre o nome original). Um banner "Server unavailable. Try again. Try again" aparece no topo do menu.

## Causa raiz — parcialmente diagnosticada por leitura de código

O banner é **enganoso**, não vem do rename:
- Renderizado por `session.drawerLessonListError != null` em `lib/shared/widgets/shared_widgets.dart:~289`. Texto: `'${t('aula_server_unavailable')} ${t('retry')}'` — explica a duplicação "Try again. Try again" (as duas traduções concatenadas).
- `lessonListError` é setado como `'remote_lessons_unavailable'` em `lib/features/session/lab_session_drawer_controller.dart:258`, dentro de `listCloudLessons()`, quando a chamada a `/api/student-state/summaries` falha — isso é o refresh de LISTA, não o rename.
- Não vi essa chamada nos logs do droplet nos momentos exatos das minhas tentativas (só vi `/api/student-state/get` bem-sucedido para o lessonLocalId correto de cada aula) — sugere exceção client-side antes da requisição sair, ou falha de rede intermitente não capturada no log do servidor.

O rename em si falha **silenciosamente**:
- `onRename` em `shared_widgets.dart:421-427` descarta o retorno de `session.renameDrawerCloudLesson(...)` sem nenhum feedback — diferente de `onOpen`, que mostra SnackBar em caso de falha.
- **Hipótese descartada**: mismatch de `expectedRevision`. Verifiquei por leitura de código que tanto o drawer controller (`_readExistingLocalState`) quanto o coordinator (via `_statePort.readLocalLesson`, que chama o MESMO `_readExistingLocalState`) leem do mesmo canonical store — não há divergência de fonte entre os dois pontos onde a revisão é lida. Essa hipótese, registrada na Atualização anterior deste relatório, não se sustenta.
- **Candidato mais provável agora, achado por leitura mais funda do handler real** (`lib/sim/state/student_state_store.dart:2569`, `_applyRenameLessonCommand`): há duas guardas que rejeitam o comando **sem lançar exceção e sem nenhum log visível fora de debug build**, retornando só `applied: false` com uma string de razão:
  1. `rename_expected_objective_mismatch` (linha ~2596): se `command.expectedObjective` (capturado em `renameLesson`, coordinator, como `state.profile.objetivo` no momento da hidratação/leitura) não bate com `before.profile.objetivo` (o valor real no estado no momento em que o comando é efetivamente aplicado, que pode ter sido escrito por uma sessão/fork anterior com um `objetivo` diferente do que a hidratação recém-feita devolveu).
  2. `rename_duplicate` (linha ~2603): se `objetivo`, `targetTopic` E `sessionGoal` já são TODOS iguais ao novo nome — bem menos provável de ter disparado nos meus testes (troquei o nome, não deixei igual).
  A guarda (1) é a suspeita principal: essas 4 aulas foram tocadas por múltiplos forks/sessões de teste diferentes ao longo do dia (algumas passaram por `complete-lesson`, geração de visual, etc.), então é plausível que o `profile.objetivo` real no estado canônico local diverge do que a hidratação capturou como "esperado" nesse instante — um efeito de múltiplas escritas concorrentes de sessões de teste diferentes na mesma conta, não necessariamente um bug que afetaria uma conta de aluno real usada normalmente (um único usuário, um único device, sem essa concorrência entre sessões de teste).
- **Não consegui confirmar qual das duas guardas (ou outra causa) disparou de fato**: build release não expõe o `debugPrint` de `emitLessonWorkflowEvent` (`lab_session_drawer_controller.dart:615`, que logaria exatamente `decision=... reason=...`) via `adb logcat` — confirmei isso tentando capturar o log real durante a reprodução, resultado vazio. Precisa de `flutter run` attached (que os 3 outros bugs desta sessão usaram com sucesso) para ver esse log e confirmar a razão exata antes de aplicar qualquer fix.

## Não corrigido nesta rodada

Não apliquei fix especulativo sem confirmação ao vivo — risco de corrigir a guarda errada. Próxima ação exata: `flutter run -d 100.124.23.2:5555 --dart-define=FLUTTER_APP_MODE=production --dart-define=SIM_SERVER_URL=https://simaitutor.com`, reproduzir o rename, e ler a linha `[SIM] LESSON_WORKFLOW_OPEN ... decision=renameLesson reason=...` no console. Se for `rename_expected_objective_mismatch`: considerar se essa guarda faz sentido receber o `expectedObjective` capturado ANTES da hidratação vs. o valor real pós-hidratação (pode ser um "expected" desatualizado por construção, não um problema real de concorrência) — nesse caso a correção provável é não exigir esse match quando a lição acabou de ser hidratada remotamente (ela é, por definição, a fonte mais atual). Depois: corrigir, adicionar teste de regressão, e adicionar feedback de erro visível ao rename (hoje inexistente).

## Também descoberto nesta investigação (não é bug)

A conta de QA já acumulou 4 aulas de sessões anteriores, incluindo uma de currículo grande (60 itens, "Fracoes para o 6 ano do ensino fundamental", item 2/60) — útil e pronta para a validação de CG-1 (task #120) na próxima rodada, sem precisar gerar uma nova.

Handoff master atualizado com este achado: `2026-09-21-HANDOFF-CHECKLIST-FISICO-PARA-CODEX.md`.
