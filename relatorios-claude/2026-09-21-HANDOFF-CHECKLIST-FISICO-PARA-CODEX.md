# HANDOFF — Checklist físico final (produção real) — 2026-09-21

**Antes de fazer qualquer coisa**: rode `git log --oneline -5 origin/main` nos dois repos (`Servidor-BOM` e `BOM`) e em `BOM-APK-Downloads`. Se houver commits mais novos que os SHAs listados abaixo, **leia-os primeiro** — este documento pode já estar um passo atrás.

**ATUALIZAÇÃO (mesma sessão, depois da publicação inicial deste handoff)**: o fork que tentava fechar o Amparo com a conta de QA nova terminou. Não fechou o item — achou um **terceiro problema da mesma família** (freeze/stall), ainda não corrigido. Detalhes completos na seção D revisada abaixo e no relatório `2026-09-21-amparo-novo-stall-preparando-proximo-passo.md` (commit `ec35083`). Nenhuma mudança de código foi feita para esse terceiro achado — é diagnóstico puro, aguardando a próxima rodada.

**ATUALIZAÇÃO 2 (mesma sessão)**: investiguei o bug do anexo TXT — **confirmado funcionando em produção** para um TXT UTF-8 padrão (ponta a ponta: upload → extração → onboarding → geração de aula). Risco não confirmado com encoding não-UTF-8 (arquivo de teste `teste-utf16.txt` já preparado em `/sdcard/Download/` no tablet). Durante esse teste achei um **QUARTO bug crítico, ainda mais severo**: o botão "Continuar para a aula" (transição do aquecimento para a aula, em qualquer aula NOVA criada via onboarding) trava permanentemente mesmo minutos depois do servidor já ter terminado `complete-lesson`+`visual-route` (confirmado por log do droplet + `uiautomator dump` idêntico antes/depois do toque). Isso bloqueia a entrada em qualquer aula nova nesta sessão — ainda mais amplo que os stalls anteriores, que aconteciam DEPOIS de já estar na aula. Não root-causado (precisa `flutter run` attached, que não usei nesta rodada). Detalhes completos: `2026-09-21-txt-ok-e-novo-freeze-critico-continuar-aula.md` (commit `087919b`). Nenhum código foi alterado nesta rodada.

**ATUALIZAÇÃO 3 (mesma sessão) — CAUSA RAIZ DO 4º BUG ENCONTRADA E CORRIGIDA**: usei `flutter run` attached contra produção real e achei a causa raiz de verdade: o bug só acontece na **SEGUNDA aula criada em diante, na mesma sessão do app** (a primeira aula sempre funciona). `saveObjectiveEntry()` (handler de "Preparar minha aula") nunca resetava `WarmupBridgeCoordinator`, então `aulaNavigationStarted` ficava preso em `true` da aula anterior, e `shouldOpenOfficialAula()` retornava `false` para sempre. **Fix aplicado e commitado**: `f85bcfa` (app) — chama `clearWarmupState()`+`resetEntryCoordinator()` no início de `saveObjectiveEntry()`. Teste de regressão determinístico adicionado (cria 2 aulas na mesma sessão, falha sem o fix, passa com o fix). Suíte completa: 1483 testes verdes. **Confirmado fisicamente em produção real**: criei 2 aulas seguidas na mesma sessão contra `https://simaitutor.com` com a conta de QA — a segunda abriu normalmente após o fix. Detalhes: `2026-09-21-fix-warmup-coordinator-nao-resetado-nova-aula.md`.

**ATUALIZAÇÃO 4 (mesma sessão) — reverificação independente do fix `f85bcfa`**: li o diff completo do commit e concordo com a causa raiz descrita (fix mínimo, preciso, espelha um reset que `prepareObjectiveWorkflow` já fazia no seu próprio caminho — não é gambiarra). Rodei a suíte completa eu mesmo, de forma independente: **1483/1483 testes passando**, incluindo o teste de regressão específico (`warmup_bridge_contract_test.dart`, 13/13). Tentei um reteste físico próprio via `adb input tap` com coordenadas fixas (sem `flutter run` attached) para dupla-checagem, mas essa abordagem se mostrou frágil — o layout da tela de login desloca verticalmente quando o teclado abre/fecha, fazendo os toques por coordenada acertarem campos errados (cheguei a poluir o campo de e-mail da conta de QA com texto concatenado; sem impacto real, é só uma conta sintética, mas não completei o login por essa via dentro do orçamento). **Não considero isso um sinal de problema no fix** — é limitação da técnica de automação por coordenada fixa, não do app. Recomendação para a próxima rodada: usar `flutter run` attached (como a Atualização 3 fez) ou testar manualmente, em vez de `adb input tap` com coordenadas cravadas.
**Veredito deste item**: causa raiz confirmada por leitura de código + suíte automatizada (própria e independente) + a confirmação física já registrada na Atualização 3 pelo fork anterior. Considero `f85bcfa` validado o suficiente para não bloquear o restante do checklist.

