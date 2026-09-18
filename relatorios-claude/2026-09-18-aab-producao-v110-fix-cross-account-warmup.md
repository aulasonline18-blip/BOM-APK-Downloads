# AAB de produção gerado — versionCode 110 (fix vazamento entre contas)

Data: 2026-09-18

## Conteúdo desta build

Contém a correção do bug BLOCKER de vazamento de aquecimento entre contas
(commit `1a9cc08`), descrita em detalhe em
[2026-09-18-bug-cross-account-warmup-leak-e-lesson-ownership.md](2026-09-18-bug-cross-account-warmup-leak-e-lesson-ownership.md).
Nenhuma outra mudança de código nesta build além do bump de versão.

## Dados da build

- **App HEAD**: `eb2693373a2919b789cabb9757eac987bc030a86`
- **versionName/versionCode**: `1.0.0+110`
- **applicationId**: `com.simaitutor.app`
- **SIM_SERVER_URL**: `https://simaitutor.com`
- **SIM_AUTH_REDIRECT_URL**: `simaitutor://login-callback`
- **Assinado com**: `/root/.sim-android-signing/sim-public-github-20260726.jks`, alias `sim_upload` — **mesma chave usada no build 108/109**, fingerprint SHA-256 conferido e idêntico: `54:FB:04:AE:A6:7C:27:E0:D0:CF:F6:34:B1:39:C8:CE:58:BB:68:B3:C8:C9:24:A0:84:0A:F1:D1:1E:6D:E5:6A`
- **Arquivo**: `/root/sim-release-artifacts/sim109-v1.0.0+110-eb26933-app-release.aab`
- **SHA-256 do AAB**: `ae0d0c9d6d2e13896b12647443a1666b2e6067de01413897e2723f78deed4328`
- **Gerado em**: 2026-09-18, via `scripts/build-bom-production-aab.sh` (script oficial, sem modificação)

Verificado: manifesto do AAB confirma `versionName=1.0.0` e
`applicationId=com.simaitutor.app`; certificado de assinatura conferido via
`keytool -list -v` batendo com o fingerprint já registrado nas builds
anteriores.

## Servidor

Nenhum deploy de servidor foi necessário para este build — a correção é
inteiramente no app Flutter. O servidor de produção já está na versão que
contém a correção de `sim_state_owners` (deploy anterior, commit
`7bbc8fa`), sem relação com este fix.

## Pendências antes da promoção

- [ ] Teste no tablet físico do cenário adversarial completo (trocar contas
  sem fechar o app, repetir várias vezes) — ainda não feito, pois exige o
  dispositivo físico.
- Não fiz nenhuma tentativa de publicação no Play Console — aguardando sua
  validação manual, como de costume.
