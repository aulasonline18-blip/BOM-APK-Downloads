# Validação física retomada (Amparo) — achado crítico: freeze reproduzível no gatilho do Amparo

**Data:** 2026-09-21
**Dispositivo:** Galaxy Tab A9 (SM-X216B), app `com.simaitutor.app` versionCode 110, instalado em 2026-09-19, última atualização 2026-09-20.

## Descoberta prévia importante: servidor real usado pelo tablet

O APK instalado no tablet **não aponta nem para esta VM de dev (porta 3000) nem para o droplet real (simaitutor.com)**. Extraindo `kernel_blob.bin` do APK (`assets/flutter_assets/`), encontrei o literal `http://167.179.109.137:3020` — um TERCEIRO servidor, saudável (`/api/health` → `{"status":"ok","service":"sim-api"}`, `/api/readiness` → `{"status":"ready","service":"bom-api"}`). Todas as ações físicas deste relatório foram contra esse servidor, não os dois que eu vinha auditando esta sessão. Isso explica por que tentativas anteriores de correlacionar cliques no app com logs desta VM ou do droplet real davam zero resultado.

## O que foi validado com sucesso

1. **Limiar de 3→5 agravantes (fix da task #24) se sustenta fisicamente**: respondi errado 4 vezes seguidas (com "I am sure" a cada vez) sem o Amparo disparar prematuramente — antes do fix disparava com 3. Este ponto está confirmado.
2. **Janela de reset do orçamento de retry (~100s) funciona**: em certo momento o botão "Continue to next item" ficou desabilitado ("Preparing the next step.") por mais de 100 segundos; voltei a checar depois de ~100s adicionais e ele reabilitou sozinho, sem precisar de restart. Consistente com o fix documentado em `project_sim_next_advance_permanent_stuck_fixed.md` (janela de 2 min).
3. **Restart/Resume (parte da task #126) funciona corretamente**: em dois momentos diferentes o app travou (tela em branco, botão "Review" some do topo) e, após `force-stop` + reabrir, o app **sempre** voltou exatamente para o item correto, com a mensagem "No problem. Let us resume step by step." — nenhuma perda de posição.

## Achado crítico novo (não estava mapeado antes)

**O app trava com tela em branco especificamente na transição de abertura do Amparo — após ~5 respostas erradas seguidas.**

Sequência exata observada:
1. Acumulei repetidas respostas erradas ("Not this time. Let us repair the foundation.") em itens consecutivos, sempre confirmando com "I am sure".
2. Na resposta errada seguinte (seria a 5ª ou 6ª contando o histórico total, não consigo cravar o número exato porque um restart intermediário — motivado por um travamento anterior não relacionado ao Amparo — pode ter zerado a contagem parcialmente), toquei em "Continue to next item".
3. A tela foi para um estado **em branco total**: só a barra "Lesson" + barra de progresso ficam visíveis; o botão "Review" desaparece do topo (nas travas anteriores ele continuava visível — aqui não). Isso persistiu por mais de 15s sem nenhuma mudança.
4. `netstat` no tablet mostrou conexões TCP ativas com o servidor (`167.179.109.137:3020`), incluindo uma em `CLOSE_WAIT` com ~190KB não lidos no buffer — ou seja, o servidor respondeu, mas o app não processou a resposta.
5. Reiniciei o app (`force-stop` + reabrir). Ele voltou para o mesmo item da aula normal (Item 6/20), **não para uma sala de Amparo** — ou seja, se o Amparo chegou a ser decidido no lado do app antes de travar, essa decisão se perdeu no restart, e o aluno simplesmente volta para a aula normal sem receber o suporte que a sequência de erros deveria ter acionado.

**Impacto:** se isso reproduz de forma consistente (só testei uma vez até o fim, por causa do orçamento de tempo desta sessão), significa que um aluno que erra bastante seguido pode nunca ver a sala de Amparo — o app trava exatamente no momento da transição, e ao reabrir volta pra aula normal como se nada tivesse acontecido. Isso pode ser a causa raiz (ou uma causa irmã) dos incidentes já registrados em `project_sim_review_room_retry_bug.md` e `project_sim_tablet_freeze_handler.md`, que também descrevem travamentos sem causa raiz fechada em torno de transições de sala.

**Não root-caused**: não consegui ver logs internos do app (as tags `[SIM_OBS]`/`[SIM_AUTH]` visíveis no bytecode do app não aparecem no logcat desta build — provavelmente vão para `dart:developer log()`, que só é capturado com uma sessão de debug/DevTools anexada, não por `adb logcat` puro). Também não tenho acesso a logs do servidor `167.179.109.137:3020` (não é uma máquina que eu controlo). Uma investigação de causa raiz precisaria: (a) rodar o app via `flutter run` conectado a esse tablet para capturar os logs Dart completos, ou (b) acesso ao servidor 3020 para correlacionar exatamente qual chamada travou.

## Não concluído nesta sessão (ficam pendentes)

- **Confirmação de que o Amparo realmente abre e completa um ciclo** (5 agravantes → sala de Amparo → resposta → volta pra aula) — bloqueado pelo freeze acima.
- **Bug do anexo TXT** relatado pelo usuário ("tentando anexar um arquivo TXT para gerar uma aula, não funciona") — não investigado nesta sessão por falta de orçamento de tempo; fica como prioridade da próxima rodada.
- Tasks #123 (Doubt/Review/Recovery), #124 (Finalização), #125 (Menu/Drawer/Rename), #127 (isolamento de conta + billing) — não iniciadas nesta sessão.

## Recomendação

Abrir uma investigação dedicada ao freeze de transição de sala (Amparo, e possivelmente Review/Recovery pelo padrão já visto), com o app rodando via `flutter run` anexado ao tablet para captura completa de logs, antes de continuar o restante do checklist físico — esse bug, se confirmado recorrente, é mais importante do que fechar os itens de checklist restantes.