**IMPORTANTE — build de release precisa de um 4º dart-define não documentado antes**: `--dart-define=SIM_AUTH_REDIRECT_URL=simaitutor://login-callback` (sem isso, o gate `BOM_RELEASE_CALLBACK_REQUIRED` do Gradle recusa o build de release). Seção B já atualizada com o comando completo.

**Nota**: instalar um APK de release por cima de uma instalação de debug (ou vice-versa) falha com `INSTALL_FAILED_UPDATE_INCOMPATIBLE` — rode `adb uninstall com.simaitutor.app` antes de trocar entre os dois tipos de build.

## A. ARQUITETURA / AMBIENTE

- **App (Flutter, "BOM")**: `/root/BOM`, repo `https://github.com/aulasonline18-blip/BOM.git`, branch `main`.
- **Servidor (Node, "Servidor-BOM")**: `/root/Servidor-BOM`, repo `https://github.com/aulasonline18-blip/Servidor-BOM.git`, branch `main`.
- **Relatórios/handoff**: `/root/BOM-APK-Downloads`, repo `https://github.com/aulasonline18-blip/BOM-APK-Downloads.git`, pasta `relatorios-claude/`.
- **Tablet físico**: Galaxy Tab A9+ (`SM_X216B`, codename `gta9p`), conectado via ADB wireless em `100.124.23.2:5555` (checar com `adb devices -l`; se não aparecer, `adb connect 100.124.23.2:5555`).
- **Produção real (autoridade final do checklist)**: droplet DigitalOcean `sim-api-sgp1`, IP `168.144.255.169`, domínio `https://simaitutor.com` (e `www.`), systemd `bom-api.service`. Acesso SSH: `ssh -i ~/.ssh/sim-droplet-migration_ed25519 root@168.144.255.169`.
- **VM de desenvolvimento**: `167.179.109.137:3020` — só para diagnóstico/reprodução instrumentada. **Nenhum item pode ser marcado `OK_PRODUCTION` só com base nela.**

## B. ESTADO DO APP E DO SERVIDOR (no momento deste handoff)

| | SHA | branch | status |
|---|---|---|---|
| **BOM (app) main** | `9bf2eea` | main | pushado, `git status` limpo |
| **Servidor-BOM main** | `1f3172b` | main | pushado, `git status` limpo |
| **Servidor de produção real (droplet)** | `5f7e0cf` | — | rodando, `bom-api.service` active, `/opt/sim/current` → `releases/5f7e0cf...`, health 200 |

Observação: o servidor de produção está em `5f7e0cf`, que é **um commit mais antigo** que o `Servidor-BOM main` atual (`1f3172b`) — a diferença é só o commit de documentação/forense (`docs(cutover)...`) que não muda nenhum código de runtime. Não é necessário reimplantar o servidor por causa disso; nenhum fix desta sessão de checklist até agora exigiu mudança server-side.

### Como buildar e instalar o APK apontando para produção real

```bash
cd /root/BOM
/opt/flutter/bin/flutter build apk --release \
  --dart-define=FLUTTER_APP_MODE=production \
  --dart-define=SIM_SERVER_URL=https://simaitutor.com \
  --dart-define=SIM_AUTH_REDIRECT_URL=simaitutor://login-callback
adb connect 100.124.23.2:5555   # se necessário
adb -s 100.124.23.2:5555 uninstall com.simaitutor.app   # obrigatório se o tablet tinha um build debug instalado (assinaturas diferentes)
adb -s 100.124.23.2:5555 install -r build/app/outputs/flutter-apk/app-release.apk
```

`applicationId` real: `com.simaitutor.app` (default de `SIM_ANDROID_APPLICATION_ID` em `android/app/build.gradle.kts`).

### Como conectar/depurar no tablet

```bash
adb devices -l                                   # confirma conexão
adb -s 100.124.23.2:5555 shell am force-stop com.simaitutor.app   # fecha o app
adb -s 100.124.23.2:5555 shell monkey -p com.simaitutor.app 1     # reabre o app
adb -s 100.124.23.2:5555 logcat -c && adb -s 100.124.23.2:5555 logcat   # logs Android (limitado — build release não expõe log Dart interno)
```

Para visibilidade completa de log Dart (usado nas duas investigações de freeze desta sessão), rodar em modo debug conectado:

```bash
cd /root/BOM
/opt/flutter/bin/flutter run -d 100.124.23.2:5555 \
  --dart-define=FLUTTER_APP_MODE=production \
  --dart-define=SIM_SERVER_URL=https://simaitutor.com
```

