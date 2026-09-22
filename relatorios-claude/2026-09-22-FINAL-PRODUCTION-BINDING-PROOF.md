# FINAL_PRODUCTION_BINDING — prova de reinstalação limpa e vínculo real com produção — 2026-09-22

Fecha a última incerteza antes do teste interno da Google Play: confirma, com reinstalação a partir do arquivo-fonte hash-verificado, que o binário que roda no tablet é exatamente o artefato final registrado e que ele conversa de fato com o droplet real de produção.

## Procedimento (todo executado diretamente por mim, coordenador, não delegado)

1. **Evidência do estado anterior preservada** em `evidencias/2026-09-22-final-production-binding/` antes de qualquer alteração: screenshot, `dumpsys package`, dump de UI, e cópia do APK extraído do dispositivo antes da reinstalação.
2. **Hash do arquivo-fonte confirmado ANTES de instalar**:
   `sha256sum SIM-v110-6e3b360-w10-integration-final.apk` → `fcaeb540ba822d5fe888ea878a9a099b5a41e899efd41f2ce085b287d7745199` — idêntico ao hash final registrado no relatório `2026-09-22-W10-INTEGRACAO-GLOBAL-FINAL.md`.
3. **Desinstalação completa**: `adb uninstall com.simaitutor.app` → `Success`.
4. **Reinstalação exata** do mesmo arquivo (`adb install -r SIM-v110-6e3b360-w10-integration-final.apk`) → `Success`. `firstInstallTime`/`lastUpdateTime` novos (`2026-09-23 05:55:35` fuso tablet +12 = `2026-09-22 17:55:35Z`), confirmando instalação genuinamente nova, não uma atualização sobre estado antigo — a tela de login voltou ao estado inicial ("Sign in to start"), sem sessão anterior.
5. **Login real**: criada conta de QA nova (`qa-finalbinding-20260922@sim-internal-test.invalid`) via Supabase Admin API (senha não registrada em nenhum lugar versionado). Login realizado fisicamente no app reinstalado, com toque orientado por `uiautomator dump` a cada etapa (o layout do formulário desloca depois de digitar o e-mail — confirmado de novo; usar sempre a posição pós-digitação, nunca coordenada memorizada).
6. **Ação real disparada**: toque em "Sign in" às `2026-09-22T18:03:22Z`.
7. **Confirmação no log do droplet real**: `POST /api/credits/me` chegou ao servidor real às `2026-09-22T18:03:24.877Z` (HTTP 200, 764ms) — ~2,5s depois do toque, consistente com o fluxo de autenticação real seguido de consulta de saldo.
8. **Reconfirmação do servidor no exato momento desta prova**: `readlink /opt/sim/current` → `bb7c56fc56a45093661e37c7359fe44ba14365dd`, `systemctl is-active bom-api.service` → `active`, `health` local e via `https://simaitutor.com` → `200`.

## Resultado

```
SERVER_RUNTIME_SHA = bb7c56fc56a45093661e37c7359fe44ba14365dd
APP_BUILD = 6e3b360
APK_SOURCE_SHA256_CONFIRMED_BEFORE_INSTALL = fcaeb540ba822d5fe888ea878a9a099b5a41e899efd41f2ce085b287d7745199
APK_POINTS_TO_PRODUCTION = YES
PHYSICAL_TEST_TRAFFIC_CONFIRMED_ON_REAL_SERVER = YES
FINAL_PRODUCTION_BINDING = PASS
```

**Nota de honestidade metodológica**: a extração pós-instalação do `base.apk` via `pm path` para comparação de hash byte-a-byte não foi concluída (timeout de rede na extração de um arquivo de ~76MB pela conexão adb wireless até o tablet). A prova de vínculo aqui não depende disso — depende de (a) hash do arquivo-fonte confirmado imediatamente antes do `adb install`, (b) desinstalação completa comprovada (estado de app zerado, nova sessão exigida), e (c) tráfego real, ao vivo, gerado por uma ação física minha e observado no log do servidor real dentro de segundos. Isso é uma cadeia de custódia equivalente, não uma comparação bit-a-bit do binário instalado.

Evidências (screenshots, dumps, dumpsys) preservadas em `evidencias/2026-09-22-final-production-binding/`.
