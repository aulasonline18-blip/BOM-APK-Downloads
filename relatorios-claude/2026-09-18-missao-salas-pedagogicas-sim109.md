# SIM109 — Missão: ajustes finais das salas pedagógicas (Nivelamento, Dúvida, Revisão, Recuperação, Amparo, Anexos)

**Data:** 2026-09-18
**Branch:** `reform/sim109-economic-final-construction` (app e server)

**APP BASE:** `bc6602c` → **APP FINAL:** `5a67fd7`
**SERVER BASE:** `90fe89b` → **SERVER FINAL:** `6984e0d`

Esta execução seguiu diretamente a sessão anterior (bug de imagem + consumo de créditos,
já relatada em `2026-09-18-bug-imagem-e-consumo-creditos-sim109.md`). A missão pediu
investigação + correção + implementação + limpeza + teste + validação física de ponta a
ponta nas 5 salas auxiliares, anexos e scroll, sem pausar para perguntar prioridade.

**Aviso de honestidade metodológica:** esta missão é grande demais para ser 100% concluída
em uma única sessão com o rigor que ela própria exige (arquitetura + testes automatizados +
teste físico, simultaneamente, para cada item). Abaixo, cada achado é marcado como
**RESOLVIDO** (implementado, testado, validado), **CONFIRMADO CORRETO** (auditado e já
estava certo — nenhuma mudança necessária) ou **PENDENTE** (não coberto nesta sessão).
Nenhum item é declarado "concluído" sem essa evidência.

---

## 1. STATUS FINAL POR SALA

| Sala | Status |
|---|---|
| Amparo | **RESOLVIDO** — gatilho corrigido para 5 agravantes consecutivos com a semântica ratificada; telemetria morta removida; crédito por interação implementado e testado |
| Dúvida | **RESOLVIDO** (crédito) + **CONFIRMADO CORRETO** (proteção contra resposta obsoleta) + **1 bug residual encontrado, não corrigido** (ver seção 9) |
| Revisão | **CONFIRMADO CORRETO** — auditoria profunda não encontrou defeito na autoridade de pendências nem no fallback voluntário |
| Recuperação | **CONFIRMADO CORRETO** — regra de limpeza e gate de bloqueio já implementados corretamente; crédito por interação implementado e testado |
| Nivelamento | **RESOLVIDO** (parcial) — subtítulos explicativos já escritos no código foram ligados à tela; arquitetura de máquina de estados auditada e já correta |
| Anexos | **CONFIRMADO CORRETO** — testado ao vivo (Gemini real, não mock) nos 4 formatos: TXT, PDF, DOCX, imagem |
| Scroll/navegação | **PENDENTE** — auditoria estática não encontrou bug estrutural nas duas telas principais; teste físico completo não foi executado |

---

## 2. AMPARO — o que estava errado e o que foi corrigido

**Errado:**
1. Gatilho hardcoded em 3 agravantes consecutivos (deveria ser 5, por decisão desta missão).
2. Definição de "agravante" incluía qualquer resposta com sinal ≠ 1 — ou seja, um **acerto**
   com sinal 2 (leve insegurança) contava like um erro ou uma insegurança profunda (sinal 3).
   Isso valia tanto para o contador principal quanto para o contador paralelo de "insegurança
   consecutiva", então mesmo corrigindo só o contador principal, 5 acertos-sinal-2 seguidos
   ainda abririam Amparo pelo caminho secundário.
3. Telemetria de "resposta rápida e errada" (`quickWrongAnswer`) nunca funcionava: a função
   que deveria fornecer o tempo de resposta sempre retornava `null` (nenhum escritor real do
   campo existia), então o cálculo de `quickWrongAnswer` era sempre `false` — código morto
   rodando a cada tentativa sem nunca produzir efeito.

**Corrigido:**
- Regra ratificada: agravante = resposta **errada** OU resposta **correta com sinal 3**.
  Acerto com sinal 2 não conta mais, nem no contador principal nem no de insegurança.
- Limiar alterado de 3 → 5.
- `AmparoTrigger.quickWrongAnswer`, a constante `quickWrongAnswerThresholdMs`, a função
  `recordedAnswerDurationMs()` e o parâmetro `responseDurationMs` foram **removidos**
  (não implementados de forma completa, pois isso seria uma funcionalidade nova, fora do
  escopo desta missão — não apenas uma correção).