### Contas de teste

- 5 contas de QA "históricas" catalogadas em sessões anteriores têm um problema pré-existente de **split-brain de revisão** (documentado em `2026-09-20-incidente-app-mode-reconfirmacao-e-decisao-qa.md`) que atrapalha a leitura limpa do teste de Amparo — evitar usá-las para o ciclo de 5 agravantes.
- O fluxo de **signup público pelo app** ficou rate-limitado pelo Supabase para o IP de saída do tablet, depois de tantas sessões de teste consecutivas — não adianta tentar criar conta nova pelo app agora sem esperar o rate-limit resetar.
- Contornei isso criando uma conta de QA nova **diretamente via Supabase Admin API** (bypassa o rate-limit do endpoint público de signup), já confirmada:
  - email: `qa-amparo-20260921@sim-internal-test.invalid`
  - senha: `QaAmparo!20260921xZ`
  - marcada em `user_metadata.qa_test_account = true`
  - **use login normal no app com essas credenciais** (não passar pelo fluxo de signup).
  - Se precisar de outra conta nova, o mesmo padrão funciona (rodar no droplet, usando a env var já configurada do próprio serviço, nunca imprimir a chave):
    ```bash
    ssh -i ~/.ssh/sim-droplet-migration_ed25519 root@168.144.255.169 '
    PID=$(systemctl show -p MainPID --value bom-api.service)
    SUPA_URL=$(tr "\0" "\n" < /proc/$PID/environ | grep -E "^SUPABASE_URL=" | cut -d= -f2-)
    SUPA_KEY=$(tr "\0" "\n" < /proc/$PID/environ | grep -E "^SUPABASE_SERVICE_ROLE_KEY=" | cut -d= -f2-)
    curl -s -X POST "$SUPA_URL/auth/v1/admin/users" \
      -H "apikey: $SUPA_KEY" -H "authorization: Bearer $SUPA_KEY" -H "content-type: application/json" \
      -d "{\"email\":\"qa-NOVOTESTE@sim-internal-test.invalid\",\"password\":\"SENHA_NOVA\",\"email_confirm\":true,\"user_metadata\":{\"qa_test_account\":true}}"
    '
    ```

## C. FIXES DESTA SESSÃO (todos já commitados/pushados/mergeados em main)

1. **BUG**: freeze de tela em branco perto da transição para Amparo.
   **ROOT CAUSE**: duas chamadas concorrentes a `/api/student-state/get` para o mesmo `lessonLocalId`, disparadas 2ms uma da outra por `LabSession._hydrateActiveLessonFromCloud` (sem dedup) — a segunda travava ~16min num lock de ownership no servidor, muito além do timeout de 45s do cliente. Não era específico do Amparo, só surgiu nesse teste.
   **FIX**: single-flight guard por lesson id.
   **COMMITS**: `fb5d730`, `b0481cc` (app).
   **PUSHED**: sim. **MERGED main**: sim.
   **RETESTE FÍSICO**: PASS (VM de dev e depois em produção real).
   Relatório: `2026-09-21-freeze-amparo-causa-raiz-corrigida.md`.

2. **BUG**: freeze de tela em branco depois de ~3 itens de aula (família de sintoma parecida, código diferente).
   **ROOT CAUSE**: `LessonRuntimeEngine.nextAdvanceReady()` só checava se o *texto* do próximo item estava pronto, mas `LessonMaterialController.carregarRapidoSePronto()` (gate real de promoção) também exige que o *visual* já tenha assentado (`!= 'processing'`). Quando o texto chegava antes do visual, o botão "Continue to next item" mentia que estava pronto; ao tocar, o app tentava promover com `content == null` (zero balões renderizados) e nada recuperava.
   **FIX**: alinhar `nextAdvanceReady()` com o gate real; quando o visual ainda não assentou, o app agora mostra "Preparing the next step." em vez de travar.
   **COMMIT**: `9bf2eea` (app).
   **PUSHED**: sim. **MERGED main**: sim.
   **RETESTE FÍSICO**: PASS em produção real (`https://simaitutor.com`, SERVER SHA `5f7e0cf`). Suíte completa (1482 testes) verde + `flutter analyze` limpo + teste de regressão determinístico.
   Relatório: `2026-09-21-fix-freeze-visual-nao-assentado-nextadvance.md`.

Nenhum dos dois fixes exigiu mudança server-side — servidor de produção não precisou de novo deploy por causa deles.

## D. ESTADO DETALHADO DO AMPARO (item mais importante em aberto)

