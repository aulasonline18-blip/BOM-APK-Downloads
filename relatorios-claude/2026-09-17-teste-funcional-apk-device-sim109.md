# Teste funcional em device físico — SIM109 Economic Final

**Data:** 2026-09-17
**Escopo:** `sim109-economic-final-app` + `sim109-economic-final-server`, branch `reform/sim109-economic-final-construction` (main já atualizado, ver relatórios anteriores).
**Device:** tablet físico Samsung Galaxy Tab A9 5G (SM-X216B, Android 16), conectado via Tailscale (IP `100.124.23.2`), instalado via ADB.

## Ambiente de teste

- APK debug (sem assinatura de produção) compilado com `flutter build apk --debug --dart-define=SIM_SERVER_URL=http://100.115.54.33:3012`, apontando para uma instância standalone do servidor (não a produção `sim-api.service` nem o staging `bom-api-staging.service` — uma terceira instância isolada, porta 3012, mesmo código do branch, mesmas chaves de IA da produção).
- APK: `app-debug.apk`, SHA-256 `4e6fe5ff98da6254851514609841d8319cc7649d924de6565d7a85b5b0120d45`.
- Servidor de teste rodava exatamente o código do commit `4b9c939` (HEAD do server na época do teste), com `TEST_CREDIT_EMAILS`/`TEST_CREDIT_BALANCE` para crédito ilimitado nas contas de teste, sem qualquer outra alteração de comportamento.
- Nenhum serviço de produção ou staging foi tocado. A instância de teste foi encerrada ao final.

## O que foi testado e resultado

1. **Login** — Testado. Login via Google OAuth (conta `aulasonline18@gmail.com`, já sessionada no tablet) funcionou; `/api/credits/me` confirmou `testCreditMode: true`, saldo ilimitado.
2. **Onboarding (9 etapas)** — Testado, completo. Todas as 9 etapas (objetivo, contexto, uso, prazo, meta, bloqueio, condução, dados pessoais, resumo) responderam corretamente; validação de campo obrigatório funcionando (botão "Salvar e continuar" só habilita com texto suficiente no campo livre).
3. **Criação da aula (bootstrap T00 + T02)** — Testado, com sucesso na segunda tentativa. `/api/bootstrap-t00` e `/api/complete-lesson` (T02, modo "lesson") responderam 200 com um `item_package` SIM109 válido, contendo as duas experiências (fundação + conclusão/aplicação) para o item 1, incluindo `visual_trigger`.
4. **Comunicação com IA / resposta recebida** — Confirmado. Logs do servidor mostram chamada real ao provedor Gemini, resposta em ~4s, JSON estruturado e válido para o modo "lesson".
5. **Renderização de imagem** — Confirmado. `/api/visual-route` respondeu 200 e o app exibiu o chip de estado `T02_READY_SVG_RASTERIZED`, confirmando que o SVG do visual foi rasterizado com sucesso.
6. **Responder um item e avançar** — **Parcialmente testado — bloqueado.** A primeira resposta (item 1, Experiência 1, alternativa correta "A") foi processada corretamente: feedback "Correto. Você dominou este ponto." foi exibido, e o nível de confiança ("Tenho certeza") foi registrado e persistido no servidor (200 OK). A partir daí, o app trava permanentemente na tela "Preparando próximo passo" (botão desabilitado), sem avançar para a Experiência 2 do mesmo item, sem novas chamadas de rede, sem log de erro e sem qualquer mensagem ao usuário. Reproduzido de forma idêntica em duas rodadas completas e independentes (uma com texto de objetivo malformado, outra com texto limpo), afastando a hipótese de erro de digitação como causa. Aguardado 3+ minutos sem qualquer mudança.

## Causa raiz identificada (achado técnico, não corrigido)

Durante a investigação, adicionei um log de diagnóstico temporário (revertido depois, `git status` limpo) para capturar a resposta bruta da IA no endpoint `/api/warmup`, que também falhava (502, `AI_CONTRACT_INVALID`) em ambas as rodadas, 3 tentativas em cada.

Achado: a IA está retornando, para o modo **"warmup"**, um JSON no formato completo `item_package` (schema `SIM109_ITEM_PACKAGE_V1`, com `experience_1`/`experience_2`) — o mesmo formato usado pelo modo `"lesson"`. Só que o validador de contrato do servidor (`normalizeLessonJson` em `src/t02/complete-lesson-controller.js`), para qualquer modo que não seja `"lesson"`, espera um formato **achatado** diferente (`explanation`/`question`/`options`/`correct_answer` direto, não dentro de `item_package`). Por isso toda resposta de warmup é rejeitada como contrato inválido, mesmo sendo um JSON bem formado e semanticamente correto.

Essa falha de warmup, por si só, não impediu a criação da aula nem a resposta do item 1 (que usa o modo `"lesson"`, com o parser certo). Mas a trava em "Preparando próximo passo" logo após a primeira resposta é consistente com o mecanismo de pré-preparo de próxima janela (`PREPARE_READY_WINDOW`, visto em `lesson_answer_progress_controller.dart`/`bom_prepared_experience_coordinator.dart`) ficando marcado como `failed` e nunca sendo reexecutado nem exibindo erro ao usuário — não confirmei de forma definitiva que é o mesmo job do warmup falho, por limite de tempo desta sessão, mas os sintomas (momento exato da trava, ausência total de nova atividade de rede, nenhuma mensagem de erro) apontam fortemente nessa direção.

**Isto é um bug real, reprodutível e não é uma regressão desta sessão** (não alterei nenhum arquivo de `t02/` ou do fluxo de avanço nesta sessão de teste) — é um problema pré-existente descoberto agora, pela primeira vez, através deste teste ponta-a-ponta em device físico. Nenhuma correção foi aplicada; a investigação foi só até identificar a causa raiz, conforme sua instrução de parar ao encontrar um erro real.

## Estado do ambiente ao final

- Todo código de diagnóstico temporário foi revertido; `git status` limpo em `sim109-economic-final-app` e `sim109-economic-final-server`.
- Nenhum commit foi feito nesta sessão de teste (não havia correção a commitar).
- O servidor de teste standalone (porta 3012) foi encerrado.
- `main` de ambos os repositórios permanece no mesmo estado já confirmado no relatório anterior de verificação (hash inalterado).

## Conclusão

Login, onboarding completo, criação de aula, comunicação real com a IA (T00 e T02 modo lesson), e renderização de imagem — todos confirmados funcionando corretamente em device físico via Tailscale. O fluxo trava de forma reproduzível e silenciosa logo após a primeira resposta de item, impedindo o uso normal da aula além do primeiro item. Recomendo tratar isso como bloqueador antes de liberar a build para uso real, e decidir comigo os próximos passos: corrigir o contrato do modo warmup (alinhar parser ao formato que a IA já retorna, ou instruir a IA a retornar o formato achatado esperado) e/ou adicionar tratamento de erro visível quando o job de preparo da próxima janela falha, em vez de travar silenciosamente.
