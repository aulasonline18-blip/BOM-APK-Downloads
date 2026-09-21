# 2026-09-21 — RWR-001 clean-install rehydration fix

## Escopo

Bug de produção em instalação limpa:

- endpoint: `POST /api/complete-lesson`
- ambiente: `https://simaitutor.com`
- sintoma anterior: HTTP 409 `CREDIT_OPERATION_REQUIRES_RECONCILIATION`
- aula de prova: `cyber-15cy53v`
- marcador de prova: `M0002`

## Causa raiz

O bloqueio vinha de uma divisão entre duas identidades econômicas:

- a chave comercial do item, usada pela reserva/captura de crédito do aluno;
- a chave de material/provider, usada para recuperar o resultado T02 durável.

Na reinstalação limpa, o app perdia o material local. A chamada nova podia não encontrar o resultado pela chave de material atual, mas encontrava uma operação comercial já capturada/aceita. O servidor então bloqueava com `CREDIT_OPERATION_REQUIRES_RECONCILIATION`, corretamente impedindo nova cobrança cega, mas sem convergir para a recuperação do material.

## Correção

Servidor `Servidor-BOM`, commit:

`42d541dafab17a99fc8d55c1470d8abcb24ea8ce`

Mensagem:

`fix(cost): recover captured T02 material on clean reinstall`

Arquivos principais:

- `src/ai/ai-cost-protection-gate.js`
- `test/rwr001_economic_saga_baseline_contract.test.js`

Comportamento novo:

1. Antes de reservar novo crédito, o cost gate consulta a operação comercial do item.
2. Se ela já tem resultado durável aceito, o servidor reusa esse resultado.
3. Se a operação comercial está terminal/capturada mas sem referência de material, o servidor executa uma recuperação absorvida de resultado, sem nova reserva/captura do aluno.
4. O resultado recuperado é persistido e anexado à operação comercial.
5. Tentativas seguintes fazem replay, sem nova cobrança.

## Provas locais do servidor

Executado em `/root/Servidor-BOM`:

- `node --check src/ai/ai-cost-protection-gate.js` — PASS
- `node test/rwr001_economic_saga_baseline_contract.test.js` — PASS
- `node test/phase_b2_durable_ledger_contract.test.js` — PASS
- `git diff --check` — PASS
- `npm run check:server-minimum` — PASS
- `npm run check:protected` com autorização SIM109 — PASS
- `npm audit --omit=dev` — 0 vulnerabilidades
- `npm test` — PASS, 139/139 arquivos

## Deploy de produção

Droplet real atualizado para:

`/opt/sim/releases/42d541dafab17a99fc8d55c1470d8abcb24ea8ce`

Serviço:

`bom-api.service`

Rollback anterior preservado:

`/opt/sim/releases/5f7e0cf11c9ba3a773e5f9360dea8d180683ca12`

Health depois do deploy:

- `https://simaitutor.com/api/health` — HTTP 200
- `https://www.simaitutor.com/api/health` — HTTP 200

## Prova de produção

Após o deploy, a mesma aula que antes bloqueava foi chamada duas vezes em produção:

- primeira chamada `POST /api/complete-lesson` — HTTP 200, schema `SIM109_ITEM_PACKAGE_V1`
- segunda chamada igual — HTTP 200, schema `SIM109_ITEM_PACKAGE_V1`

Logs sanitizados após o deploy:

- primeira chamada `/api/complete-lesson` terminou com HTTP 200;
- segunda chamada teve `duplicateSuppressed: true`;
- nenhum novo `CREDIT_OPERATION_REQUIRES_RECONCILIATION` apareceu no recorte pós-deploy.

## Prova física no Samsung SM-X216B

APK instalado:

- package `com.simaitutor.app`
- versionCode `110`
- dispositivo `SM-X216B`

Estado inicial observado no tablet:

- tela da aula mostrava `Failed to generate content`;
- botão visível: `Try again`;
- esse era o estado deixado pelo bloqueio anterior de reidratação.

Ação física:

- tocar `Try again` no app instalado;
- aguardar a resposta de produção.

Resultado físico:

- a tela saiu do erro;
- o item `2 / 20` voltou a aparecer;
- conteúdo pedagógico, visual board e alternativas ficaram visíveis;
- não apareceu novo HTTP 409 no app.

Logs de produção da ação física:

- `POST /api/complete-lesson` — HTTP 200;
- `AI_FINANCIAL_OPERATION` com `duplicateSuppressed: true`;
- `POST /api/student-state/persist` — HTTP 200;
- chamadas de `visual-route` — HTTP 200;
- nenhum `CREDIT_OPERATION_REQUIRES_RECONCILIATION` no recorte da ação física.

## Veredito

`RWR-001 CLEAN-INSTALL REHYDRATION BLOCKER RESOLVED FOR THE REPRODUCED PRODUCTION CASE AND PHYSICALLY RETESTED ON SM-X216B`

Limites:

- não é prova global de todos os caminhos RWR-001;
- não substitui os demais itens do checklist físico;
- não encerra Amparo, Dúvida, Revisão, Recuperação, Finalização, Placement, CG1, Billing ou demais validações pendentes.

Próxima ação operacional:

continuar o checklist físico em produção real, começando pelos itens que estavam bloqueados pela reidratação do material remoto.
