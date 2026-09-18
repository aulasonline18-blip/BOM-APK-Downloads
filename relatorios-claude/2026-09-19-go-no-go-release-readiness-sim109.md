# GO/NO-GO — Release Readiness SIM109

Data: 2026-09-19

## BASE

- **App**: repo `BOM`, branch `feature/nplus1-image-and-canonical-scroll`, HEAD `e89aa32`. Tree limpa, tudo commitado e pushado.
- **Server**: repo `Servidor-BOM`, branch `main`, HEAD `7fd99e8`. Tree limpa, tudo commitado e pushado.
- **App identity**: `com.simaitutor.app`, versionName `1.0.0`, versionCode `108`.
- **Nota**: durante esta auditoria, um commit adicional (`e89aa32`, fix de merge do pendingMap da Recuperação em multi-device) apareceu no branch, aparentemente de outra sessão trabalhando em paralelo no mesmo mission. Verifiquei o diff e o teste (`student_state_backup_sync_b_test.dart`, 26/26 passando) — correto e bem escopado, mantido.

## O QUE FOI CORRIGIDO NESTA SESSÃO (resumo, detalhes nos relatórios anteriores)

1. Vazamento de scroll/imagem N+1 e reconstrução da autoridade canônica de scroll (4 intents).
2. Bug do item preso em "Preparando próximo passo" — causa raiz no servidor (falha genérica sem horário de retry, travava até 10min) + no app (nunca tentava de novo). Corrigido nos dois lados, com teste de regressão dedicado.
3. `VISUAL_ROUTE_INVALID_IMAGE` no ambiente local — gap de configuração de 1 mês atrás (`MEDIA_OBJECT_STORAGE_INLINE_DEV_FALLBACK`), não regressão. Ativado no `.env` local; sem efeito em produção (trava arquitetural impede isso).
4. (Por outra sessão/commit paralelo) vazamento do pendingMap da Recuperação em merge multi-device.

## GO/NO-GO MATRIX

