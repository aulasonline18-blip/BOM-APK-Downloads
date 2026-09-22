# Walkthrough físico final + tentativa de AAB — 2026-09-22

Continuação do `2026-09-22-sweep-exaustivo-final-consolidado.md` (commit `84c882b`). Objetivo: completar o walkthrough interativo pendente (scroll manual + resposta + avanço) e gerar o AAB final.

## Técnica usada

Abandonei `adb input tap` com coordenadas memorizadas (frágil, já documentado em rodadas anteriores). Usei o ciclo `uiautomator dump` → parsear `content-desc`/`bounds` do XML → calcular centro → tocar, imediatamente antes de cada toque, sem reusar coordenadas de um dump anterior. Funcionou de forma confiável depois que passei a respeitar essa disciplina (um mistoque real aconteceu quando reusei coordenadas de um dump anterior ao layout mudar — ver abaixo).

## Walkthrough físico — resultado

Conta `qa-amparo-20260921@sim-internal-test.invalid`, produção real (`https://simaitutor.com`), APK já instalado (`0ec2440691de8a70141146ea59be340f1870c9edd6ce7cb34df12aee5ca93b45`, commit `58a2c81`). A sessão já estava no Item 16/20 (Placement, "Fractions in Real Life") de uma rodada anterior.

1. **Scroll manual (`df09c02`/`58a2c81`)**: dois swipes independentes, em dois momentos diferentes (E1 e E2), sempre reconferidos após pausa de 2-3s. **Nenhum salto de volta em nenhum dos dois casos** — o conteúdo ficou exatamente onde o gesto manual o deixou. `PASS`.
2. **Avanço E1→E2 (`cdd09e2`, sem depender do `finite_feedback_auto_advance.dart` removido)**: respondi corretamente (A, 3/8), sinal "I am sure", toquei "Continue to experience 2" — abriu a experiência 2 (`Comparative Table`/nova pergunta) corretamente. `PASS`.
3. **Avanço E2→próximo item**: respondi corretamente (B, 2/5), sinal "I am sure", toquei "Continue to next item" — barra de progresso foi de 75%→80%, Item 17/20 carregado com conteúdo e visual novos. `PASS`.

### Incidente registrado durante o teste (não é regressão de código)

No meio do passo 3, um toque meu com coordenadas de um dump anterior (o layout já tinha mudado) abriu sem querer o modal de Dúvida ("I need help with this question"); ao tentar fechar com back, o app foi para segundo plano/home inesperadamente. Ao reabrir, o app **corretamente não perdeu nem duplicou nada**: voltou ao Item 16/E1 com a resposta A já marcada como correta (não pediu para responder de novo, não gerou nova cobrança). Separadamente, durante uma pausa de observação, o logcat mostrou `FreecessHandler: freeze com.simaitutor.app` — congelamento de processo pelo próprio Android/Samsung por economia de bateria (já documentado como comportamento conhecido do tablet nesta sessão, não relacionado a nenhum commit desta missão). `force-stop` + reabertura resolveu, sem perda de estado. Depois disso repeti o passo 3 do zero com sucesso limpo (relatado acima).

**Conclusão dos dois corredores tocados pelo sweep**: `PHYSICAL_RETEST_RESULT = PASS` (scroll e avanço, ambos confirmados fisicamente em produção real, sem regressão).

## Bloqueio novo para o AAB — árvore de trabalho suja por outra sessão concorrente

Ao gerar o AAB (`scripts/build-bom-production-aab.sh`), descobri que **outra sessão está ativamente editando `/root/BOM` meio do meu teste**:

