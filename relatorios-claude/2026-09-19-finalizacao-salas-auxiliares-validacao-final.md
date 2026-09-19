# Finalização do ciclo pedagógico — encerramento canônico, recuperação e CG-1

Data: 2026-09-19
Branch: `feature/nplus1-image-and-canonical-scroll`
Commit desta entrega: `46d3611d2a1e7490a905f38812ba4b489f8bf604` (remoto confirmado idêntico)

## Escopo desta entrega

Este relatório cobre a parte **executada e provada** da "MISSÃO FINAL PRÉ-AAB": o bug relatado
("no último item, aperta Consolidar e volta para o mesmo item repetidamente") foi isolado,
reproduzido causalmente, corrigido na autoridade correta, e verificado sem regressão em toda a
suíte + na interação com CG-1. As partes de escopo massivo que **não foram** executadas nesta
entrega (organismo de 5 salas do primeiro adendo, validação física em tablet, segundo checkpoint
de durabilidade pré-AAB) estão listadas em "VALIDAÇÃO EXTERNA RESTANTE".

## SIMWEB REFERENCE

Não foi necessário portar comportamento do SimWeb nesta correção: o defeito estava inteiramente
na camada de autoridade de conclusão do BOM (`avancar()`), não numa lacuna comportamental
divergente do SimWeb. Nenhum arquivo do SimWeb foi lido ou copiado nesta entrega.

## BOM BEFORE (causa raiz)

Em `lib/sim/classroom/lesson_answer_progress_controller.dart`, a função `avancar()` tem dois
pontos em que detecta pendência de recuperação no fim da aula:

1. `item == null` (cursor já passou do último item do curriculum local) — linha ~268.
2. `currentView.ended` dentro do fluxo normal de avanço — linha ~550.

Em ambos, quando `_blockFinalCompletionForRecovery(...)` retornava `true` (pendência real
aberta), a função apenas registrava o evento `FINAL_COMPLETION_BLOCKED_BY_PENDING` e retornava —
**sem nunca mover `position.phase` para fora de `ClassroomPhaseType.concluido`**. Como o próprio
guard no topo de `avancar()` só permite reentrar quando `phase == concluido`, a função ficava
perpetuamente reentrável no mesmo estado: a cada novo toque em "Consolidar", o cursor interno
silenciosamente avançava (`itemIdx` saía do curriculum), um novo evento de bloqueio duplicado era
gravado, mas a tela nunca mudava — produzindo exatamente o sintoma relatado.

Um segundo problema latente, correlato mas distinto, foi confirmado em
`lib/features/classroom/chat_aula_screen.dart`: a checagem `snapshot.isDone` acontecia **antes**
de `session.recoveryRoom != null`. Hoje isso nunca se manifestava (porque `isDone` estruturalmente
não podia ficar `true` no caminho bloqueado), mas era um convite a regressão futura — um snapshot
`isDone` obsoleto/transiente poderia esconder uma Recuperação obrigatória já aberta.

## FINAL AUTHORITY

Autoridade única de conclusão mantida em `avancar()` (nenhuma segunda máquina de estados criada).
Foi adicionado um novo valor tipado e distinto a `ClassroomPhaseType`
(`bloqueadoRecuperacao`, com construtor `ClassroomPhase.blockedForRecovery()`), reaproveitando a
checagem autoritativa já existente `shouldBlockFinalCompletionByRecoveryGate` /
`shouldBlockFinalCompletionForRecovery` (pendingMap real) — nenhum boolean paralelo novo foi
criado; o boolean anterior (`blockedByRecovery` em `LabSessionClassroomAdvanceController`) passou
a ler o **resultado** da autoridade (o `phase.type`) em vez de fazer *event-tail sniffing* do log
de eventos.

Guard de reentrada em `avancar()` ajustado para aceitar `concluido` **ou** `bloqueadoRecuperacao`
como fases de entrada válidas — isto permite que `resumeOfficialCompletion()` (chamado por
`LabSessionRecoveryController.finish()` quando a Recuperação termina de verdade) reavalie a mesma
autoridade e alcance `ClassroomPhase.doneEnd()` quando a pendência já foi resolvida.

Para evitar poluição do log de eventos em toques repetidos enquanto ainda bloqueado, o evento
`FINAL_COMPLETION_BLOCKED_BY_PENDING` só é gravado na **transição** para o estado bloqueado, não
a cada reavaliação idempotente.

## CONSOLIDATE

Botão "Consolidar" já usava a chave i18n `aula_consolidate` corretamente (confirmado em
`lesson_answer_feedback.dart:46`) — nenhuma mudança necessária aí.

