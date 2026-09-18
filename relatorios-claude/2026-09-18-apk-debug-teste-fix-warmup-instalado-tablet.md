# APK de debug/teste instalado no tablet — validar fix do vazamento entre contas

Data: 2026-09-18

## Objetivo

Dar ao Joel um jeito de testar o cenário adversarial de troca de conta
(sem fechar o app) antes de decidir promover o AAB 110 na Play Store.

## Build

- **Código**: exatamente o commit `1a9cc08` (o commit que corrigiu o
  vazamento de aquecimento entre contas), buildado a partir de um worktree
  isolado nesse commit exato — nenhum código adicional, nem o bump de
  versão que veio depois (`eb26933`).
- **Comando**: `flutter build apk --debug --dart-define=SIM_SERVER_URL=https://simaitutor.com`
  (mesmo padrão já usado em testes anteriores nesta sessão) — aponta para o
  servidor de produção real (`simaitutor.com`), não para nenhum ambiente de
  dev/staging.
- **Assinatura**: chave de debug padrão do Flutter (não a chave de
  produção) — por isso o pacote mostra `versionCode=109` /
  `versionName=1.0.0` no dispositivo (o número de versão não foi
  incrementado neste build de teste; o código é o do fix, isso é só
  cosmético).
- **SHA-256 do APK**: `b9de787efbb1e83005845d6416f5bbb075df6aae7ade85330ed3239691724103`

## Instalação

O tablet já tinha a build 109 oficial da Play Store instalada (assinatura
de produção). Assinatura de debug é incompatível com update in-place, então
pedi confirmação e, com sua autorização, desinstalei a build da Play Store
e instalei esta build de teste no lugar (`adb uninstall` +
`adb install -r`, dispositivo Galaxy Tab via Tailscale ADB,
`100.124.23.2:5555`). Instalação confirmada com sucesso.

**Importante para o Joel:** o app agora no tablet é este build de teste, não
mais o da Play Store. Nenhum dado de conta se perde (créditos, aulas e
progresso ficam no servidor/Supabase), mas para voltar à versão oficial da
Play Store depois do teste será preciso reinstalar por lá.

## Próximo passo

Pedir ao Joel para repetir o cenário adversarial: logar como conta A, gerar
aula, avançar itens, fazer logout **sem fechar o app**, logar como conta B,
gerar aula nova, confirmar que aquecimento e menu mostram só conteúdo de B
— repetir algumas vezes. Se passar, seguimos para publicar o AAB 110 na Play
Store.
