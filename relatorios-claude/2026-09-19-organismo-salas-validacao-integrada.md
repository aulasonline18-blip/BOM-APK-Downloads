# Organismo integrado das 5 salas do SIM — auditoria e correção

Data: 2026-09-19
Branch: `feature/nplus1-image-and-canonical-scroll`
Commit desta entrega: `77317d2024f72bfb53bc5a301598e0510507f6e6` (remoto confirmado idêntico)
Depende de / continua: `46d3611` (correção do loop de conclusão, relatório
`2026-09-19-finalizacao-salas-auxiliares-validacao-final.md`)

## Método

Auditoria paralela das 5 salas (Nivelamento, Dúvida, Revisão, Recuperação, Amparo) via leitura
direta de código, cada uma avaliada nas mesmas 8-9 dimensões do adendo (entrada/saída/retorno,
texto/imagem/áudio, avanço interno, matriz stale, restart, billing, sync/exclusão mútua,
identidade). Achados classificados CODE-VERIFIED-OK / GAP / UNCLEAR com evidência `arquivo:linha`.
Gaps confirmados foram corrigidos nesta mesma entrega e cobertos por teste automatizado novo.

## Matriz de capacidade pedagógica por sala (ratificada, não alterada nesta missão)

| Sala | Pode avaliar | Pode limpar pending | Pode alterar mastery | Pode avançar a aula |
|---|---|---|---|---|
| Nivelamento | Sim (score inicial) | N/A (roda antes da aula) | Define ponto inicial | N/A |
| Dúvida | Não | Não | Não | Não |
| Revisão | Sim (evidência própria) | Sim, por evidência suficiente | Não (mastery forte) | Não |
| Recuperação | Sim (evidência própria) | Sim, por evidência suficiente | Não (mastery forte) | Não |
| Amparo | Não (acolhimento) | Não | Não | Não |

Confirmado por código nesta auditoria (não apenas assumido): `AmparoRoomService.answerAmparoRoom`
nunca despacha `ConfirmAnswerLearningCommand`/mastery; `AmparoRoomService.finishAmparoRoom` só
toca `aux['amparo']`, nunca `aux['pendingMap']`.

## SIMWEB REFERENCE

Não foi necessário portar comportamento do SimWeb nesta auditoria — os dois defeitos confirmados
(discard de metadado visual, perda de pendência entre partes do CG-1) são bugs de arquitetura do
BOM, não lacunas comportamentais divergentes do SimWeb.

## NIVELAMENTO (Placement)

- Entrada/saída/retorno: CODE-VERIFIED-OK. `PlacementRouteController.skip()` (começar do zero,
  sem diagnóstico) e `chooseFindMyPoint()`→`startTest()` (achar meu ponto) são os dois únicos
  caminhos; `continueToAula()` é o único retorno.
- Texto/imagem/áudio: sem imagem/áudio por design — é um gate rápido de diagnóstico, não uma
  experiência completa. Não usa o pipeline visual nem `AudioCore`. Aceito como está.
- Avanço interno: local, sem controlador compartilhado (linear, autocontido).
- **GAP CONFIRMADO E CORRIGIDO**: `startTest()` não verificava se a tela havia sido fechada
  (`dispose()`) enquanto o loop de T02 do diagnóstico estava em voo, podendo escrever um
  resultado obsoleto. Corrigido com checagem de `_disposed` após o `await` e antes de qualquer
  escrita de estado (`lib/sim/placement/placement_route_controller.dart`). Teste novo:
  "startTest descarta resultado de T02 se a tela ja foi fechada (dispose) enquanto a chamada
  estava em voo" — AUTOMATED-VERIFIED.
- Restart: CODE-VERIFIED-OK nos 4 pontos exigidos (antes de responder, no meio, após pontuação,
  antes de entrar na aula) — já coberto por testes pré-existentes.
- Billing: gratuito (gate obrigatório de entrada, não sala auxiliar paga) — CODE-VERIFIED-OK.
- Exclusão mútua: estruturalmente impossível de coexistir com Recovery/Review/Amparo (Placement
  só roda antes da aula existir).

## DÚVIDA

- Entrada/saída/retorno: CODE-VERIFIED-OK. Bloqueada estruturalmente por `chat_aula_screen.dart`
  quando Recovery/Amparo/Review estão ativos (checagem de prioridade de tela). Nunca escreve
  `progress`/`current`.