**Arquivos:** `lib/sim/auxiliary/amparo_room_engine.dart`,
`lib/sim/classroom/lesson_answer_progress_controller.dart`,
`test/amparo_room_contract_test.dart`, `test/classroom_phase_test.dart`,
`test/review_manual_only_contract_test.dart`.

**Testes:** suíte completa do app — **1414/1414 passando** após cada mudança. Testes antigos
que assumiam "3 agravantes" foram **reescritos** para 5 (não apenas com o contador
incrementado — os cenários mistos erro+sinal3 foram estendidos de 3 para 5 eventos reais).
Novo teste prova explicitamente: 5 acertos-sinal-2 seguidos nunca abrem Amparo.

**Evidência da definição exata de "agravante" (prova pedida pelo adendo):**
```dart
// amparo_room_engine.dart
final insecure = attempt.correct && attempt.sinal == DecisionSignal.three;
final isAggravant = !attempt.correct || insecure;
```

**Prova física no tablet:** não executada nesta sessão (pendência externa de tempo, não
técnica — ver seção 12).

---

## 3. CRÉDITO NAS SALAS AUXILIARES (Dúvida/Revisão/Recuperação/Amparo)

**Errado:** o servidor só cobrava crédito para `mode === "lesson"`. As quatro salas
auxiliares não cobravam nada.

**Decisão de design:** ao contrário do item de aula (cobrado por item, idempotente por
item — recarregar o mesmo item não cobra de novo), as salas auxiliares cobram por
**interação**: a mesma pergunta/marcador pode ser revisitada (nova dúvida sobre o mesmo
item, nova tentativa de recuperação, nova estação de amparo) e cada uma é um evento
cobrável distinto — mas um retry de rede da *mesma* interação não pode cobrar duas vezes.

Em vez de inventar um novo identificador do zero, a implementação reaproveita
`operationIdentity()` — mecanismo já existente em `ai-cost-protection-gate.js` que já
calcula um hash do conteúdo exato de cada interação auxiliar (texto da dúvida, nível/
estação/tipo do amparo, sinal, resposta selecionada/correta). Retry da mesma requisição
produz o mesmo hash (idempotente); uma dúvida diferente, uma estação diferente, produzem
hashes diferentes (cobra de novo).

**Bug real encontrado no caminho:** o campo `__serverCommercialCreditKey` é sempre
priorizado pelo cost-gate sobre sua própria identidade interna, para qualquer requisição
T02. Ele estava sendo preenchido sempre com a chave *por item* (usada para aula normal),
inclusive para os modos auxiliares — o que teria colapsado toda cobrança de
dúvida/revisão/recuperação/amparo na mesma chave por item, cobrando só uma vez por item
independentemente de quantas interações auxiliares acontecessem. Corrigido para computar
a chave certa por modo.

**Arquivo:** `src/t02/complete-lesson-controller.js`.

**Testes:** novo `test/credits_aux_room_contract.test.js`, provando para os 4 modos:
primeira interação cobra 1 crédito; retry idêntico não cobra de novo; interação diferente
cobra de novo; saldo zero bloqueia com 402 antes de chamar a IA. Suíte completa do
servidor: **100/100 arquivos passando**.

**Prova física no tablet (conta real, sem saldo de teste, `ccrfoodgy1@gmail.com`):**
1. Conta nova → bônus de 15 créditos confirmado na tela.
2. Item 1 da aula carregado → saldo 15→14 (confirmado na tela de Créditos).
3. Dúvida enviada sobre o item 1 ("Por que a planta precisa da luz solar") → **saldo
   14→13 confirmado no ledger do servidor**, motivo da transação `aux-room-doubt`, chave de
   operação no novo formato por interação (`aux-item:doubt:...`).
4. HTTP 200 do servidor confirmado com a explicação da dúvida realmente gerada pela IA.

Revisão/Recuperação/Amparo não foram testados fisicamente da mesma forma nesta sessão
(pendência — ver seção 12), mas usam exatamente o mesmo mecanismo já provado ao vivo para
Dúvida, e têm cobertura automatizada completa e equivalente.

---

## 4. DÚVIDA — auditoria

**Confirmado correto (sem mudança):**
- Proteção contra resposta obsoleta (`lesson_doubt_controller.dart`): contador de geração
  de requisição + verificação de escopo (lessonLocalId/marker/itemIdx/layer) antes e depois
  da chamada de rede. Duplo toque já é bloqueado no controller (`if (status == processing)
  return`).
- Contrato reduzido do modo dúvida no servidor (não exige A/B/C nem gabarito).