- `git rev-parse HEAD` = `a1c79ae41d6240dbd0c6b514eaedf2e26a436bc2` — um commit novo, não conhecido antes (`fix(doubt): bind timeline messages to originating experience`), um passo à frente do `58a2c81` que eu esperava.
- `git status --short` mostra **mudanças não commitadas** em `lib/features/classroom/chat_aula_widgets.dart` (+39 linhas), `test/canonical_pedagogical_scroll_test.dart`, `test/chat_aula_widgets_test.dart` (+52 linhas) — uma nova função `_clampManualScrollOutOfBottomReserve()`, que impede o scroll manual de entrar na área de padding reservada vazia no fim da timeline (não é um "jump corretivo pedagógico" como o que foi removido em `df09c02` — só um clamp de limite de conteúdo — mas ainda assim é uma nova intervenção de scroll, tecnicamente no mesmo corredor que acabei de validar, e ainda não teve suíte nem reteste físico).

**Eu NÃO commitei nem descartei essas mudanças** — são trabalho em andamento de outra sessão, e mexer nelas seria correr por cima de alguém. O AAB que eu já tinha gerado (`scripts/build-bom-production-aab.sh` já tinha rodado antes de eu notar isso) foi construído em cima dessa árvore suja:

```
FINAL_AAB_SHA256 (NÃO CONFIÁVEL COMO ARTEFATO FINAL): e51b5c15063944c597c8288fcd1643c9b5fa41f8bee0e0ab8d556aa35e05ab44
```

Esse hash **não corresponde a nenhum commit limpo e pushado** — corresponde a `a1c79ae4` + as 3 mudanças não commitadas acima. Não é rastreável, não pode ser o artefato de release final.

## Veredito desta rodada

```
PHYSICAL_RETEST_REQUIRED (herdado do sweep anterior): scroll (df09c02/58a2c81) e avanço de feedback (cdd09e2)
PHYSICAL_RETEST_RESULT: PASS (ambos os corredores confirmados fisicamente em produção real, ver acima)
FINAL_APK_SHA256: 0ec2440691de8a70141146ea59be340f1870c9edd6ce7cb34df12aee5ca93b45 (inalterado, commit 58a2c81, já validado fisicamente)
FINAL_AAB_SHA256: não confiável nesta rodada — árvore de trabalho suja por sessão concorrente (ver acima); precisa ser regerado depois que o commit `a1c79ae4` + o trabalho em andamento de scroll (_clampManualScrollOutOfBottomReserve) forem commitados/pushados e testados
RELEASE_CANDIDATE: NO
```

**Motivo do NO**: o único item não satisfeito é o AAB final rastreável. O walkthrough físico interativo (o outro item pendente do relatório anterior) está agora `PASS`.

## CONTINUE DAQUI

1. Verificar se a outra sessão terminou e commitou/pushou o trabalho em `chat_aula_widgets.dart` (`_clampManualScrollOutOfBottomReserve`) — `cd /root/BOM && git status --short` e `git log --oneline -5`.
2. Se sim: rodar `flutter analyze` + `flutter test` completo (incluindo `canonical_pedagogical_scroll_test.dart`, que já foi atualizado para essa mudança) para confirmar que não introduziu regressão.
3. Se o novo clamp de scroll passar nos testes: reteste físico rápido específico dele (fazer overscroll manual até o fim da timeline e confirmar que só o clamp de padding-vazio acontece, sem nenhum salto pedagógico — mesma técnica de `uiautomator dump` antes de cada toque usada nesta rodada).
4. Gerar o AAB de novo (`scripts/build-bom-production-aab.sh`, mesmas env vars desta rodada: `SIM_SERVER_URL=https://simaitutor.com`, `SIM_CHECKOUT_RETURN_ORIGIN=https://simaitutor.com`, `SIM_AUTH_REDIRECT_URL=simaitutor://login-callback`, `SIM_ANDROID_APPLICATION_ID=com.simaitutor.app`) a partir de uma árvore **limpa e commitada**, registrar o novo `FINAL_AAB_SHA256`.
5. Se tudo isso fechar: `RELEASE_CANDIDATE = YES` — essa seria a conclusão real de toda a missão (checklist físico + auditoria econômica + auditoria estrutural + sweep exaustivo + walkthrough físico + build final).
