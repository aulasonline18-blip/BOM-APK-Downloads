# Correção final da Dúvida por escopo semântico — 2026-09-22

## Causa raiz

O estado efêmero da Dúvida dependia de uma assinatura de conteúdo baseada no histórico. Durante a promoção E1→E2 o histórico pode ficar transitoriamente vazio, e dois controladores publicavam snapshots diretamente no runtime, contornando o setter canônico da sessão. A resposta antiga podia então ser reconstruída com a identidade da experiência nova.

## Correção

- APP `9e667090538b94b3ae5b4824065015ee3ca8c106`.
- `DoubtState` passou a carregar o escopo semântico estável aula/item/experiência.
- Toda escrita de snapshot dos controladores de interação e avanço passa pela autoridade da sessão.
- O estado ativo da Dúvida expira quando o cursor canônico muda de escopo; estados antigos sem a nova identidade mantêm apenas a compatibilidade transitória da assinatura existente.

## Provas automatizadas

- `flutter analyze --no-pub`: PASS.
- `flutter test`: 1508/1508 PASS.
- `./tool/check-sim-reform`: OVERALL PASS.
- Regressão cobre mudança E1→E2 com histórico vazio e proíbe escrita direta de snapshot pelos controladores.

## Prova física

Samsung SM-X216B, APK de produção `f351bf2ac0f736001cd6e1f3a20a6394a7407b78b97d0bfdad881db31e006e55`, servidor real `https://simaitutor.com`:

1. Item 18 E1 da aula de Frações: Dúvida apresentada abaixo do feedback.
2. Item 18 E2: bloco antigo ausente.
3. Feedback de E2: bloco antigo continuou ausente.
4. Item 19: bloco antigo ausente.
5. Encerramento e reabertura do processo: bloco não ressuscitou e o estado permaneceu acessível.

A aula de Kiribati não foi aberta nem alterada.

## Veredito

`DÚVIDA = OK_PRODUCTION` para o corredor causal corrigido.

Promoção do release permanece bloqueada somente pela rotação/desativação da credencial QA que foi exposta no histórico deste repositório de handoff.