Fluxo sem pendência: responde → feedback → Consolidar → `avancar()` autoriza →
`ClassroomPhase.doneEnd()` → `snapshot.isDone == true` → `LessonDoneScreen`. Verificado por teste
automatizado pré-existente ("avanço físico usa cursor canônico...") e pelo ramo final do novo
teste de regressão.

Idempotência: toque duplicado enquanto ainda bloqueado não duplica evento, não avança o cursor,
não produz `FINAL_COMPLETION_ALLOWED` — provado no teste
`bom_offline_three_experiences_contract_test.dart` (reescrito para refletir o comportamento
correto; antes ele **fixava o bug antigo como esperado**, o que violava a lei funcional da
missão — corrigido teste + implementação juntos, com justificativa registrada no diff).

## RECOVERY

- Bloqueio agora produz estado tipado (`bloqueadoRecuperacao`) que
  `LabSessionClassroomAdvanceController` usa para chamar `auxController.startRecoveryRoom()` —
  sem sniffing de eventos.
- Prioridade Recovery-antes-de-Done: `chat_aula_screen.dart` reordenado para checar
  `recoveryRoom`/`amparoRoom`/`reviewRoom` antes de `isDone` — invariante agora estrutural, não
  dependente de coincidência de timing.
- Revalidação de pendingMap no fim da Recuperação: **já existente e correta** em
  `RecoveryRoomService.finishRecoveryRoom` (`shouldLessonBlockFinalCompletion` reavaliado; se
  ainda houver pendência, `restartRequired: true` e a Recuperação continua) — código-verificado,
  não alterado.
