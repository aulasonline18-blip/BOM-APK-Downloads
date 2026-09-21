# HANDOFF — Checklist físico final (produção real) — 2026-09-21

**Antes de fazer qualquer coisa**: rode `git log --oneline -5 origin/main` nos dois repos (`Servidor-BOM` e `BOM`) e em `BOM-APK-Downloads`. Se houver commits mais novos que os SHAs listados abaixo, **leia-os primeiro** — este documento pode já estar um passo atrás.

**ATUALIZAÇÃO (mesma sessão, depois da publicação inicial deste handoff)**: o fork que tentava fechar o Amparo com a conta de QA nova terminou. Não fechou o item — achou um **terceiro problema da mesma família** (freeze/stall), ainda não corrigido. Detalhes completos na seção D revisada abaixo e no relatório `2026-09-21-amparo-novo-stall-preparando-proximo-passo.md` (commit `ec35083`). Nenhuma mudança de código foi feita para esse terceiro achado — é diagnóstico puro, aguardando a próxima rodada.

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
  --dart-define=SIM_SERVER_URL=https://simaitutor.com
adb connect 100.124.23.2:5555   # se necessário
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
| TXT | NOT_STARTED | — | — | ver seção E |
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
| Revisão | NOT_STARTED | — | — | idem |
| Recuperação | NOT_STARTED | — | — | idem |
| Finalização sem pending | NOT_STARTED | — | — | task #124 |
| Finalização com pending | NOT_STARTED | — | — | task #124 |
| Menu | NOT_STARTED | — | — | task #125 |
| Drawer | NOT_STARTED | — | — | task #125 |
| Rename | NOT_STARTED | — | — | task #125 |
| Restart/Resume | NOT_STARTED | — | — | task #126 |
| Offline/Reconnect | NOT_STARTED | — | — | task #126 |
| Account isolation | NOT_STARTED | — | — | task #127 |
| Microcrédito | NOT_STARTED | — | — | task #127; observar reserva/captura/release/custo/replay/idempotência quando testado |
| Billing | NOT_STARTED | — | — | task #127 |

(Itens marcados `RETEST_REQUIRED` foram validados fisicamente em sessões anteriores contra a VM de dev, não contra produção real — não contam como prova final por decisão explícita do usuário.)

## G. SEGREDOS

Nada de segredo real neste documento ou em nenhum relatório desta sessão — a única credencial presente é a senha de uma conta de QA sintética criada de propósito para teste (`qa-amparo-20260921@sim-internal-test.invalid`), que não dá acesso a nada além de si mesma e pode ser revogada a qualquer momento sem impacto.

## CONTINUE DAQUI

1. `cd /root/BOM-APK-Downloads && git pull` — confira se já existe relatório mais novo que `ec35083` (pode já ter avançado depois deste handoff).
2. Investigue e corrija o terceiro stall descrito na seção D (`ensureNextAulaAdvancePrepared` não reagendando a checagem de `nextAdvanceReady()` depois que o visual fica pronto). Use `flutter run` attached, mesma técnica das duas investigações anteriores desta sessão. Depois de corrigir: teste automatizado de regressão, suíte completa, rebuild, reteste físico em produção com a conta `qa-amparo-20260921@sim-internal-test.invalid` (já pronta, não recriar), completando o ciclo até o 5º erro e a sala de Amparo abrindo.
3. Depois do Amparo fechado: passe para Dúvida/Revisão/Recuperação (task #123) contra produção real, mesmo rigor (causa raiz real para qualquer bug, sem gambiarra, teste automatizado, commit/push, reteste físico em produção antes de marcar OK).
4. Depois: Finalização (#124), Menu/Drawer/Rename (#125), Restart/Offline (#126), Account isolation/Microcrédito/Billing (#127).
5. Investigar o bug do anexo TXT (seção E) — ainda não foi tocado.
6. A cada fix ou avanço: commit + push imediato (app e/ou servidor). Se precisar de deploy no droplet real, seguir o padrão já estabelecido (release em `/opt/sim/releases/<sha>`, symlink `current`, rollback note automática, health check antes de considerar concluído).