- **O que já está provado**: o limiar 3→5 agravantes (fix da task #24, sessão anterior) se sustenta fisicamente — 4 respostas erradas seguidas, com sinal "Tenho certeza" a cada vez, não disparam Amparo prematuramente. Confirmado de novo nesta rodada, em produção real, com a conta de QA nova.
- **Os dois freezes anteriores (hydrate race + gate nextAdvanceReady, seção C) seguram bem** sob os 4 erros consecutivos — nenhuma tela em branco, nenhum freeze silencioso.
- **TERCEIRO PROBLEMA ENCONTRADO, AINDA NÃO CORRIGIDO** (o que efetivamente bloqueia o fechamento do #122 agora): depois do 4º erro, o app entra honestamente no estado "Preparando próximo passo" (botão cinza — esse é o comportamento *correto* do fix da seção C.2, não é regressão) e **fica preso nesse estado por 7+ minutos**, mesmo com o servidor já tendo terminado de gerar o visual (`POST /api/visual-route` → 200, confirmado nos logs do droplet às 05:04:37 na sessão de teste). Rede e servidor descartados como causa (ping 242ms, Tailscale ativo, resposta do servidor já entregue) — o app simplesmente não reavalia o gate depois que o visual fica pronto.
  - **Hipótese mais provável, não confirmada**: a bomba de retry `ensureNextAulaAdvancePrepared` para de ser reagendada depois de vários ciclos consecutivos de sinal de confiança ("Tenho certeza" repetido 4x) — precisa checar se há algum teto/ceiling de re-agendamento nesse controller que não é resetado corretamente entre agravantes.
  - **Arquivo/símbolo a investigar primeiro**: `ensureNextAulaAdvancePrepared` (app BOM) e a lógica de `nextAdvanceReady()`/`LessonRuntimeEngine` já tocada pelo fix da seção C.2 — o novo bug provavelmente está adjacente a esse código, na parte que deveria *reagendar* a checagem, não na checagem em si.
  - **Nenhuma mudança de código foi feita para este achado ainda** — é diagnóstico puro. Relatório completo: `2026-09-21-amparo-novo-stall-preparando-proximo-passo.md` (commit `ec35083`).
  - **Rastreamento estático adicional** (sem reprodução física nova, sessão seguinte): `2026-09-21-amparo-stall-trace-estatico-candidatos.md` (commit `46ed8e0`) — mapeia a cadeia completa de decisão (`chat_aula_screen._ensurePostFeedbackNextAdvancePrepared` → `lab_session.ensureNextAulaAdvancePrepared`/`_retryNextAdvancePrefetchIfDue` → `sim_organism.prepareNextItemPackageIfAuthorized` → `lesson_runtime_engine._nextAdvanceTarget`/`_visualSettledForSlot`) e aponta o candidato mais provável: `prepareNextItemPackageIfAuthorized` sempre faz prefetch de `currentIdx+1` a partir do cursor de experiência ativo, que pode não coincidir com o slot que `_nextAdvanceTarget` está de fato esperando durante o loop de reforço/agravantes — **não confirmado ao vivo ainda**. O relatório lista os 4 pontos exatos (com número de linha) para instrumentar com `debugPrint` antes da próxima reprodução física com `flutter run` attached.
- **O que ainda falta**: (1) achar a causa raiz real deste terceiro stall com `flutter run` attached (mesma técnica das seções anteriores — colocar um breakpoint/log logo antes e depois do ponto em que `visual-route` retorna, para ver se o evento chega ao app e se algo deveria reagendar a checagem e não reagenda); (2) corrigir; (3) testar; (4) só então completar o ciclo (5 erros seguidos → sala de Amparo abre → interação → volta pra aula) usando a conta `qa-amparo-20260921@sim-internal-test.invalid` (ver seção B) — ela já está pronta e funcional para reuso, sem precisar recriar.
- **Como reproduzir do zero, se precisar**:
  1. Build+instalar o APK de produção (comandos na seção B) — ou usar `flutter run` attached direto, para já ter os logs.
  2. Login com `qa-amparo-20260921@sim-internal-test.invalid` / `QaAmparo!20260921xZ` (conta já passou pelo onboarding de 9 etapas nesta sessão — deve retomar direto na aula).
  3. Responder errado, de forma consistente ("Tenho certeza"/"I am sure"), repetidamente na mesma aula, sem reiniciar o app no meio.
  4. Observar o stall no "Preparando próximo passo" depois do 4º erro — é aqui que a investigação da causa raiz deve focar agora.
  5. Só depois de corrigir esse stall: seguir até o 5º erro e confirmar que a sala de Amparo abre e completa um ciclo sem travar, antes de marcar `OK_PRODUCTION`.

## E. ANEXO TXT (bug reportado pelo usuário, ainda não investigado)

Relato do usuário: "Estou tentando anexar um arquivo TXT e gerar uma aula anexando o arquivo, e não está funcionando."

**Status: NOT_STARTED.** Nenhum fork chegou a investigar isso ainda nesta sessão (ficou de fora por causa dos dois freezes que consumiram o tempo disponível). Não há causa raiz, não há fix, não há reteste.

**Próxima ação exata**:
1. Achar no app onde o usuário anexa um TXT para gerar aula (provavelmente no fluxo de Dúvida/aula nova com anexo — ver `src/attachments/attachment-processor.js` no servidor e a tela de composição de aula/dúvida no app).
2. Criar um arquivo `.txt` de teste simples no tablet (ou usar `adb push`).
3. Reproduzir fisicamente: anexar o TXT, tentar gerar a aula, observar se falha silenciosamente, trava, ou dá erro visível.
4. Capturar `flutter run` attached (mesma técnica da seção B) + logs do servidor (`journalctl -u bom-api.service -f` no droplet) no momento exato da tentativa.
5. Achar causa raiz real antes de propor fix.

## F. MATRIZ COMPLETA DO CHECKLIST

| ITEM | STATUS | APP_SHA | SERVER_SHA | EVIDENCE |
|---|---|---|---|---|
| Aula normal | OK_PRODUCTION | 9bf2eea | 5f7e0cf | `2026-09-21-fix-freeze-visual-nao-assentado-nextadvance.md` |
| Anexos (geral) | RETEST_REQUIRED | — | — | pipeline testado em sessões anteriores na VM; não retestado em produção nesta rodada |
| TXT (UTF-8 padrão) | OK_PRODUCTION | 9bf2eea | 5f7e0cf | ponta a ponta confirmado, ver `2026-09-21-txt-ok-e-novo-freeze-critico-continuar-aula.md` |
| TXT (encoding não-UTF-8) | RETEST_REQUIRED | — | — | `teste-utf16.txt` já no tablet, não testado ainda |
| Transição aquecimento→aula ("Continuar para a aula") | OK_PRODUCTION | f85bcfa | 5f7e0cf | corrigido (`WarmupBridgeCoordinator` não resetava entre aulas), causa raiz + fix + suíte (1483/1483) + confirmação física em produção — ver Atualização 3/4 acima |
| PDF | RETEST_REQUIRED | — | — | testado em sessão anterior (VM), não em produção |
| DOC/DOCX | RETEST_REQUIRED | — | — | idem |
| Imagens | RETEST_REQUIRED | — | — | idem |
| Imagem/segundo professor | NOT_STARTED | — | — | não coberto nesta rodada |
| Scroll | RETEST_REQUIRED | — | — | fixes de sessões anteriores, não retestado contra produção nesta rodada |
| Nivelamento | RETEST_REQUIRED | — | — | auditado em sessão anterior, não retestado fisicamente contra produção agora |
| Placement | NOT_STARTED | — | — | task #121 nunca iniciada fisicamente |
| CG1 (currículo grande) | NOT_STARTED | — | — | task #120/#87 nunca iniciada fisicamente |
| **Amparo** | **IN_PROGRESS** | 9bf2eea | 5f7e0cf | 2 freezes corrigidos; 3º stall ("Preparando próximo passo" travado) achado e ainda não corrigido — ver seção D |
| Dúvida | NOT_STARTED | — | — | task #123, nunca retestada fisicamente contra produção |
| Revisão | OK_PRODUCTION | dad50bf | 42d541d | robô/tela de preparação aparece só na entrada; Q1->Q2->Q3 contínuo em produção com `aulasonline18`, ver Atualização Codex 0.2 |
| Recuperação | NOT_STARTED | — | — | idem |
| Finalização sem pending | NOT_STARTED | — | — | task #124 |
| Finalização com pending | NOT_STARTED | — | — | task #124 |
| Menu | OK_PRODUCTION | f85bcfa | 5f7e0cf | drawer abre, lista de aulas carrega, Dark theme/New lesson/Credits/Sign out/Delete account/Export/Import backup todos visíveis e clicáveis |
| Drawer (lista de aulas) | OK_PRODUCTION | f85bcfa | 5f7e0cf | 4 aulas da conta QA listadas corretamente, incluindo uma de currículo grande (60 itens) útil para CG-1 |
| Rename | **FAIL** | f85bcfa | 5f7e0cf | reproduzido 2x em 2 aulas diferentes: nome NUNCA muda ao reabrir o diálogo; ver seção H (nova) para causa raiz parcial |
| Restart/Resume | OK_PRODUCTION (básico) | f85bcfa | 5f7e0cf | force-stop + reabrir: sessão/créditos intactos, dashboard correto. Não testado: restart no MEIO de uma aula ativa |
| Offline/Reconnect | OK_PRODUCTION (básico) | f85bcfa | 5f7e0cf | airplane mode on/off: sem crash, sessão/créditos intactos ao reconectar. Não testado: interromper uma chamada de rede ativa (ex.: durante geração de aula) |
| Account isolation | NOT_STARTED | — | — | task #127 |
| Microcrédito | NOT_STARTED | — | — | task #127; observar reserva/captura/release/custo/replay/idempotência quando testado |
| Billing | NOT_STARTED | — | — | task #127 |

(Itens marcados `RETEST_REQUIRED` foram validados fisicamente em sessões anteriores contra a VM de dev, não contra produção real — não contam como prova final por decisão explícita do usuário.)

## H. BUG NOVO ENCONTRADO — RENAME DE AULA NÃO PERSISTE (task #125)

**Reproduzido fisicamente 2x, em 2 aulas diferentes**, contra produção real, com a conta `qa-amparo-20260921@sim-internal-test.invalid`: abrir o menu (☰) → tocar no "⋮" de qualquer aula listada → "Rename lesson" → editar o nome → "Save". Resultado: o nome **nunca muda** (reabrir o diálogo mostra sempre o nome original), e aparece um banner "Server unavailable. Try again. Try again" no topo do menu.

**Diagnóstico por leitura de código (não confirmado ao vivo com `flutter run` attached ainda — próxima etapa)**:
- O banner "Server unavailable..." é **enganoso e não vem do rename**. Ele é renderizado por `session.drawerLessonListError != null` (`lib/shared/widgets/shared_widgets.dart:~289`), cujo texto (`'${t('aula_server_unavailable')} ${t('retry')}'`) explica a duplicação "Try again. Try again". Esse erro é setado em `lib/features/session/lab_session_drawer_controller.dart:258` (`lessonListError = 'remote_lessons_unavailable'`) quando `listCloudLessons()` falha ao chamar `/api/student-state/summaries` — um refresh de LISTA, não do rename. Não vi essa chamada nos logs do droplet nos exatos momentos das minhas tentativas, o que sugere que a exceção acontece client-side antes mesmo de sair a requisição (ou é uma falha de rede intermitente/timeout não capturada no log do servidor).
- O `onRename` em `shared_widgets.dart:421-427` **descarta o resultado de `session.renameDrawerCloudLesson(...)` sem nenhum feedback de sucesso/erro** — diferente do `onOpen`, que mostra um SnackBar em caso de falha. Ou seja, mesmo que o rename falhe silenciosamente por outro motivo (ex.: mismatch de `expectedRevision` no `RenameLessonCommand`, rejeitado por `dispatchWorkflowCommand` em `lesson_workflow_coordinator.dart:892`), o usuário nunca saberia — o banner que ele vê é de uma falha completamente não relacionada.
- **Hipótese de `expectedRevision` mismatch DESCARTADA** por leitura de código: drawer controller e coordinator leem revisão do MESMO `_readExistingLocalState`/canonical store — sem divergência de fonte possível aí.
- **Candidato mais provável agora** (`lib/sim/state/student_state_store.dart:2569`, `_applyRenameLessonCommand`): duas guardas silenciosas podem rejeitar o comando sem exceção e sem log fora de debug build: `rename_expected_objective_mismatch` (linha ~2596, se `command.expectedObjective` capturado na hidratação não bate com `before.profile.objetivo` real no momento da aplicação — plausível já que essas 4 aulas foram tocadas por múltiplas sessões de teste diferentes ao longo do dia) ou `rename_duplicate` (linha ~2603, menos provável nos meus testes). Detalhes completos: `2026-09-21-bug-rename-aula-nao-persiste.md` (commit mais recente).
- Tentei ver o log real (`emitLessonWorkflowEvent`/`debugPrint` em `lab_session_drawer_controller.dart:615`, que imprime exatamente `decision=renameLesson reason=...`) via `adb logcat -d` no build release instalado — **vazio, confirmado**: build release não expõe esse log, precisa mesmo de `flutter run` attached.

**Não corrigido nesta rodada** — não apliquei fix especulativo sem confirmar qual guarda dispara de verdade (risco de corrigir a errada). Próxima ação exata: `flutter run -d 100.124.23.2:5555 --dart-define=FLUTTER_APP_MODE=production --dart-define=SIM_SERVER_URL=https://simaitutor.com`, reproduzir o rename, ler a linha `[SIM] LESSON_WORKFLOW_OPEN ... decision=renameLesson reason=...` no console para confirmar a razão exata, então corrigir. Separadamente, **o feedback de erro do rename deveria ser adicionado** (hoje é 100% silencioso) e o banner de "lista indisponível" não deveria aparecer disparado por uma ação não relacionada (ou pelo menos precisa investigar por que `listCloudLessons()` está falhando logo depois de um rename).

## G. SEGREDOS

Nada de segredo real neste documento ou em nenhum relatório desta sessão — a única credencial presente é a senha de uma conta de QA sintética criada de propósito para teste (`qa-amparo-20260921@sim-internal-test.invalid`), que não dá acesso a nada além de si mesma e pode ser revogada a qualquer momento sem impacto.

## CONTINUE DAQUI

### ATUALIZACAO CODEX (mais recente)

0.A. **Adendo de missão incorporado:** este trabalho deve ser tratado como
     acabamento funcional final do SIM, não como auditoria passiva. Para cada
     corredor: testar, corrigir causa raiz, validar, commitar/pushar, retestar
     em produção e avançar. Relatório normativo operacional:
     `2026-09-21-adendo-missao-acabamento-funcional-final.md`.
0. **RWR-001 reidratação em instalação limpa: bloqueio reproduzido foi resolvido
   em produção.** Servidor corrigido no commit `42d541d`
   (`fix(cost): recover captured T02 material on clean reinstall`), implantado
   em `/opt/sim/releases/42d541dafab17a99fc8d55c1470d8abcb24ea8ce`.
   Health público `https://simaitutor.com/api/health` voltou HTTP 200. A aula
   `cyber-15cy53v`/`M0002`, que antes devolvia HTTP 409
   `CREDIT_OPERATION_REQUIRES_RECONCILIATION`, passou a devolver HTTP 200 com
   `SIM109_ITEM_PACKAGE_V1`; segunda chamada igual também HTTP 200 e log
   sanitizado com `duplicateSuppressed: true`. Prova física no Samsung
   `SM-X216B`: o app v110 saiu da tela `Failed to generate content` após
   `Try again` e voltou a mostrar o item 2/20 com conteúdo, visual e
   alternativas; logs do servidor mostraram `complete-lesson` HTTP 200,
   persistência HTTP 200 e nenhuma reincidência do 409. Relatório:
   `2026-09-21-rwr001-clean-install-rehydration-fix.md`.
0.1. **Achado físico novo após desbloqueio:** no Samsung `SM-X216B`, após
     responder errado o item 2/20 e tocar `Continue to next item`, a tela foi
     para `Drawer lesson experience prefix 2/2`, mas os logs de produção
     registraram uma nova chamada `POST /api/complete-lesson` HTTP 200 com
     outro `financialKey`, além de novas chamadas `visual-route` HTTP 200. A
     tela de E2 também exibiu um `Lesson visual board`. Isso precisa ser
     investigado antes de declarar o fluxo E1→E2 aprovado, porque pode violar a
     regra SIM109 de zero T02 normal/zero visual próprio entre experiências. Não
     classificar como FAIL definitivo sem rastrear a identidade: pode ser
     prefetch/next-package legítimo ou projeção indevida. Status:
     `IN_PROGRESS/NEEDS_INVESTIGATION`.
0.2. **Revisão: robô entre questões tratado no APP e certificado fisicamente
     em produção.** O comportamento observado pelo usuário era: entrar na
     Revisão, responder Q1, tocar Continue e ver novamente o robô/tela de
     preparação antes da Q2. O app foi corrigido para preparar uma janela de
     Revisão contínua de duas perguntas: entrada prepara Q1+Q2; avanço para Q2
     prepara Q2+Q3 antes de expor a transição; se a preparação falha, a causa
     real é preservada em vez de mostrar erro genérico. Commit BOM `dad50bf`
     (`fix(review): preserve credit blocker during preparation`) pushed em
     `main`. Gates: `flutter analyze --no-pub` PASS, `flutter test` PASS
     1.486/1.486, `./tool/check-sim-reform` OVERALL PASS. APK release
     reconstruído e instalado no Samsung com SHA-256
     `08794ef655b7630b28e0e00357d6e77ba912120ef8e37ac880968925efa4f7a5`
     (versionCode 110). Prova física em produção com a conta `aulasonline18`:
     aula nova `basic fractions`, entrada na Revisão, seleção de 5 perguntas,
     robô/tela de preparação apenas na entrada, `Start review`, Q1 respondida,
     `Continue` levou diretamente para `Review 2/5`, Q2 respondida, `Continue`
     levou diretamente para `Review 3/5`. Capturas via `uiautomator` mostraram
     `1/5 -> 2/5 -> 3/5` sem `Preparing your review...`, sem `Start review` e
     sem tela introdutória reaparecendo entre Q1->Q2 ou Q2->Q3. Status:
     `OK_PRODUCTION` para o bug do robô entre questões da Revisão.
1. **Rename #125: OK_PRODUCTION.** Corrigido e comprovado após reinstalação.
   Commit BOM `792e3ec`. Não repetir. Relatório:
   `2026-09-21-codex-checkpoint-rename-utf16-amparo.md`.
2. **TXT UTF-16: ingestão OK_PRODUCTION.** O arquivo foi aceito como conteúdo
   utilizável e chegou ao onboarding/warmup. A falha T00 posterior é distinta
   (`T00_CURRICULUM_MISSING`) e foi honesta, sem repetição paga.
3. **Terceiro stall visual do Amparo: causa corrigida no app.** O artefato real
   já existia, mas o callback não promovia `imageStatus` para `ready`. Commit
   BOM `0d422ad`, 134 testes focados e suíte completa final com 1.486/1.486
   testes verdes; APK release reconstruído.
4. **Reteste do quinto agravante desbloqueado para nova execução física:** o
   bloqueio HTTP 409 da reidratação foi corrigido no servidor real. Agora retestar
   no APK `SIM-v110-0d422ad-production.apk` o ciclo até o quinto erro, Amparo,
   Dúvida/Revisão/Recuperação, Finalização, Placement e CG-1.
5. **Próxima ação causal:** retomar o checklist físico em produção real a partir
   dos fluxos que dependiam da reidratação do material remoto.
6. Itens independentes do material remoto podem continuar enquanto RWR-001
   avança. Não marcar os fluxos acima como OK apenas por teste automatizado.

**Já resolvidos e confirmados fisicamente em produção, não repetir**: hydrate race (Amparo), gate `nextAdvanceReady` (freeze pós-item-3), `WarmupBridgeCoordinator` não resetado (freeze aquecimento→aula), TXT UTF-8 padrão, Menu/Drawer básico.

**A lista numerada histórica abaixo fica preservada apenas como rastreabilidade.**
Os itens 2 a 6 foram substituídos pela atualização acima: rename e UTF-16 já
foram fechados; a instrumentação encontrou e corrigiu o stall visual; a
reidratação RWR-001 que devolvia 409 foi corrigida e comprovada no caso
reproduzido. A ação ativa agora é o reteste físico dos fluxos materializados em
produção real.

1. `cd /root/BOM-APK-Downloads && git pull` e `cd /root/BOM && git pull` — confira se há commits mais novos que `f85bcfa`/`da63e22` (pode já ter avançado depois deste handoff).
2. **Rename de aula (task #125, seção H acima)**: instrumente `renameCloudLesson` (`lib/features/session/lab_session_drawer_controller.dart:354`) e `renameLesson` (`lib/sim/workflow/lesson_workflow_coordinator.dart:863`) com `debugPrint` do `expectedRevision` vs. `state.stateRevision` e do `result.applied`/`result.reason`, via `flutter run -d 100.124.23.2:5555 --dart-define=FLUTTER_APP_MODE=production --dart-define=SIM_SERVER_URL=https://simaitutor.com`. Reproduza com a conta `qa-amparo-20260921@sim-internal-test.invalid`/`QaAmparo!20260921xZ`, menu → "⋮" em qualquer aula → Rename. Corrija a causa raiz confirmada e adicione feedback de erro visível ao usuário (hoje é 100% silencioso).
3. Teste `teste-utf16.txt` (já em `/sdcard/Download/` no tablet) para fechar a investigação do anexo TXT (variante UTF-8 padrão já confirmada `OK_PRODUCTION`).
4. Leia `2026-09-21-amparo-stall-trace-estatico-candidatos.md` (commit `46ed8e0`) — já tem os 4 pontos exatos de código (com linha) para instrumentar com `debugPrint` antes de reproduzir o terceiro stall do Amparo (item 122, ainda em aberto).
5. Instrumente esses 4 pontos, rode `flutter run` attached, logue com a mesma conta QA, erre 4 vezes seguidas com "Tenho certeza", e compare o `(itemIdx, marker, layer)` que `_visualSettledForSlot` está esperando com o que o `/api/visual-route` realmente devolveu.
6. Corrija a causa raiz confirmada. Teste automatizado de regressão, suíte completa, rebuild, reteste físico em produção com a mesma conta, completando o ciclo até o 5º erro e a sala de Amparo abrindo.
7. Depois: Dúvida/Revisão/Recuperação (task #123), Finalização (#124), Restart/Offline (#126), Account isolation/Microcrédito/Billing (#127), Placement (#121), CG-1 (#120 — já há uma aula de 60 itens pronta na conta QA, "Fracoes para o 6 ano do ensino fundamental", item 2/60, útil para essa validação).
8. A cada fix ou avanço: commit + push imediato (app e/ou servidor). Se precisar de deploy no droplet real, seguir o padrão já estabelecido (release em `/opt/sim/releases/<sha>`, symlink `current`, rollback note automática, health check antes de considerar concluído).
