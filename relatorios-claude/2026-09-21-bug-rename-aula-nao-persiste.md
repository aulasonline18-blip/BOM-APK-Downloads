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
- Hipótese mais provável (não confirmada ao vivo): em `renameCloudLesson` (`lab_session_drawer_controller.dart:354`), `expectedRevision = local?.stateRevision` — se a aula não está em cache local (comum para aulas antigas só listadas via summaries remotas), esse valor pode não bater com a revisão real que o coordinator hidrata do servidor dentro de `renameLesson` (`lesson_workflow_coordinator.dart:863`, via `_readOrHydrateLifecycleLesson`), causando rejeição por `dispatchWorkflowCommand` (linha ~892) sem que ninguém veja o motivo.

## Não corrigido nesta rodada

Precisa de `flutter run` attached para confirmar com `debugPrint` o `expectedRevision` enviado vs. a revisão real, e o `result.applied`/`result.reason` retornado. Depois: corrigir a causa raiz confirmada e adicionar feedback de erro visível ao rename (hoje inexistente).

## Também descoberto nesta investigação (não é bug)

A conta de QA já acumulou 4 aulas de sessões anteriores, incluindo uma de currículo grande (60 itens, "Fracoes para o 6 ano do ensino fundamental", item 2/60) — útil e pronta para a validação de CG-1 (task #120) na próxima rodada, sem precisar gerar uma nova.

Handoff master atualizado com este achado: `2026-09-21-HANDOFF-CHECKLIST-FISICO-PARA-CODEX.md`.
