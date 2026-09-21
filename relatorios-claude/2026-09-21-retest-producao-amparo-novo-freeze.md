# Reteste em produção real do fix de freeze (single-flight hydrate) + novo freeze encontrado

**Contexto:** fix `fb5d730`/`b0481cc` (single-flight guard em `_hydrateActiveLessonFromCloud`, `lib/features/session/lab_session.dart`) já commitado/pushado para `main` do BOM, com 1475 testes passando e `flutter analyze` limpo. Esta sessão tinha como objetivo confirmá-lo fisicamente contra o servidor real de produção (`https://simaitutor.com`, droplet `sim-api-sgp1`, SHA `5f7e0cf`), por instrução explícita do usuário de que produção é a autoridade final do checklist.

## Build de produção

- Endpoint configurado via `--dart-define=SIM_SERVER_URL=https://simaitutor.com` (achado em `lib/sim/config/sim_environment.dart`; não foi necessário `FLUTTER_APP_MODE=production` para este teste, já que isso ativaria trilhas de billing/Play Store não relevantes aqui).
- Confirmado por evidência de rede, não por suposição: `/proc/net/tcp` no tablet mostrou conexão ativa para `A8.90.FF.A9:01BB` = `168.144.255.169:443` (o droplet real).
- APP SHA: build local em `fb5d730`/`b0481cc` (branch main do BOM). SERVER SHA: `5f7e0cf11c9ba3a773e5f9360dea8d180683ca12` (confirmado via `readlink /opt/sim/current` no droplet).

## Bloqueio 1: conta de teste com lição tombstoned (não é bug do fix)

A sessão já autenticada no tablet tinha uma lição resumida de testes anteriores. Toda chamada a `/api/complete-lesson` para aquele `lessonLocalId` retornava 403 `STUDENT_STATE_LESSON_DELETED` (`src/auth/resource-owners.js:136` no Servidor-BOM) — a lição foi tombstoned em produção em algum momento anterior (provavelmente rotina de QA). Não relacionado ao fix. Contornado abrindo uma lição nova ("Menu → New lesson") para obter um `lessonLocalId` limpo.

## Confirmado funcionando

Na lição nova, respondi errado 2x com "I am sure" (aggravantes 1 e 2) e uma vez sem querer certo — em nenhum momento houve o freeze original (resume/hydrate concorrente). A tela avançou normalmente entre itens várias vezes.

## Achado novo: freeze diferente, ainda não corrigido

Após o 3º/4º "Continue to next item", o app parou de responder de forma equivalente ao bug original, mas por outro caminho:

- Nenhuma nova requisição ao servidor desde `2026-09-21T03:48:19Z` (confirmado via `journalctl` no droplet, silêncio total por mais de 4 minutos).
- O processo do app continua vivo (mesmo PID, `pidof` responde) e processa toques (`ViewPostIme pointer` aparece no logcat a cada tap/scroll).
- Nenhuma exceção, nenhum crash, nenhum erro no logcat do processo do app.
- Rolar para o topo confirma que só os itens 1-3 (já respondidos) existem na árvore renderizada; abaixo deles é permanentemente branco, mesmo minutos depois, mesmo com toques registrados.

Ou seja: a resposta do servidor a `/api/complete-lesson` (200, ~5.2s, presente no log) nunca chegou a produzir um novo item na tela — sintoma na mesma família do bug original (rede ok, estado da UI nunca avança), mas **não** é a mesma causa raiz (não há aqui duas leituras concorrentes de `/api/student-state/get`; é avanço de item pós-resposta de `complete-lesson`). Precisa de investigação própria, com `flutter run` conectado ao tablet para visibilidade completa (mesmo padrão usado para achar a causa raiz do bug anterior).

## Status da task #122 (Amparo 5-aggravantes)

**Não pode ser marcada como concluída ainda.** O cenário de 5 erros seguidos abrindo a sala de Amparo nunca foi alcançado em produção: primeiro bloqueado pela lição tombstoned, depois por este novo freeze antes de acumular os 5 aggravantes necessários.

## Recomendação

Abrir uma investigação dedicada (fork com `flutter run` attached) para este novo freeze antes de tentar novamente o reteste do Amparo em produção — sem isso, qualquer novo teste físico provavelmente esbarra no mesmo travamento antes de chegar ao 5º erro.