- Texto/imagem/áudio: **NÃO** compartilha o gap das outras 3 salas — usa seu próprio
  `DoubtT02Caller` com `normalizeDoubtVisualTrigger`, preservando `visualTrigger` corretamente, e
  renderiza via `renderDoubtSoftwareVisual` (mesmo `lesson_visual_pipeline.dart` canônico, um
  segundo mecanismo síncrono legítimo, não uma rota paralela indevida). CODE-VERIFIED-OK.
- Matriz stale: CODE-VERIFIED-OK e forte — `DoubtRequestScope` (lessonLocalId+marker+itemIdx+
  layer) + assinatura de conteúdo validados antes de aplicar qualquer resposta; descarte explícito
  via evento `DOUBT_ANSWER_STALE_IGNORED` em vez de aplicar resultado obsoleto.
- Restart: UNCLEAR — estado de dúvida é presumivelmente efêmero por design (apresentação, não
  domínio durável), consistente com a separação exigida pelo adendo, mas não confirmado lendo
  código de persistência dedicado.
- Billing: não verificado nesta auditoria (cobrança presumivelmente no servidor via
  `mode: doubt`), já tratado em missão anterior.

## REVISÃO

- Entrada/saída/retorno: CODE-VERIFIED-OK. `startReviewRoom` recusa abrir se `recoveryRoom != null`
  (`lab_session_review_controller.dart:63`). Contexto reconstruído a cada chamada a partir do
  cursor real da aula principal — nunca cacheia posição obsoleta.
- **GAP CONFIRMADO E CORRIGIDO (compartilhado com Recuperação/Amparo)**: `AuxRoomContent.fromLesson`
  descartava `imageDataUrl`/`imageId`/`imageStatus`/`mimeType` do `T02LessonMaterial` — exatamente
  a suspeita do adendo. Ver seção "CORREÇÃO 1" abaixo.
- Seleção: CODE-VERIFIED-OK, sem aleatoriedade. `buildReviewQueue` prioriza pendências reais por
  antiguidade (FIFO); só cai para rotação sequencial (`sequentialCursor % length`) quando não há
  pendência.
- Lei de evidência: CODE-VERIFIED-OK — `answerReviewRoom` usa a MESMA `recordAuxRoomAnswer` que a
  Recuperação usa; mostrar a pergunta não limpa pendência, só resposta correta+sinal 1.
- **TESTE DE INTEGRAÇÃO NOVO**: "pendencia resolvida na Revisao (evidencia suficiente) nao e
  re-apresentada pela Recuperacao no fim da aula" (`test/c6_review_engine_test.dart`) — prova que
  resolver M1 na Revisão remove M1 de `buildRecoveryQueueForLesson` e libera
  `shouldBlockFinalCompletionForRecovery`. AUTOMATED-VERIFIED.
- Stale-result: CODE-VERIFIED-OK — contador monotônico por requisição (`_reviewRequestId`)
  checado antes de aplicar qualquer resultado assíncrono.
- Restart: CODE-VERIFIED-OK por design — só a fila/índice em `auxRooms['review']` é durável; a
  projeção de UI não é persistida (correto para sala opcional não-bloqueante).
- Mutua exclusão: CODE-VERIFIED-OK (recusa abrir sobre Recovery/Amparo).

## RECUPERAÇÃO

- Entrada/saída/retorno: CODE-VERIFIED-OK. Uma única `RecoveryRoomScreen` +
  `SimPreparationExperience(stage: recovery/recoveryDone)`. Sequência do robô confirmada em
  código: INTRO → QUESTIONS → RECOVERY DONE → (revalidação de pendingMap) → FINAL DONE → HOME.
- **GAP CONFIRMADO E CORRIGIDO (compartilhado)**: mesmo defeito de `AuxRoomContent.fromLesson`
  descrito acima.
- **GAP CRÍTICO CONFIRMADO E CORRIGIDO — sobrevivência de pendência entre partes do CG-1**: uma
  pendência de recuperação aberta na parte N era **perdida silenciosamente** ao cruzar para a
  parte N+1, porque cada parte do CG-1 é um `lessonLocalId` novo, criado do zero pelo pipeline de
  onboarding de continuação, sem qualquer cópia de `auxRooms`/`pendingMap` da parte anterior. Ver
  "CORREÇÃO 2" abaixo — este era o achado mais importante desta auditoria.
