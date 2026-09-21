# Checkpoint Codex - rename, TXT UTF-16 e stall visual - 2026-09-21

## Escopo e ambiente

- Produção real: `https://simaitutor.com`.
- Tablet: Samsung SM-X216B via ADB.
- BOM inicial: `f85bcfaf21fe03b4de458407c423fc67b894cc1d`.
- BOM após rename: `792e3eca4e5613a19290b1eb88c3d1e4f678d12e`.
- BOM após visual: `0d422ad`.
- Servidor-BOM permaneceu em `1f3172b09c178165d609345fbb09948486f7a56e`.

## Rename de aula - OK_PRODUCTION

O rename falhava por duas causas encadeadas:

1. o `TextEditingController` do diálogo era descartado durante a animação de
   fechamento da rota;
2. estados remotos antigos continham shells inválidos de
   `sim109ItemPackages`, porque o contrato remoto removia o texto interno do
   pacote, mas mantinha sua estrutura. A hidratação falhava antes do comando
   de rename.

O diálogo agora possui e descarta seu controller no ciclo correto. O contrato
remoto não serializa pacotes pedagógicos locais, e o leitor rejeita os shells
contaminados já existentes. Falha de rename também recebe mensagem visível.

Validação:

- `git diff --check`: PASS;
- `flutter analyze --no-pub`: PASS;
- suíte completa após todos os alinhamentos deste checkpoint: 1.486 testes PASS;
- APK release v110 instalado;
- nome `QA-Rename-Passed` persistiu no servidor e reapareceu após reinstalação.

Commit: `792e3eca4e5613a19290b1eb88c3d1e4f678d12e`.

## TXT UTF-16 - INGESTAO OK_PRODUCTION

O arquivo `teste-utf16.txt` foi selecionado no fluxo real de objetivo. O app
mostrou `Usable content.` e `1 of 1 material(s) usable.`. O onboarding de
material terminou e o warmup abriu. Isso comprova seleção, leitura e envio do
TXT UTF-16 contra produção.

Depois da ingestão, T00 foi chamado, mas a resposta do provider foi rejeitada
como `T00_CURRICULUM_MISSING`. O app mostrou falha humana recuperável. Um retry
controlado foi bloqueado pelo cost gate, sem segunda chamada paga. Portanto, a
falha posterior não é do decoder/anexo UTF-16.

## Stall em "Preparing the next step" - causa corrigida

A reprodução attached comprovou que o próximo slot já possuía uma URL real de
imagem, mas seguia persistido com `imageStatus=processing`. O callback de
sucesso gravava a URL e não promovia o status para `ready`. O gate de avanço
esperava para sempre por uma imagem que já existia.

Alinhamento aplicado:

- callback visual bem-sucedido persiste URL, metadados e status `ready`;
- estado histórico com imagem real e status antigo `processing` é reconhecido
  como visual assentado;
- nenhuma imagem falsa, bypass de N3 ou fallback paralelo foi criado.

Validação:

- `git diff --check`: PASS;
- `flutter analyze --no-pub`: PASS;
- 134 testes focados PASS, incluindo sala de aula e avanço oficial;
- suíte completa final: 1.486 testes PASS;
- regressões novas cobrem callback de sucesso e estado histórico inconsistente.

Commit: `0d422ad` (`fix(classroom): settle completed lesson visuals`).

## APK release reconstruido

- Artefato: `SIM-v110-0d422ad-production.apk`.
- SHA-256: `f4ae2c365ff73a3f6dc088e1aa668381d444428487372b0f41cf41e7ae0a5b6b`.
- Tamanho: aproximadamente 74 MiB.
- Endpoint: `https://simaitutor.com`.
- Callback: `simaitutor://login-callback`.
- Instalacao no SM-X216B: PASS.
- Login release: PASS.
- Drawer e aulas remotas: PASS.

## Bloqueio externo observado no reteste limpo

Uma instalação limpa não possui os pacotes pedagógicos locais, por desenho.
Ao abrir aula remota, o app pediu o resultado durável ao servidor. Produção
respondeu repetidamente HTTP 409 com
`CREDIT_OPERATION_REQUIRES_RECONCILIATION`, antes de qualquer nova aula ser
entregue. O app mostrou `Failed to generate content` e `Try again`; não fingiu
sucesso e não criou conteúdo sintético.

Esse bloqueio pertence à convergência da saga econômica RWR-001 em andamento.
Não foi criado contorno no app e o worktree paralelo não foi tocado.

Consequência: o ciclo físico até o quinto agravante, Dúvida, Revisão,
Recuperação, Finalização, Placement e CG-1 permanecem bloqueados quando exigem
materialização remota. Itens independentes devem continuar; estes itens devem
ser repetidos após a reconciliação econômica chegar à produção.

## Evidências

- `evidencias/2026-09-21-rename-release/rename-persisted-release-v110.png`
- `evidencias/2026-09-21-rename-release/logcat-release-v110.txt`
- `evidencias/2026-09-21-codex-checkpoint/utf16-release-logcat.txt`
- `evidencias/2026-09-21-codex-checkpoint/release-v110-reconciliation-blocker.png`
- `evidencias/2026-09-21-codex-checkpoint/release-v110-logcat-filtered.txt`
- `evidencias/2026-09-21-codex-checkpoint/production-reconciliation-logs.txt`