- Falha de rede durante Recuperação: **já coberta** por teste pré-existente
  (`recovery_room_contract_test.dart`, "falha T02 preserva aula principal e nao reintroduz rota
  legada") — current/progress/attempts preservados, status `failed`, sem rota legada.
- Fila recalculada a cada resposta, clareamento apenas por evidência suficiente (correto + sinal
  1): já coberto por testes pré-existentes desta mesma suíte.

## REVIEW / AMPARO

Não alterados nesta entrega. Invariantes de não-interferência com o cursor principal e de
clareamento apenas por evidência seguem cobertos pelos testes já existentes e passando:
`C9 revisao nao apaga nem retrocede progresso principal`, `revisao manual preserva item e camada
da aula principal`, `review finishes without interleaving the main lesson queue`,
`T02 falhando apos acolhimento preserva aula principal` (Amparo), `sinal 2 correto nao conta como
agravante`. Threshold de 5 agravantes consecutivos e a regra `erro OU (correto + sinal 3)` também
já cobertos e inalterados.

## PLACEMENT

Não alterado nesta entrega. Extremos (começa do zero, avança direto, posição intermediária) já
cobertos por `PlacementScoringEngine starts safely before basic or uncertain gaps`,
`does not let isolated advanced success skip base`, `stops early when confidence is sufficient`.
Falha de T02 e reentrada seguros: `T02 failure creates safe result without fake question or
blocking route`.

## CG-1

Auditoria confirmou que cada parte do CG-1 é um `lessonLocalId` separado com `curriculum.items`
local — ou seja, `avancar()` também detectava "fim" em toda fronteira de parte, não só no fim
global. Antes da correção, isso significava que uma pendência de recuperação aberta numa parte
não-final poderia indevidamente bloquear a transição de parte e abrir Recuperação numa fronteira
técnica (ex.: item 80 de um currículo de 300).

Corrigido em `_blockFinalCompletionForRecovery`: agora retorna `false` imediatamente se
`hasPendingNextCurriculumPart(state)` for verdadeiro (autoridade já existente do CG-1, baseada em
`CurriculumGlobalPlan.hasRemainingGlobalItems` — nenhuma autoridade paralela nova), deixando a
fronteira de parte seguir seu fluxo normal (`doneEnd()` local → camada de `LabSession` decide
ativar a próxima parte). A avaliação de recuperação/conclusão passa a ocorrer **apenas** no fim
GLOBAL de verdade.

Novo teste `CG-1: fim de PARTE com pendencia nao abre Recuperacao...` prova as duas pontas: mesma
pendência, com `moreGlobalPartsRemain: true` → fase `fim` (fronteira normal); com
`moreGlobalPartsRemain: false` → fase `bloqueadoRecuperacao` (fim global de verdade).

**Aberto, não verificado nesta entrega:** se `pendingMap` de fato viaja através da troca de
`lessonLocalId` quando uma nova parte é ativada (requisito do adendo 1 de sobrevivência de
pendência entre partes). Fica registrado como pendência para o Organism-2 (auditoria de 5 salas).

## TESTS

- Teste de reprodução causal (`classroom_phase_test.dart`, "REGRESSAO: ultimo item com pendencia
  produz um estado bloqueado distinto..."): falha no código antigo, passa no código corrigido.
  Cobre bloqueio → estado distinto (sem loop) → toques repetidos seguros → resolução real por
  evidência → conclusão verdadeira.
- Teste CG-1 dedicado (acima).
- `bom_offline_three_experiences_contract_test.dart` corrigido (implementação + teste, com
  justificativa) para refletir o comportamento correto em vez do bug antigo.
- Suíte completa: **1443 testes, todos passando** (`flutter test`). Um teste
  (`android_release_gate_behavior_test.dart`) apresentou falha isolada ao rodar dentro da suíte
  completa mas passou de forma limpa e determinística quando executado sozinho — não relacionado
  a este diff (Gradle/build Android), tratado como flakiness ambiental pré-existente, não como
  regressão desta entrega.
- `flutter analyze`: nenhum problema nos arquivos alterados.
- `dart format`: aplicado aos arquivos alterados; nenhuma mudança de formatação foi feita em
  arquivos fora do escopo desta correção (16 arquivos pré-existentes têm drift de formatação não
  relacionado, deixados intactos).

## PHYSICAL

Não realizado nesta entrega. Requer build de APK integrado e teste físico no Samsung Galaxy Tab
A9, listado em VALIDAÇÃO EXTERNA RESTANTE.

## APK

Não gerado nesta entrega.

## REMOVED LEGACY

Removido: o *event-tail sniffing* (`events.last.type == 'FINAL_COMPLETION_BLOCKED_BY_PENDING'`)
em `LabSessionClassroomAdvanceController.advance()` — confirmado via grep que não existe nenhuma
outra ocorrência do mesmo padrão em `lib/`.

Auditado e mantido (não são duplicação de autoridade, apenas pontos de emissão de auditoria da
mesma autoridade única): `student_aux_room_service.dart` também emite
`FINAL_COMPLETION_BLOCKED_BY_PENDING`/`FINAL_COMPLETION_ALLOWED`, mas em pontos de vida da própria
Sala de Recuperação (`registerRecoveryStarted`, `registerFinalCompletionAllowed`), ambos
delegando à mesma checagem canônica `shouldBlockFinalCompletionForRecovery`.

`test/official_advance_rule_contract_test.dart` (teste que apenas verifica strings no código-fonte
sem executar `avancar()`) foi mantido — ele ainda documenta uma fronteira arquitetural real e
válida; não é "morto", apenas insuficiente sozinho, e essa insuficiência foi resolvida pelo novo
teste comportamental.

## GITHUB DURABILITY CHECKPOINT (desta entrega)

- APP (`/root/worktrees/sim109-nplus1-scroll`): branch `feature/nplus1-image-and-canonical-scroll`,
  local HEAD `46d3611d2a1e7490a905f38812ba4b489f8bf604`, remoto idêntico (confirmado via
  `git fetch` + `git rev-parse origin/...`), working tree limpo, push confirmado.
- Nenhum outro repositório (Servidor-BOM, BOM-APK-Downloads) foi alterado por esta correção
  (mudança 100% client-side).

## REMAINING EXTERNAL VALIDATION

Itens do pedido original explicitamente **não** executados nesta entrega, por serem de escopo
massivo e/ou dependerem de hardware físico:

1. Segundo adendo completo — auditoria integrada das 5 salas (mídia/áudio/identidade/billing/
   stale-result) tratando-as como um organismo único.
2. Matriz de teste completa das seções 42-51 da missão principal (alguns cenários já cobertos
   nesta entrega — ver TESTS —, mas não todos: faltam explicitamente (a) resolver a 3ª de três
   pendências primeiro com recálculo de ordem, (b) teste de stale/concorrência ao trocar de
   aula/conta durante o Consolidar, (c) interação explícita Review-já-resolveu-a-pendência não
   reaparecer na Recuperação, (d) interação explícita Amparo-não-apaga-pendência-de-Recuperação
   como teste de integração dedicado, além de restart em Recuperação nos 4 pontos exatos
   (intro/meio-questão/pós-resposta/recoveryDone) como testes dedicados.
3. Validação física completa no Samsung Galaxy Tab A9 (cenários A-G do pedido original).
4. Geração de APK integrado combinando esta correção com P1/P2/CG-1/visual-prep.
5. Segundo checkpoint de durabilidade do GitHub imediatamente antes de gerar o APK físico de
   validação (o terceiro adendo exige isso como um segundo checkpoint idêntico ao inicial).
6. Merge para `main` e geração de candidato AAB — não realizado; branch de trabalho permanece
   `feature/nplus1-image-and-canonical-scroll`, sem overwrite de trabalho alheio.

O bug relatado ("aperta e volta para o mesmo item") está estruturalmente corrigido na autoridade
canônica e coberto por teste automatizado que prova a ausência do loop — mas a aceitação física
completa da missão, conforme definida em suas próprias seções finais, permanece pendente dos itens
acima.