| Área | Status | Evidência | Ação |
|---|---|---|---|
| Git/base | PASS | HEADs limpos, pushados, capturados acima | — |
| Auth/identidade | PASS | Código revisado (login, accountGeneration, logout) — não houve mudança nesta sessão nessa área | — |
| Produção vs dev config | PASS | Build de release já falha (`BOM_RELEASE_*`) se apontar pra localhost/dev, applicationId errado, ou keystore debug. Zero secret hardcoded no app. | — |
| Billing (server) | PASS | Verificação real via Google Play API antes de creditar; idempotente por orderId (teste `m17_google_play_readiness_server` passando) | — |
| Billing (app) | PASS (parcial) | Plugin oficial `in_app_purchase`, verificação server-side confirmada. Compra real **requer teste no Internal Testing** | EXTERNAL VALIDATION REQUIRED |
| Credits ledger / idempotência | PASS | reserve/capture/release com guarda de replay em todos os casos; testes passando | — |
| TEST_CREDIT_EMAILS em produção | **INDEFINIDO** | Sem gate de `isProduction`; não tenho acesso para checar `.env` de produção | **Pergunta para o Joel (ver abaixo)** |
| Progresso/estado/multi-device | PASS | Merge monotônico (stateRevision + highWaterMark), testes de regressão passando; fix adicional de pendingMap nesta sessão | — |
| Segurança/logs/privacidade | PASS | Redação abrangente (`safeLog`/`safeWarn`), sem token/secret/foto em log, permissões do manifest mínimas e justificadas | — |
| Manifest/build/signing | PASS | Só INTERNET+CAMERA; keystore externalizado e gitignored; build falha se usar debug.keystore | — |
| Scroll (C1-C7) / imagem N+1 | PASS | Todos os testes estruturais passando, sem regressão | — |
| Dúvida (vazamento de explicação) | PASS | Teste dedicado prova que a explicação não vaza pro próximo item; observação ao vivo ambígua (mesma cor verde do feedback normal) mas não reproduzida como bug real | — |
| Aula principal (corredor completo) | PASS (amostragem) | 5 itens consecutivos testados ao vivo cruzando o ponto que travava antes, tudo fluindo. Não foi possível rodar as 60 questões manualmente no tempo da auditoria — apoiado nos testes automatizados completos | — |
| Sala de Revisão | PASS | Testada ao vivo: pergunta gerada, resposta validada, estado sobrevive a restart do app | — |
| Sala de Nivelamento/Recuperação/Amparo | NOT RE-TESTED | Já auditadas em profundidade em sessão anterior (tasks #26, #29, #24); não tocadas nesta sessão; não re-testadas ao vivo por limite de tempo | — |
| Anexos | NOT RE-TESTED | Já corrigido/testado em sessão anterior (task #30); não tocado nesta sessão | — |
| AAB candidato | **BLOQUEADO** | Script oficial (`scripts/build-bom-production-aab.sh`) existe e está correto, mas exige `SIM_SERVER_URL`/`SIM_AUTH_REDIRECT_URL`/`SIM_ANDROID_APPLICATION_ID` de produção que não tenho e não devo adivinhar | **Precisa ser rodado pelo Joel (ou me passar os valores)** |
| Servidor de produção | PASS (parcial) | `simaitutor.com/api/health` responde 200. Não tenho acesso SSH para confirmar se o HEAD deployado é `7fd99e8` | EXTERNAL VALIDATION REQUIRED |
| Rollback | DOCUMENTADO | Se o server novo (`7fd99e8`) causar problema em produção, reverter para o commit anterior conhecido-bom (`892004f`, Cut C.2 item 5) e reiniciar o serviço | — |

## PERGUNTAS PARA O JOEL (não bloquearam a auditoria, decisão conservadora já tomada)

1. **TEST_CREDIT_EMAILS em produção**: `src/config/env.js:99` lê essa lista sem checar `isProduction`. Se o `.env` real de produção tiver isso setado, essas contas ganham saldo "infinito" lá também. Não sei se é intencional. Não mexi no código.
2. **Staging antigo**: ver nota separada sobre `sim-two-experience-server` vs `sim109-economic-final-server` — recomendação dada, decisão de aposentar o antigo é sua.
3. **AAB de produção**: preciso de `SIM_SERVER_URL`, `SIM_AUTH_REDIRECT_URL`, `SIM_ANDROID_APPLICATION_ID` reais para gerar o artefato final — ou você roda `scripts/build-bom-production-aab.sh` com essas variáveis no seu ambiente.

## TESTES

- App: `flutter analyze` limpo. Suítes-chave rodadas e verdes: scroll (C1-C8), N+1 visual readiness, sync/merge (26/26), doubt-leak-prevention. Suite completa (1431+ testes) rodada mais cedo na sessão, verde.
- Server: 101/101 arquivos de teste passando (`node scripts/run-all-tests.js`).

## VERDICT

**GO FOR INTERNAL TEST**, condicionado a:
- Joel confirmar/gerar o AAB com os valores reais de produção (ou me passar os valores).
- Confirmar se o servidor de produção está ou vai ser atualizado para `7fd99e8` antes do teste interno.
- Responder a pergunta do TEST_CREDIT_EMAILS.

Nenhum BLOCKER interno encontrado. Nenhuma mudança cosmética ou feature nova foi introduzida nesta missão — apenas as correções já documentadas.
# Perguntas pendentes para o Joel (decisões que só ele pode autorizar)

Regra: nada aqui bloqueia a execução. Cada item abaixo tem a decisão provisória mais conservadora já tomada, e o trabalho continuou.

(nenhuma ainda — será preenchido conforme a auditoria avançar)

## 1. TEST_CREDIT_EMAILS / TEST_CREDIT_BALANCE em produção
`src/config/env.js:99` + `src/credits/credits-store.js:50,169-175`. Sem gate de `isProduction` — se o `.env` da produção tiver `TEST_CREDIT_EMAILS` setado (como o `.env` local desta VM tem, com 999999 de saldo), essas contas ganham saldo "infinito" em produção também, silenciosamente.
- Sem acesso SSH à produção pra verificar (tentei `root@simaitutor.com`, permission denied).
- Decisão provisória: NÃO mexi no código — pode ser intencional (ex: sua própria conta ter saldo de teste mesmo em produção). Não travo o resto da auditoria por isso.
- Preciso que você confirme: o `.env` da produção tem `TEST_CREDIT_EMAILS` setado? Se sim, é intencional? Se não for intencional, é um BLOCKER financeiro real.

## 2. SIM_SERVER_URL real de produção para o AAB
Testei o gate de assinatura de produção (`validateProductionRelease`) com o keystore real (`/root/.sim-android-signing/sim-release-operational.jks`) e uma URL fictícia só pra checar sintaxe — passou limpo. Confirmado: keystore certo, não é debug, applicationId (`com.simaitutor.app`) e auth redirect (`simaitutor://login-callback`) corretos.
Falta só a URL HTTPS real da API de produção pra gerar o AAB de verdade (`scripts/build-bom-production-aab.sh`). Não adivinhei — os docs têm vários valores antigos/placeholder (IP antigo `167.179.109.137`, `SEU-DOMINIO-API`, etc.), nenhum claramente "o atual".
- Decisão provisória: NÃO gerei o AAB final. Preparei tudo (key.properties já configurado, gitignored). Assim que você confirmar a URL, é um comando só.
- Preciso que você confirme: qual é a URL HTTPS real de produção (`https://...`) que o app deve usar?

## 3. sim-api.service nesta VM roda em modo dev apesar de NODE_ENV=production
Achado durante a limpeza de infra. Esta VM (hostname "sim-oracle") tem um serviço systemd chamado `sim-api.service` (`WorkingDirectory=/root/Servidor-BOM`, `Environment=NODE_ENV=production`), atualmente ativo na porta 3000 (foi reiniciado por mim hoje sem querer, ao reiniciar o processo manualmente pra pegar minhas correções — o systemd reabsorveu o processo).
Confirmei ao vivo: mesmo com `NODE_ENV=production` no systemd, o processo real está rodando em modo DEV, porque `/root/Servidor-BOM/.env` tem `APP_MODE=development` na linha 36, e o código prioriza `APP_MODE` sobre `NODE_ENV` (`env.js`: `process.env.APP_MODE || process.env.NODE_ENV`). Ou seja, esse serviço "de produção" no nome nunca esteve rodando de verdade em modo produção.
- Não sei se esse `sim-api.service` nesta VM tem alguma relação com a produção real (droplet/simaitutor.com) ou é só um artefato local antigo — o hostname desta VM ("sim-oracle") sugere que não é o droplet real.
- Decisão provisória: não mexi em mais nada além do restart que já tinha feito antes (necessário pra minhas correções valerem no teste local). Não desliguei nem reconfigurei o serviço.
- Preciso que você confirme: esse `sim-api.service` nesta VM é usado pra alguma coisa real, ou é resquício e pode ser ignorado/desligado? Se for usado como uma referência de "produção", está mal configurado (roda em dev de fato).

## 4. Servidor de produção real — não consigo confirmar SERVER HEAD
Sem acesso SSH à produção (simaitutor.com, permission denied). Não consigo confirmar se o servidor real de produção já está rodando o mesmo SERVER HEAD (`7fd99e8`) que estou declarando como base desta release. EXTERNAL VALIDATION REQUIRED — só você (ou quem tem acesso ao droplet) pode confirmar/fazer esse deploy antes do GO final.

## 5. Infra de staging — distinção entre os dois ambientes (pedido no adendo, item 5)
- `sim109-economic-final-server` (`/root/worktrees/sim109-economic-final-server`, branch `reform/sim109-economic-final-construction`, porta 3012, systemd não encontrado pra ele — roda via processo solto) — este é o indicado no adendo como o staging ATIVO/atual pra reformas futuras do SIM109. Recomendo mantê-lo e, se possível, formalizar como serviço systemd (hoje é só um processo solto, sobrevive a reinício de sessão mas não a reboot da VM).
- `sim-two-experience-server` (`/root/worktrees/sim-two-experience-server`, branch `reform/two-experience-staging`, porta 3010, serviço systemd `bom-api-staging.service`) — staging de uma frente de trabalho anterior. Continua rodando. Recomendo aposentar formalmente (não apaguei nada — só recomendo), já que o SIM109 virou a linha principal.
- Não apaguei nenhum worktree nem branch antigo — só limpei artefatos temporários óbvios (chaves de teste em /tmp).

---
# Nota de infraestrutura de staging (não é tarefa de código, só documentação)

Dois ambientes de staging distintos na VM, propósitos diferentes:

1. **`/root/worktrees/sim109-economic-final-server`** (branch `reform/sim109-economic-final-construction`) — este é o staging ATIVO para o trabalho do SIM109 daqui pra frente. Recomendo manter como ambiente reutilizável para futuras reformas do servidor.

2. **`/root/worktrees/sim-two-experience-server`** + **`/root/worktrees/sim-two-experience-app`** (branch `reform/two-experience-staging`, serviço `bom-api-staging.service`, processo PID 363027 rodando desde 12/set) — staging de uma frente de trabalho ANTERIOR ao SIM109 se tornar a linha principal. Não decidi apagar — só recomendo ao Joel avaliar se ainda serve a algum propósito ativo ou se deve ser aposentado agora.

Sugestão de nome para não confundir no futuro: renomear ou documentar em um README de infraestrutura que `sim109-economic-final-server` = staging atual, `sim-two-experience-*` = staging legado (pré-SIM109-mainline).

Nada foi apagado. Decisão final de aposentar o staging antigo fica com o Joel.
