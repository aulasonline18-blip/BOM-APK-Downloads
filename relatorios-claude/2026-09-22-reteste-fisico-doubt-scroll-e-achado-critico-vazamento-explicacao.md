# Reteste físico dos commits `a1c79ae`/`9a668a6` + achado crítico novo — 2026-09-22

**Contexto**: continuação do relatório `2026-09-22-RELATORIO-FINAL-SWEEP-E-RELEASE.md` (commit `210c5ae`), que deixou `RELEASE_CANDIDATE = NO` só por falta de confirmação física interativa dos dois commits mais recentes. Login já estava ativo no tablet (sessão QA preservada), então não foi necessário lidar com a tela de login/toque no botão Google.

## O que foi confirmado (PASS)

1. **Dúvida não vaza entre experiências (`a1c79ae`)**: enviei uma Dúvida em E1 (Item 17/20, "Why common denominator" — pergunta sugerida), recebi a resposta ("Think of a fraction as a specific size of a piece..."), avancei para E2 ("Continue to experience 2"). A pergunta/resposta de Dúvida de E1 **não apareceu** na timeline de E2 — confirmado por dump completo da árvore de semântica, do topo ao fundo da tela, sem nenhum traço do bloco de Dúvida entre o feedback de E1 e o cabeçalho do item E2.

2. **Scroll contido dentro da timeline renderizada (`9a668a6`)**: em E2, apliquei 3 e depois 4 arrastos agressivos consecutivos (fling) para além do fim do conteúdo. A posição de repouso final foi **idêntica, pixel a pixel**, nas duas tentativas (mesmo texto "A" destacado, mesma quantidade de espaço reservado abaixo) — o clamp mantém uma posição estável e determinística em vez de permitir uma área vazia crescente ou instável.

## Achado novo, não relacionado aos 2 commits testados, mas descoberto durante o teste (CRÍTICO)

Depois de avançar para E2, respondi (sem querer, por reaproveitar uma coordenada de toque de uma tela anterior) a pergunta **"If you are asked to calculate (3/5) / (1/2), which method is correct?"** com a alternativa errada (A). O feedback mostrou corretamente o cabeçalho de erro **"Not this time. Let us repair the foundation."** (i18n fixo, esperado), mas a caixa verde de explicação abaixo dele mostrou **o texto exatamente idêntico, palavra por palavra**, à resposta de Dúvida que eu tinha recebido minutos antes para uma pergunta completamente diferente ("Why common denominator", sobre adição/subtração de frações) — incluindo o mesmo exemplo específico "1/4 + 2/4 = 3/4".

A pergunta atual era sobre **divisão** de frações ((3/5) / (1/2)); a explicação mostrada é sobre **adição/subtração** e por que elas precisam de denominador comum — tematicamente adjacente, mas não é uma explicação de por que a alternativa A está errada nem por que a B é a correta para *esta* pergunta. A probabilidade de o provedor de IA gerar coincidentemente o mesmo texto, com o mesmo exemplo numérico específico, para duas perguntas diferentes em turnos de conversa diferentes é desprezível — isso tem cara de **reaproveitamento indevido de conteúdo em cache** (a explicação de erro está puxando o conteúdo do último `doubt-response` em vez de gerar/exibir a explicação própria da questão atual).

Confirmei pela árvore de semântica completa, em ordem de renderização (topo→fundo), que não há nenhum outro bloco de Dúvida na tela nessa hora — ou seja, **não é uma bolha de Dúvida antiga ainda visível por engano**; é o próprio bloco de explicação de resposta errada que está mostrando o conteúdo errado. Print de tela anexado como evidência: `wrong_feedback_leak.png` (não commitado neste relatório por ser um artefato de trabalho, mas a transcrição completa do texto está acima, palavra por palavra, e pode ser reproduzida).

Fiz uma checagem rápida de código (`grep` por `why_wrong`/`whyWrong` em `lib/features/classroom/*.dart`) e não encontrei esse nome — a fonte real desse conteúdo não foi identificada nesta rodada. **Não investiguei a causa raiz nem tentei corrigir** — isso está fora do escopo desta verificação pontual e merece uma investigação dedicada com `flutter run` attached, reproduzindo deliberadamente a sequência (Dúvida em um item → resposta errada em outro item logo em seguida) e inspecionando de onde vem o texto da caixa verde de erro.

## Reavaliação do veredito

`RELEASE_CANDIDATE` permanece **NO** — mas agora por um motivo mais forte e mais específico que a lacuna de confirmação física anterior (que está resolvida: os dois commits testados fisicamente passaram). O motivo agora é este achado novo: um possível vazamento de conteúdo de Dúvida para a caixa de explicação de resposta errada, que é uma classe de bug adjacente à que `a1c79ae` corrigiu, mas não coberta pelo teste automatizado existente (que testa vazamento entre a lista de mensagens de Dúvida, não entre o cache de Dúvida e o widget de explicação de erro).

**Recomendação para a próxima rodada**: priorizar a investigação deste achado antes de qualquer novo build de release. Reproduzir deterministicamente: (1) enviar uma Dúvida em qualquer item, (2) errar a pergunta seguinte (mesmo item ou item diferente), (3) verificar se a explicação de erro mostrada é genuinamente gerada para a nova pergunta ou é o cache da Dúvida anterior. Se confirmado, achar o code path que renderiza a explicação de erro e ver por que ele está lendo do estado/cache de Dúvida em vez do campo próprio da resposta.

## Estado dos artefatos

Nenhuma mudança de código nesta rodada — só verificação física + este achado. `FINAL_APK_SHA256`/`FINAL_AAB_SHA256` do relatório anterior (`210c5ae`) continuam válidos como build candidato, mas **não devem ser promovidos a produção** até este achado ser investigado e, se confirmado como bug real, corrigido.