- Finish revalida pendingMap: CODE-VERIFIED-OK e já correto —
  `RecoveryRoomService.finishRecoveryRoom` reavalia `shouldLessonBlockFinalCompletion` antes de
  concluir; se restar pendência, `restartRequired: true` e a Recuperação continua (não confia em
  índice de fila).
- Falha de rede: CODE-VERIFIED-OK — já coberto por teste pré-existente (preserva aula principal,
  não reintroduz rota legada).
- Restart nos 4 pontos exatos (intro/meio-questão/pós-resposta/recoveryDone): UNCLEAR — estado é
  durável (`state.auxRooms['recovery']`) e há caminho de reidratação
  (`restoreRecoveryProjection`/`openRecoveryInstant`), mas não foi testado explicitamente em cada
  um dos 4 pontos nesta entrega.
- Billing: UNCLEAR nesta auditoria (fora do escopo do app-repo).

## AMPARO

- Entrada/saída/retorno: CODE-VERIFIED-OK. Guards simétricos contra Review/Recovery ativos em
  todos os 4 pontos de abertura (`lab_session_amparo_flows.dart`). Nunca escreve
  `state.current`/`state.progress` — cursor preservado por construção.
- **GAP CONFIRMADO E CORRIGIDO (compartilhado)**: mesmo defeito de `AuxRoomContent.fromLesson`.
- **Garantia "nunca apaga pending por participação"**: CODE-VERIFIED-OK, confirmado com o maior
  rigor desta auditoria — `finishAmparoRoom` só toca `aux['amparo']`, nunca lê nem escreve
  `aux['pendingMap']`. **TESTE DE INTEGRAÇÃO NOVO**: "ciclo completo de amparo (3 estacoes) nao
  apaga pendencia de recuperacao real" (`test/amparo_room_contract_test.dart`) — completa as 3
  estações reais, conclui o amparo, e prova que a pendência de M1 continua `status: pending` e
  `shouldBlockFinalCompletionForRecovery` continua `true`. AUTOMATED-VERIFIED.
- Nunca avança/nunca marca mastery: CODE-VERIFIED-OK — `answerAmparoRoom` não despacha nenhum
  comando canônico de resposta oficial.
- Identidade de estação: CODE-VERIFIED-OK — `stationIdentity = operationId:marker:amparoType`,
  única por ciclo/estação/tipo.
- Stale/restart: UNCLEAR — não totalmente traçado nesta entrega (token de requisição de amparo
  não auditado ponta-a-ponta).
- Billing: UNCLEAR — nenhuma referência a crédito/cobrança encontrada nos arquivos de Amparo;
  precisa de trace dedicado (fora do escopo desta entrega).

## CORREÇÃO 1 — metadado visual descartado (Revisão + Recuperação + Amparo)

`T02LessonMaterial` (resposta real do T02) sempre carregou `imageDataUrl`/`imageId`/
`imageStatus`/`mimeType`, mas `AuxRoomT02Caller.call()` construía `LessonContent` (e depois
`AuxRoomContent.fromLesson`) usando só os campos textuais — descartando a imagem
incondicionalmente, para as 3 salas que passam pelo funil compartilhado
`student_aux_room_service.dart`. Corrigido:

- `AuxRoomContent` ganhou `imageDataUrl`/`imageId`/`imageStatus`/`mimeType` (ausência continua
  sendo `no_image` legítimo, nunca geração forçada).
- `AuxRoomCallResult` carrega esses campos desde `T02LessonMaterial`.
- Os dois pontos de reconstrução em `student_aux_room_service.dart` passam os campos adiante.
- `_AuxQuestionScaffold` (`aux_room_screens.dart`) renderiza a imagem via o widget canônico já
  usado pela aula principal (`LessonMediaImageView`) — nenhuma rota visual paralela criada.
- Teste novo: "imagem do T02 sobrevive ate o conteudo da sala" — prova presença E ausência
  legítima. AUTOMATED-VERIFIED.

## CORREÇÃO 2 — pendência de recuperação perdida na fronteira de parte do CG-1

Rastreamento confirmou: `activateReadyNextPart` (`lab_session_curriculum_boundary_controller.dart`)
trocava `lessonLocalId` para a próxima parte sem qualquer carregamento de `pendingMap` da parte
que terminou. Corrigido com `carryForwardUnresolvedRecoveryPendingAcrossPart` (nova função pura em
`lib/sim/experience/curriculum_utils.dart`, reutilizando `ApplyAuxRoomsStateCommand` — nenhuma
autoridade paralela nova), chamada no exato momento da ativação da próxima parte — depois que a
parte atual está genuinamente terminada, garantindo que o `pendingMap` de origem é o final e
completo. Idempotente (reentrada não duplica a pendência carregada).

