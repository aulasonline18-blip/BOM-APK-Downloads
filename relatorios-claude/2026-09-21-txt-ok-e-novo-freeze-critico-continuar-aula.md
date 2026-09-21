# TXT confirmado OK em produção + novo freeze crítico na transição aquecimento→aula — 2026-09-21

**Contexto**: retomada do checklist físico contra produção real (`https://simaitutor.com`, SERVER SHA `5f7e0cf1...`), seguindo a instrução de missão contínua (não travar em um único item). Conta usada: `qa-amparo-20260921@sim-internal-test.invalid`. APP SHA no início desta rodada: `9bf2eea` (sem mudança de código nesta rodada — só investigação).

## 1. Bug do anexo TXT — investigado, NÃO reproduzido, `OK_PRODUCTION`

Reproduzi o fluxo completo de "Nova aula" → "Tenho material" → "Arquivo — PDF, TXT, DOC ou DOCX" → anexar `teste-anexo-sim.txt` (arquivo simples, ~200 bytes, UTF-8, texto em português com acentos) contra produção real:

- Upload → `Arquivo enviado. Estou lendo.` → `Conteúdo aproveitável.` (extração bem-sucedida).
- Avancei o onboarding completo (6 etapas) com o anexo referenciado corretamente no resumo final (`1 de 1 material(is) aproveitável(is). teste-anexo-sim.txt`).
- A aula foi gerada e o conteúdo do aquecimento ("Enquanto sua aula fica pronta") refletiu corretamente o tema do TXT (frações equivalentes).

**Conclusão**: o caminho principal de anexo TXT funciona corretamente em produção real, ponta a ponta (extração → onboarding → geração de aula). Não consegui reproduzir nenhuma falha com um TXT UTF-8 simples.

**Risco não confirmado, não testado por falta de tempo**: `src/attachments/attachment-processor.js` faz `file.data.toString('utf8')` sem detectar/tratar BOM ou encodings diferentes (ex.: UTF-16, comum quando o usuário salva o `.txt` pelo Bloco de Notas do Windows com o encoding padrão antigo, ou Latin-1/Windows-1252 com acentuação). Preparei um arquivo de teste `teste-utf16.txt` (UTF-16LE com BOM) e enviei para o tablet (`/sdcard/Download/teste-utf16.txt`), mas não cheguei a testá-lo por causa do bug crítico da seção 2 abaixo, que consumiu o tempo restante. **Próxima ação sugerida**: testar esse arquivo especificamente; se a extração vier com caracteres nulos/garbled, é a causa raiz real do relato do usuário.

**Marcar**: `Anexo TXT (caminho padrão UTF-8)` = `OK_PRODUCTION`. `Anexo TXT com encoding não-UTF-8` = `RETEST_REQUIRED` (arquivo de teste já preparado no tablet).

## 2. NOVO BUG CRÍTICO — "Continuar para a aula" não avança mesmo com o servidor pronto

Ao gerar uma aula nova (a mesma sessão do teste de TXT acima, tema "Frações equivalentes para o 6 ano"), o app entra na tela de aquecimento ("Enquanto sua aula fica pronta") com uma pergunta de prática. Depois de responder a pergunta de prática, aparece o botão **"Continuar para a aula"**.

**Sintoma**: esse botão nunca avança para a aula de verdade. Confirmei:

- **Servidor já tinha terminado** — logs do droplet (`journalctl -u bom-api.service`) mostram `POST /api/complete-lesson` (dois disparos concorrentes, mesmo `financialKey`, ambos 200) às `07:07:49-54Z`, e `POST /api/visual-route` (dois disparos, ambos 200) às `07:07:55-56Z`. Ou seja, a aula estava 100% pronta no servidor.
- **Toquei no botão "Continuar para a aula" repetidamente** entre `07:10Z` e `07:13Z` (mais de 3 minutos depois do servidor já ter terminado) — a tela nunca mudou.
- **Confirmei via `uiautomator dump`** que o botão existe, está com `clickable="true" enabled="true"`, bounds `[143,1666][1057,1761]` (depois `[143,1666][1057,1761]` em nova leitura), e toquei exatamente no centro (`600,1713`) e também na borda esquerda perto do ícone de seta (`390,1713`) — nenhum dos dois teve efeito.
- **Tirei um dump da árvore de UI antes e depois do toque — são idênticos byte a byte** (`diff` sem output), ou seja, o toque não produziu nenhuma mudança de estado visível na árvore de widgets, nem mesmo um rebuild.
- Não há overlay/scrim cobrindo a tela (todos os nós full-screen na árvore têm `clickable="false"`).

**Impacto**: isso bloqueia a entrada em QUALQUER aula nova criada via onboarding completo nesta sessão de teste — e por extensão, impede testar Dúvida/Revisão/Recuperação/Finalização/Amparo a partir de uma aula recém-criada. É potencialmente **mais crítico que os dois freezes já corrigidos nesta sessão**, porque acontece ANTES do aluno sequer começar a aula (não é um caso extremo depois de vários itens — é o primeiro obstáculo de qualquer aula nova).

**Causa raiz: NÃO INVESTIGADA.** Não tive acesso a `flutter run` attached nesta rodada (o app estava instalado como APK release, sem visibilidade de log Dart via `adb logcat` puro — confirmado, filtrei por "flutter", "sim", "error", "exception" no logcat e não veio nada). Suspeita razoável, por analogia com os dois bugs já corrigidos nesta sessão (`fb5d730`/`b0481cc` e `9bf2eea`): mais um caso da mesma família — alguma condição de "pronto" do lado do app que não é reavaliada quando a resposta do servidor chega, agora no ponto de transição aquecimento→aula em vez do avanço entre itens.

**NÃO CORRIGIDO NESTA RODADA** — ficou sem tempo/orçamento para investigar com `flutter run` attached.

## 3. Itens não alcançados nesta rodada (por causa do bug da seção 2)

Dúvida/Revisão/Recuperação (#123), Finalização (#124), Placement (#121), CG-1 (#120), e retomada do Amparo (#122) não foram testados nesta rodada específica — o tempo foi consumido pela investigação do anexo TXT (bem-sucedida) e do novo freeze crítico (não resolvido). Menu/Drawer (#125) foi parcialmente observado (o menu abre corretamente, lista aulas com progresso, tem opção de renomear via "⋮" — não testei o fluxo de rename até o fim).

## PRÓXIMA AÇÃO EXATA

1. Investigar a causa raiz da seção 2 com `flutter run -d 100.124.23.2:5555 --dart-define=FLUTTER_APP_MODE=production --dart-define=SIM_SERVER_URL=https://simaitutor.com` attached, reproduzindo o mesmo cenário (nova aula, aquecimento, "Continuar para a aula" travado).
2. Depois de corrigir: testar também o arquivo `teste-utf16.txt` já preparado em `/sdcard/Download/` no tablet, para fechar de vez a investigação do anexo TXT.
3. Só depois disso, seguir para os itens #120-127 ainda não tocados nesta rodada. Há uma aula ANTIGA já em andamento no tablet ("Fracoes para o 6 ano do ensino fundamental", item 2/60, já passada do aquecimento) que pode ser usada para testar Dúvida/Menu/Restart sem depender de uma aula nova, caso o bug da seção 2 ainda não esteja corrigido.
