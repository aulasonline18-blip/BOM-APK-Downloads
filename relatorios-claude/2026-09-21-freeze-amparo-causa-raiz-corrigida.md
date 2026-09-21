# Causa raiz do freeze perto da transição do Amparo — encontrada e corrigida

**Data:** 2026-09-21
**Ponto de partida:** relatório anterior (`2026-09-21-validacao-fisica-amparo-freeze-descoberto.md`) descrevia um freeze de tela em branco reproduzível ao acumular respostas erradas perto do gatilho do Amparo, sem causa raiz fechada.

## Método

Reproduzi o freeze ao vivo rodando o app via `flutter run` anexado ao Galaxy Tab A9 (não apenas instalando o APK), o que deu acesso aos logs Dart completos (`[SIM_HTTP]`, `[SIM] LESSON_WORKFLOW_OPEN`, etc.) — algo que a sessão anterior não tinha conseguido.

Descoberta lateral importante: a primeira tentativa de reprodução via `flutter run` sem `--dart-define=SIM_DEV_SERVER_URL=...` apontava para `127.0.0.1:3000` (inexistente no tablet), o que mascarava o problema real com uma cascata de `Connection refused`. Corrigido apontando para o servidor real usado pelo APK instalado (`http://167.179.109.137:3020`, um servidor local desta própria VM, worktree `/root/worktrees/physical-validation-server`, ligado ao Supabase real do projeto).

## Causa raiz confirmada

Reproduzido consistentemente: após 2-3 respostas erradas seguidas, ao tocar em "Continue to next item", a tela trava em branco (barra "Lesson" visível, botão "Review" some, corpo vazio).

Cruzando o log do app com o log do servidor (`/root/.sim-physical-validation-server.log`), encontrei o smoking gun:

```
REQUEST_START /api/student-state/get  ts=...26.607Z
REQUEST_START /api/student-state/get  ts=...26.609Z   <- 2ms depois, mesmo lessonLocalId
REQUEST_END   status=200 ms=366       ts=...26.975Z   <- primeira responde rápido
REQUEST_END   status=200 ms=965298    ts=...52:31.905Z <- segunda demora 16 MINUTOS
```

Duas chamadas **concorrentes** de `/api/student-state/get` para o mesmo `lessonLocalId`, disparadas com 2ms de diferença pelo mesmo cliente. A primeira responde em 366ms; a segunda fica presa no caminho de asserção de posse do recurso (`ensureLessonOwnerFromAuthoritativeStudentState` → `bom_durable_assert_ownership` no Supabase) por ~16 minutos. O app tem timeout de 45s nessa chamada, então desiste bem antes — mas fica sem conteúdo novo, sem erro visível, e o botão "Continue" fica preso em "Preparing the next step." indefinidamente (o orçamento de retry de 4 tentativas se esgota e nada reseta o estado enquanto o aluno não muda de item).

**No código do app**, a causa da dupla chamada: `LabSession._hydrateActiveLessonFromCloud()` (em `lib/features/session/lab_session.dart`) não tinha nenhuma proteção contra chamadas concorrentes — cada disparo (dois gatilhos de lifecycle/resume podem cair perto um do outro) abre sua própria requisição de rede para o mesmo `lessonLocalId`, sem reaproveitar uma leitura já em andamento.

Importante: isso **não é um bug específico do Amparo**. É um bug genérico de "duas leituras remotas concorrentes para a mesma lição" que pode disparar em qualquer transição de item — só coincidiu de aparecer perto do gatilho do Amparo no teste físico anterior porque foi ali que a sequência de erros gerou o timing certo para a corrida.

## Correção aplicada

Adicionado um guarda de single-flight por `lessonLocalId` em `_hydrateActiveLessonFromCloud`: se já existe uma hidratação em andamento para aquele id, a segunda chamada é descartada em vez de abrir uma segunda requisição concorrente. Também melhorei o log de erro de `postJson` para incluir a URI, o erro real e o stack trace (antes só logava a palavra "erro"), o que foi o que permitiu fechar esse diagnóstico e vai ajudar a próxima vez.

Arquivos alterados: `lib/features/session/lab_session.dart`, `lib/sim/external_ai/sim_http_transport.dart`, `test/classroom_phase_test.dart`.

## Prova de que o teste pega o bug de verdade

Escrevi um teste (`M7.1b duas chamadas de resumo quase simultaneas...`) que dispara duas chamadas de resumo de sessão sem esperar a primeira terminar, com uma nuvem fake que conta chamadas. Confirmei manualmente:
- **Sem a correção:** teste falha (`Expected: 1, Actual: 2`).
- **Com a correção:** teste passa (1 chamada).

Suíte completa: `flutter analyze` limpo, `flutter test` — 1475 testes passando.

## Validação física pós-correção

Rebuild do APK debug apontando pro servidor real, reinstalado no tablet, refiz a sequência de respostas erradas no item 7→8. A transição funcionou normalmente (sem freeze). Não consegui reproduzir de novo o cenário exato de concorrência dentro do tempo desta sessão (a corrida de 2ms depende de timing de lifecycle que não é 100% determinístico via automação adb) — a confiança na correção vem principalmente do teste automatizado (que reproduz e prova a causa raiz de forma determinística) e da leitura direta dos logs do incidente original, não só da tentativa de repetição manual.

## Commit

`fix(session): single-flight guard on remote-state hydrate to stop physical freeze near Amparo` — já commitado e enviado para `origin/main` do repositório BOM (commits `fb5d730` + merge `b0481cc`).

## Pendências que ficam para a próxima sessão

- Confirmar em uma sessão física mais longa que o freeze realmente não recorre em um ciclo completo de 5 agravantes → Amparo.
- O bug irmão do anexo TXT (relatado pelo usuário, "tentando anexar um arquivo TXT para gerar uma aula, não funciona") continua não investigado.
- Vale considerar, como endurecimento adicional (fora do escopo desta correção pontual): investigar se o próprio endpoint `/api/student-state/get` no servidor deveria falhar rápido em vez de travar minutos quando encontra contenção — isso é um endurecimento server-side que não fiz porque o pedido era focado na causa raiz no app.
