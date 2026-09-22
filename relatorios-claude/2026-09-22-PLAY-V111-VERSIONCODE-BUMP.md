# Google Play Internal Testing — versionCode 111 — 2026-09-22

Google Play rejeitou o upload do release funcional final (`SIM-v110-6e3b360-w10-integration-final.aab`) com "Version code 110 has already been used". Sem problema conhecido no artefato — só o número de build já tinha sido consumido.

**Correção de contexto**: o SHA completo do APP citado como baseline funcional em instruções anteriores desta sessão (`6e3b360346488f6af91abd0e49971ad37ca56160`) não existe neste repositório — o SHA real do commit `6e3b360` é `6e3b36011bbc4dfd484e0fdedf7af14a4f303c4d` (o SERVER SHA `bb7c56fc56a45093661e37c7359fe44ba14365dd` estava correto). Confirmado via `git cat-file`/`git log` antes de qualquer alteração, conforme instruído ("usar o GitHub/repositórios locais como autoridade").

## Verificação do estado antes de editar

- `CURRENT_APP_MAIN` = `6e3b36011bbc4dfd484e0fdedf7af14a4f303c4d` — idêntico ao baseline funcional certificado, `git status` limpo, nenhum commit posterior.
- `CURRENT_SERVER_MAIN` = `bb7c56fc56a45093661e37c7359fe44ba14365dd` — idêntico ao esperado, `git status` limpo. Servidor não precisou de nenhuma alteração.

## Autoridade do versionCode

- `VERSION_AUTHORITY_FILE` = `pubspec.yaml` (linha `version: 1.0.0+110`). `android/local.properties` também tem `flutter.versionCode=110`, mas esse arquivo é gitignored e regenerado automaticamente pelo Flutter a partir do pubspec — não é fonte de verdade.
- `CURRENT_VERSION_NAME` = `1.0.0`
- `CURRENT_VERSION_CODE` = `110`

## Alteração única

`version: 1.0.0+110` → `version: 1.0.0+111` em `pubspec.yaml`. Diff completo do repositório: exatamente essa linha, uma inserção/remoção. Confirmado explicitamente com `git diff 6e3b360..HEAD -- . ':!pubspec.yaml'` → vazio (nenhuma outra diferença em nenhum arquivo).

`flutter analyze --no-pub`: inicialmente reportou ~35 mil erros — causa raiz era `.dart_tool`/pub desatualizado neste checkout (pacotes como `flutter_test`/`package:image` não resolvidos), não relacionado à edição. `flutter pub get` resolveu; análise limpa depois ("No issues found!").

**Commit**: `c1c5de99a8cf82cc5081b1944812a2f9b0a373dc` (`chore(release): bump Android version code to 111`), pushado para `main` do BOM.

```
FUNCTIONAL_BASELINE_APP_SHA = 6e3b36011bbc4dfd484e0fdedf7af14a4f303c4d
PLAY_V111_RELEASE_SHA = c1c5de99a8cf82cc5081b1944812a2f9b0a373dc
FUNCTIONAL_DELTA = VERSION_CODE_ONLY
SERVER_PRODUCTION_SHA = bb7c56fc56a45093661e37c7359fe44ba14365dd
```

## Build e verificação do bundle

AAB gerado com `scripts/build-bom-production-aab.sh` (mesmos `--dart-define`s de produção: `SIM_SERVER_URL=https://simaitutor.com`, `SIM_AUTH_REDIRECT_URL=simaitutor://login-callback`, `applicationId=com.simaitutor.app`). APK correspondente também gerado (mesmo commit/config) só para permitir inspeção confiável do manifesto com `aapt2` (bundletool não estava disponível standalone com todas as dependências no classpath desta máquina).

`aapt2 dump badging` no APK gerado (não confiando no nome do arquivo):
```
package: name='com.simaitutor.app' versionCode='111' versionName='1.0.0'
```

## Artefatos finais

```
AAB_FILENAME = SIM-v111-c1c5de9-play-internal.aab
AAB_SHA256   = 2b1b6580674b79b3b1678ba47ee47b3c1cbec834c2c8ff778d8a3cb7f5e87d78
AAB_SIZE     = 73775946 bytes (~70.4 MB)
APK_FILENAME = SIM-v111-c1c5de9-play-internal.apk
APK_SHA256   = 5ae1c533309ad7e5264f2cc22db2273c105b74e12ec9c5b7c02a52f23b97e27c
```

## Publicação

GitHub Release nova (repositório público `aulasonline18-blip/BOM-APK-Downloads`), sem sobrescrever o release `v110-6e3b360-final` anterior (preservado como histórico):

https://github.com/aulasonline18-blip/BOM-APK-Downloads/releases/tag/v111-play-internal

## Resumo final

```
PLAY_V111_APP_SHA = c1c5de99a8cf82cc5081b1944812a2f9b0a373dc
FUNCTIONAL_BASELINE_APP_SHA = 6e3b36011bbc4dfd484e0fdedf7af14a4f303c4d
FUNCTIONAL_DELTA = VERSION_CODE_ONLY
SERVER_PRODUCTION_SHA = bb7c56fc56a45093661e37c7359fe44ba14365dd
VERSION_NAME = 1.0.0
VERSION_CODE = 111
AAB_FILENAME = SIM-v111-c1c5de9-play-internal.aab
AAB_SHA256 = 2b1b6580674b79b3b1678ba47ee47b3c1cbec834c2c8ff778d8a3cb7f5e87d78
AAB_SIZE = 73775946 bytes
AAB_DIRECT_DOWNLOAD = https://github.com/aulasonline18-blip/BOM-APK-Downloads/releases/download/v111-play-internal/SIM-v111-c1c5de9-play-internal.aab
GITHUB_RELEASE = https://github.com/aulasonline18-blip/BOM-APK-Downloads/releases/tag/v111-play-internal
```

Nenhuma promoção para produção na Play Console foi feita. Nenhum reteste no tablet foi refeito (próximo teste real será via instalação pelo próprio Google Play Internal Testing, conforme instruído).