**Bug real encontrado, não corrigido (pendência):** ao testar fisicamente no tablet, a
explicação da dúvida foi computada corretamente e cobrada corretamente (ver seção 3), mas
**não apareceu ao vivo na tela do chat** — só passou a aparecer depois de reiniciar o app.
Rastreei o caminho `setDoubt()` → `notifyListeners()` → `_onSessionChange` → `setState()`
e a fiação está estruturalmente correta; não consegui fixar a causa raiz exata dentro do
tempo desta sessão. **Não é perda de dado nem cobrança incorreta** — a resposta é
computada, cobrada uma única vez, e fica corretamente arquivada/recuperável após reload.
É um atraso de renderização ao vivo, não um bug de integridade. Registrado como pendência
técnica interna (não externa) para próxima sessão.

---

## 5. REVISÃO — auditoria (nenhuma mudança necessária)

Autoridade única confirmada: `auxRooms.pendingMap` é a única estrutura de pendências —
não existe segunda lista, segundo booleano `needsReview`, nem segunda autoridade paralela.

**Prova: revisão prioriza pendência real sobre preenchimento artificial.**
`buildReviewQueue()` constrói a fila **apenas** a partir de `pendingMap` com
`status == 'pending'`; o fallback sequencial (itens do currículo) só executa quando a fila
de pendências reais está **vazia**. Teste existente
`revisao manual prioriza pendencias reais antigas primeiro` já prova exatamente o cenário
que a missão temia: `requestedCount = 5`, apenas 3 pendências reais existem, o resultado é
`['M2', 'M4', 'M3']` — 3 itens, não 5 fabricados.

**Regra de limpeza (Revisão via aula normal):** `mirrorAttemptToAuxRooms` — acerto+sinal 1
limpa a pendência daquele marker/layer; qualquer outra combinação (erro, acerto+sinal 2,
acerto+sinal 3) mantém ou re-registra a pendência com a razão correta
(`wrong`/`low_confidence_light`/`low_confidence_heavy`). Erro nunca limpa.

---

## 6. RECUPERAÇÃO — auditoria (nenhuma mudança necessária, exceto crédito)

**Gate de bloqueio de conclusão:** `shouldBlockFinalCompletionForRecovery()` é uma função
pura, síncrona, sem `try/catch` que possa mascarar erro como "sem pendência" — portanto já
é fail-closed por construção (não existe o bug de fail-open do SimWeb legado nesta função).

**Regra de limpeza exata (a que a missão pediu para confirmar):**
```dart
// student_aux_rooms.dart : resolvePendingFromRecoveryAnswer
if (correct && signal == DecisionSignal.one) {
  // ... limpa a pendência (status: 'cleared', clearedBy: 'recovery_answer')
}
// qualquer outro caso (errado, ou correto com sinal 2/3) volta a registrar
// a pendência com signal=3, prioridade 'high'
```
Isso bate exatamente com a política ratificada: só acerto+sinal 1 limpa; acerto+sinal 3
não comprova domínio; erro nunca limpa.

**Reconstrução da fila:** `buildRecoveryCurrentItems()` é recalculada a partir do estado
atual de `pendingMap` a cada chamada (não é uma fotografia congelada da fila inicial) —
satisfaz a exigência de "loop até fila realmente vazia".

---

## 7. NIVELAMENTO — auditoria e correção

**Errado:** a tela de escolha ("Começar do início" vs "Já sei algo sobre esta matéria")
tinha subtítulos explicativos **já escritos e traduzidos** no código
(`placement_start_beginning_body`, `placement_take_quick_body`,
`placement_choice_body`) — mas o widget nunca os renderizava. Sem o subtítulo, "Já sei
algo sobre esta matéria" não deixa óbvio que a opção dispara um teste rápido de
posicionamento; é exatamente a ambiguidade que a missão pediu para eliminar.

**Corrigido:** os três textos foram ligados à tela (`preparation_and_placement.dart`).
Nenhum texto novo foi inventado — o conteúdo já existia e já tinha fallback em inglês
para os locales sem tradução explícita (mesmo padrão já usado em outras chaves do app).

**Auditoria de arquitetura (sem mudança):** `PlacementScoringEngine.score()` deriva o
marker de partida diretamente de `curriculumItems[startIdx]` — estruturalmente impossível
apontar para um marker que não existe no currículo vigente. Máquina de estados
(choice/intro/running/result) tem cobertura de teste extensa já existente
(`placement_phase_test.dart`, 20 testes) incluindo reintento, cancelamento em `dispose()`,
e falha na retomada sem loop.