Como a pendência carregada referencia um `marker` que só existe no currículo da parte de origem,
`LabSessionRecoveryController._context()` também foi estendido para incluir itens de qualquer
`lessonLocalId` referenciado pelas pendências carregadas — sem isso, a Recuperação teria a
pendência registrada mas seria incapaz de montar a pergunta (marker não encontrado no currículo
ativo).

Teste novo em `test/c2_curriculum_parts_test.dart`: prova que a pendência de M5 registrada na
parte 1 aparece integralmente (marker, status, lessonLocalId de origem) no `auxRooms.pendingMap`
da parte 2 após a chamada, com evento `RECOVERY_PENDING_CARRIED_FORWARD`, e que reentrada é
idempotente. AUTOMATED-VERIFIED.

## POLÍTICA COMPARTILHADA DE MÍDIA

Antes desta correção não havia política alguma de imagem para Revisão/Recuperação/Amparo (campo
inexistente no modelo). Agora as 3 compartilham exatamente os mesmos campos e o mesmo widget de
renderização da aula principal — uma política, não três implementações divergentes. Áudio: não
foi encontrada autoridade de áudio dedicada e duplicada nas salas auxiliares; presumem-se
consumidoras do `AudioCore` genérico da tela — não confirmado exaustivamente linha a linha nesta
entrega.

## MATRIZ CRUZADA (pontos exigidos pelo adendo)

| Cenário | Resultado |
|---|---|
| 3 pendências, recalcula a cada resposta | AUTOMATED-VERIFIED (`recovery_room_contract_test.dart`) |
| Review resolve pending antes do fim → Recovery não a repete | AUTOMATED-VERIFIED (`c6_review_engine_test.dart`) |
| Amparo participado não apaga pending da Recuperação | AUTOMATED-VERIFIED (`amparo_room_contract_test.dart`) |
| Pendência sobrevive à fronteira de parte do CG-1 | AUTOMATED-VERIFIED (`c2_curriculum_parts_test.dart`) |
| Consolidar idempotente sob toque duplicado | AUTOMATED-VERIFIED (relatório anterior, `46d3611`) |
| Restart da Recuperação nos 4 pontos exatos | **EXTERNAL-REQUIRED** (não testado nesta entrega) |
| Troca de conta/aula em voo durante Consolidar | **EXTERNAL-REQUIRED** (não testado nesta entrega) |
| Billing do Amparo (existe? é idempotente?) | **EXTERNAL-REQUIRED** (não rastreado nesta entrega) |

## DEAD CODE / DUPLICAÇÃO

Nenhuma duplicação de autoridade encontrada além do já corrigido. `student_aux_room_service.dart`
emite os mesmos tipos de evento de conclusão (`FINAL_COMPLETION_ALLOWED` etc.) em pontos de vida
da própria Recuperação — confirmado que ambos delegam à mesma checagem canônica, não são
autoridades paralelas (já documentado no relatório anterior). Nenhum símbolo `V2` ou sistema
paralelo introduzido ou encontrado.

## TESTS

Suíte completa: 1449 testes, todos passando (`flutter test`). `flutter analyze`: zero problemas
em todo o projeto. `dart format`: aplicado só aos arquivos desta entrega.

## PHYSICAL

Não realizado nesta entrega — ver checkpoint de durabilidade e validação física no relatório de
acompanhamento (`Org-10`/`Org-11`, ainda pendentes no momento da escrita deste documento).

## REMAINING EXTERNAL VALIDATION

1. Restart da Recuperação testado explicitamente nos 4 pontos (intro/meio-questão/pós-resposta/
   recoveryDone).
2. Teste de troca de conta/aula em voo durante o Consolidar (stale write).
3. Trace completo de billing do Amparo (existe cobrança? é idempotente sob retry?).
4. Auditoria de áudio linha-a-linha confirmando ausência de autoridade duplicada nas 5 salas.
5. Validação física completa no tablet (todas as 5 salas + fechamento + CG-1 grande).
6. Segundo checkpoint de durabilidade do GitHub imediatamente antes do APK físico (terceiro
   adendo) e geração do APK integrado.