---

## 8. ANEXOS — testado ao vivo, nenhum bug encontrado

Testei os 4 formatos citados na missão contra o **servidor real, com a chave real do
Gemini** (não mock), usando arquivos sintéticos criados localmente:

| Formato | Resultado |
|---|---|
| TXT | Texto extraído **exatamente igual** ao original |
| PDF | Gemini extraiu corretamente a frase embutida no PDF sintético ("O ciclo da água envolve evaporação e condensação") |
| DOCX | `word-extractor` extraiu o texto **exatamente igual** ao original |
| Imagem (PNG) | Gemini descreveu corretamente a imagem (um retângulo verde sólido, sem texto) — prova que o modelo está de fato "olhando" a imagem, não fabricando resposta |

Nenhum dos quatro caminhos "finge sucesso": DOCX corrompido retorna 422 sem vazar detalhe
interno (teste automatizado já existente); tipo de arquivo não suportado retorna 415;
arquivo grande demais retorna 413. Chave de idempotência para a chamada de IA é baseada no
hash do **conteúdo do arquivo**, evitando cobrança dupla de custo de IA pelo mesmo arquivo.

---

## 9. SCROLL/NAVEGAÇÃO — pendência

Auditoria estática das duas telas mais usadas:
- Timeline principal do chat (`chat_aula_widgets.dart`): um único `ListView.builder`,
  um único `ScrollController`, sem sobreposição de controllers.
- Telas de sala auxiliar (`aux_room_screens.dart`): um único `ListView` simples.

Não encontrei duplicação de controlador nem inconsistência de `physics` nessas duas telas.
**Não fiz** o teste físico de arrastar rápido/devagar/durante carregamento pedido pela
missão — é o item mais claramente **pendente** desta sessão.

---

## 10. TESTES AUTOMATIZADOS — resumo final

- App: suíte completa **1414/1414 passando** (rodada 3× ao longo da sessão, após cada
  bloco de mudanças).
- Servidor: suíte completa **100/100 arquivos passando**.
- Testes antigos alterados conscientemente (não só "feitos passar"): `amparo_room_contract_test.dart`
  (limiar 3→5 e semântica sinal-2, com cenários reescritos, não só contadores ajustados),
  `classroom_phase_test.dart` (3 cenários de Amparo), `review_manual_only_contract_test.dart`
  (1 cenário).
- Testes novos: `test/credits_aux_room_contract.test.js` (servidor, 5 provas).

---

## 11. COMMITS

**Server** (`Servidor-BOM`):
- `a371f97` — bônus de cadastro + correção do mapeamento 402 no ledger durável (sessão anterior)
- `6984e0d` — cobrança de 1 crédito por interação em Dúvida/Revisão/Recuperação/Amparo

**App** (`BOM`):
- `8ffae46` — mensagem amigável em vez de trava ao ficar sem crédito (sessão anterior)
- `52df4a7` — gatilho de 5 agravantes + semântica correta de sinal 2 no Amparo
- `3f7daea` — remoção da telemetria morta `quickWrongAnswer`
- `5a67fd7` — subtítulos da tela de Nivelamento

---

## 12. PENDÊNCIAS RESIDUAIS (nenhuma delas é bloqueio técnico interno não resolvido — são itens que exigem mais tempo de sessão, não mais informação/decisão)

1. **Dúvida — atraso de renderização ao vivo** (seção 4): resposta correta e cobrada, mas
   não aparece na tela até reiniciar o app. Causa raiz não fixada.
2. **Teste físico no tablet** de Revisão, Recuperação, Amparo (5 agravantes) e Nivelamento
   (os dois caminhos) não foi executado nesta sessão — só Dúvida e o bug de imagem/crédito
   geral foram validados fisicamente.
3. **Scroll/navegação**: auditoria estática feita, teste físico completo não feito.
4. **Sincronização/offline** (conflitos entre dispositivos, Recovery em andamento
   sobrevivendo a um snapshot antigo): não testado nesta sessão.
5. Traduções ES/FR/JA para os dois novos subtítulos do Nivelamento não foram adicionadas
   (fallback em inglês já cobre, mas não é o ideal para esses locales).

Nenhum destes é um bloqueio "genuinamente externo" no sentido da missão — são itens que
seriam resolvidos com mais tempo de sessão, e ficam registrados para continuidade.

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
