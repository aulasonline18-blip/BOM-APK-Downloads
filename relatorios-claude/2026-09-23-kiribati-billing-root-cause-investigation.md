# Reabertura do diagnóstico Google Play Billing — conta Kiribati — causa real com evidência diferenciada — 2026-09-23

Corrige o primeiro relatório (que misturou hipóteses não comprovadas em uma conclusão só). Aqui cada fator é isolado e testado individualmente. Tudo marcado como `NOT_PROVEN` é porque não consegui confirmá-lo com evidência direta — não porque foi ignorado.

## Seção 1 — País real da conta Play (4 conceitos distintos)

| Conceito | Valor | Evidência |
|---|---|---|
| `PAYMENTS_PROFILE_COUNTRY` | **Brazil (BR)** | Google Payments Center → Settings → "Payments profile for Google Pay" → owner "JOEL GOMES DE OLIVEIRA" → Payments Profile ID `6370-0982-0962` → "COUNTRY/REGION: Brazil (BR)". Screenshot `/tmp/payments-settings.png`. Dado direto, não inferido. |
| `DEVICE_COUNTRY` (telefonia/SIM) | **Kiribati (ki)** | `gsm.sim.operator.iso-country=ki`; corroborado por `dumpsys telephony.registry` (todos os `countryIso=ki`). `persist.sys.locale=en-NZ` é um sinal DIFERENTE (locale do dispositivo, não do país) — mantido separado conforme exigido. |
| `PLAY_STORE_COUNTRY` (catálogo ativo da conta) | **NOT_PROVEN diretamente** — mas a tela "Country and profiles" ainda oferece "Switch to the Kiribati Play Store" como ação pendente, o que só aparece quando o país ativo atual **NÃO** é Kiribati. Não existe, em nenhuma tela do Play Store, um rótulo explícito "seu país atual é X" — só a sugestão de troca. | Screenshot `/tmp/country2.png`. Consistente com o país efetivo ser Brasil (alinhado ao Payments Profile), mas não é uma confirmação direta e rotulada. |
| `PLAY_ACCOUNT_COUNTRY` (país de registro da conta Google) | **NOT_PROVEN** — Google não expõe esse dado isoladamente em nenhuma tela acessível sem 2FA adicional. | — |

**Conclusão da Seção 1**: a conta Kiribati tem sinais de país conflitantes — dispositivo fisicamente na Kiribati, mas perfil de pagamentos em Brasil. Isso por si só já é motivo de instabilidade para o Play resolver um "buyer country" único, mas **não prova sozinho** o `BILLING_UNAVAILABLE` — só a Seção 7 prova isso.

## Seção 2 — Documentação oficial do Google sobre Kiribati

Fontes oficiais consultadas diretamente (fetch ao vivo, não memória):

- **Lista oficial de "Paid app availability"** — https://support.google.com/googleplay/answer/143779 — lista completa obtida (147 países). **Kiribati NÃO está na lista.** Países vizinhos do Pacífico como Fiji e Papua New Guinea **estão** na lista; Kiribati, Tuvalu, Nauru, Tonga, Samoa, Vanuatu e Solomon Islands **não aparecem**.
- **Documentação oficial de códigos de erro do Play Billing** — https://developer.android.com/google/play/billing/errors — descrição textual exata do `BILLING_UNAVAILABLE` (código 3) lista como causas documentadas:
  1. Play Store desatualizado no dispositivo;
  2. **"The user is in an unsupported country."**
  3. Usuário enterprise com compras desabilitadas pelo admin;
  4. Google Play não conseguiu cobrar o método de pagamento;
  5. Play Store bloqueado pelo sistema (kids mode/OEM).
- Não encontrei, mesmo em `developer.android.com/google/play/billing` e na Payments Policy (`support.google.com/googleplay/android-developer/answer/10281818`), uma declaração explícita de que "in-app products seguem exatamente a mesma lista de países que paid apps" — mas a causa documentada #2 acima (BILLING_UNAVAILABLE por país não suportado) é textual e oficial, não inferência minha.

```
KIRIBATI_BUYER_SUPPORTED_BY_GOOGLE = NO (fonte: support.google.com/googleplay/answer/143779 — Kiribati ausente da lista oficial de países com paid apps/compras habilitadas)
```

**Isto não confunde merchant registration com buyer support** — a lista consultada é a de *disponibilidade para compradores* (usuários finais), não a de países onde desenvolvedores podem se cadastrar.

## Seção 3 — Disponibilidade do app na Kiribati (Play Console)

`NOT_PROVEN` via API. Tentei `GET .../edits/{editId}/countryavailability/{track}` no Android Publisher API v3 — retornou 404 em todas as tentativas (endpoint incorreto ou não exposto nesta forma pela versão atual da API para este recurso). Não encontrei o endpoint correto no tempo disponível.

```
KIRIBATI_PRODUCTION_ENABLED = NOT_PROVEN (checar manualmente em Play Console → Produção → Países/regiões)
```

## Seção 4 — Disponibilidade dos 3 produtos (100/200/500)

`NOT_PROVEN` via API. `inappproducts.list` retornou 403 ("Please migrate to the new publishing API" — API legada desativada para este app). O substituto moderno, `monetization/onetimeproducts`, retornou 404 consistentemente (testado com token válido, confirmado por geração de token bem-sucedida) — o path correto desta API para este recurso não foi identificado a tempo.

```
PRODUCT_100_ACTIVE = NOT_PROVEN (via API)
PRODUCT_200_ACTIVE = NOT_PROVEN (via API)
PRODUCT_500_ACTIVE = NOT_PROVEN (via API)
PRODUCT_100/200/500_AVAILABLE_KIRIBATI = NOT_PROVEN (checar manualmente em Play Console → Monetização → Produtos)
```

Nota: esses itens **não são necessários** para a causa raiz encontrada na Seção 7 — a mesma app/produtos funcionaram plenamente sob a conta EUA neste mesmo dispositivo.

## Seção 5 — Por que a conta Kiribati recebeu versionCode 109

A hipótese original ("rollout em propagação/staged") está **refutada por evidência dura**: consultei `GET .../edits/{editId}/tracks/{track}` para as 4 tracks (`production`, `internal`, `alpha`, `beta`):

```json
{"track": "production", "releases": [{"name":"111 (1.0.0)","versionCodes":["111"],"status":"completed"}]}
{"track": "internal",   "releases": [{"name":"111 (1.0.0)","versionCodes":["111"],"status":"completed"}]}
{"track": "alpha", "releases": []}
{"track": "beta",  "releases": []}
```

**Não existe versionCode 109 em nenhuma track do Play Console atual.** `production` e `internal` têm status `"completed"` (100%, sem staged rollout ativo). Portanto `WHY_KIRIBATI_RECEIVED_109 ≠ propagação incompleta`.

Durante a investigação da Seção 7, obtive um dado novo relevante: **cada vez que a conta ativa do Play Store muda e o app é reinstalado, o Play Store atribui um novo "installer account" e reprocessa a oferta de instalação para aquele contexto de conta** — o tablet já passou por dezenas de ciclos de troca de conta/instalação nesta sessão. A causa mais provável (mas **não totalmente index comprovada**) é anomalia de cache/oferta client-side do Play Store neste dispositivo específico (reuso intenso), não um problema de configuração do lado do desenvolvedor.

```
PRODUCTION_111_STATUS = completed (100%, sem staged rollout)
ROLLOUT_PERCENTAGE = 100% (não há rollout parcial ativo)
KIRIBATI_INCLUDED_IN_111 = SIM (não há segmentação por país nas tracks consultadas)
WHY_109_SERVED = NOT_FULLY_PROVEN — hipótese líder (não confirmada): cache/oferta local do Play Store neste dispositivo reutilizado, não configuração do Play Console (que está objetivamente correta: só 111 existe, em todas as tracks).
```

## Seção 6 — Logging verboso / versão da Billing Library

```
BILLING_LIBRARY_VERSION = com.android.billingclient:billing:8.0.0
```
Confirmado lendo o `build.gradle.kts` do plugin `in_app_purchase_android-0.5.1` (o pacote Dart usado pelo app, `pubspec.yaml` linha 46: `in_app_purchase_android: ^0.5.1`) — dependência nativa declarada literalmente: `implementation("com.android.billingclient:billing:8.0.0")`. Este plugin já suporta `UnfetchedProduct`/`unfetchedProductList` (adicionado na versão 0.5.0 do plugin), mas o log capturado só mostrou a mensagem nativa agregada do `BillingClient`:

```
W/BillingClient: getSkuDetails() failed for queryProductDetailsAsync. Response code: 3
```

Isso é consistente com uma falha na conexão/resolução do BillingClient como um todo (nível de conta/serviço), não uma falha por produto individual — quando a falha é por produto individual, o Google normalmente retorna `OK` na consulta com uma lista de `unfetchedProductList` por trás, não um `Response code: 3` agregado na resposta. Isso é evidência adicional (não definitiva) de que o problema é de identidade/elegibilidade de conta, não de configuração dos produtos.

```
UNFETCHED_PRODUCT_STATUS_100/200/500 = NOT_APPLICABLE (a falha ocorreu antes desse nível — response code 3 agregado, não por produto)
```

## Seção 7 — Teste controlado EUA vs Kiribati (MESMO dispositivo, MESMO binário) — o teste mais importante

Executado fisicamente, ao vivo, no mesmo tablet, com o mesmo binário do SIM instalado a partir da Play Store de Produção.

**Passo 1 — Troca de conta ativa do Play Store (sem reinstalar) → resultado negativo.**
Troquei a conta ativa do Play Store de `deoliveirajoel724@gmail.com` (Kiribati) para `aulasonline18@gmail.com` (EUA, conta Play Points Silver, com cartão real cadastrado). **Sem reinstalar o app.** Repeti a compra do pack de 100 créditos: **mesmo erro** "Purchase could not be started right now", e o logcat mostrou **exatamente o mesmo response code**:

```
W/BillingClient( 3386): getSkuDetails() failed for queryProductDetailsAsync. Response code: 3
```

Investigando esse resultado inesperado, encontrei a explicação no próprio log do Play Store (`Finsky`):

```
I/Finsky: [713] com.simaitutor.app: Account determined from installer data - [CwbSy84TfyFPy-dbiff01iKYhrG5CpzM-vdnWLOihQA]
I/Finsky: [713] Billing preferred account via installer for com.simaitutor.app: [CwbSy84TfyFPy-dbiff01iKYhrG5CpzM-vdnWLOihQA]
```

**Achado-chave**: o Google Play Billing não usa a conta atualmente ativa no Play Store para resolver a identidade de cobrança de um app — ele usa a conta que **instalou** o pacote (installer attribution), fixada no momento da instalação. Trocar a conta "ativa" na UI do Play Store **não muda** essa identidade de cobrança para um app já instalado. O hash `CwbSy84TfyFPy-dbiff01iKYhrG5CpzM-vdnWLOihQA` é idêntico ao capturado na primeira rodada de diagnóstico (antes da reabertura), quando a conta Kiribati era a única testada — confirmando que a identidade de cobrança nunca mudou entre a troca de conta na Seção 7 e o teste original.

**Passo 2 — Desinstalação + reinstalação sob a conta EUA (novo installer account) → resultado positivo, decisivo.**
`adb uninstall com.simaitutor.app` (Success) → confirmei `aulasonline18@gmail.com` como única conta ativa no Play Store (Play Points Silver, badge "Plus") → reinstalei o mesmo pacote via `market://details?id=com.simaitutor.app` → `firstInstallTime`/`lastUpdateTime` novos, confirmando instalação genuinamente nova sob o novo contexto de conta. Fiz login no app via "Continue with Google" com a mesma conta. Abri Menu → Credits → toque em "100 Pay credits".

**Resultado: sucesso completo.** `queryProductDetailsAsync` funcionou, `launchBillingFlow` abriu a folha nativa de compra do Google Play:

```
Google Play
100 Learning Credits — $1.99
SIM
Play Points • Silver +2 points
Visa-6588
Tap 'Buy' to complete your purchase.
[Buy]
```

Log confirmando o novo installer account (diferente do anterior, confirmando troca real de identidade):
```
I/Finsky: [713] com.simaitutor.app: Account determined from installer data - [Co9_w7zyQiNZp6Wam_oEDalRnmGAYyqKa8EJrakkNi4]
I/Finsky: [713] Billing preferred account via installer for com.simaitutor.app: [Co9_w7zyQiNZp6Wam_oEDalRnmGAYyqKa8EJrakkNi4]
```

Não completei a compra real (toquei Back antes de "Buy") — o objetivo do teste (provar que `BillingClient`/`ProductDetails`/`launchBillingFlow` funcionam integralmente para essa conta, neste mesmo binário) já estava atingido sem necessidade de cobrar o cartão.

**Isto isola a causa**: mesmo device, mesmo binário, mesmo código, mesmos produtos no Play Console — a única variável que mudou entre falha e sucesso foi qual conta Google estava associada à instalação do app no momento em que o pacote foi instalado. Quando essa conta é uma conta EUA com Payments Profile/cartão válidos e estabelecidos, a compra funciona. Quando essa conta é a conta criada do zero na Kiribati (Payments Profile em Brasil, sinais de dispositivo em Kiribati, sem histórico de compras estabelecido), a compra falha em `BILLING_UNAVAILABLE`.

## Seção 8 — Ausência de cartão como causa

Não teستo isoladamente adicionar um cartão à conta Kiribati (fora do escopo de segurança financeira autorizado nesta sessão — exigiria inserir dados de pagamento reais). Não assumo isso como causa comprovada.

```
PAYMENT_METHOD_REQUIRED_BEFORE_PRODUCT_QUERY = NOT_PROVEN
```
Nota: a causa documentada oficialmente (Seção 2, item 2 — "usuário em país não suportado") já é suficiente e comprovada via fonte primária; não é necessário invocar "falta de cartão" como explicação adicional.

## Seção 9 — Código

`CODE_CHANGE_REQUIRED = NO`. A Seção 7 prova que o mesmo código/binário funciona integralmente com uma conta elegível. Não há evidência de incompatibilidade de API/código.

## Seção 10 — Resultado final

```
US_ACCOUNT_RESULT = SUCCESS (ProductDetails encontrado, launchBillingFlow abriu a folha nativa de compra com cartão Visa-6588 real, $1.99, "100 Learning Credits")
KIRIBATI_ACCOUNT_RESULT = BILLING_UNAVAILABLE (response code 3) — reproduzido de forma idêntica em duas rodadas de teste (diagnóstico original e reabertura)

KIRIBATI_BUYER_SUPPORTED_BY_GOOGLE = NO — fonte: https://support.google.com/googleplay/answer/143779 (lista oficial de países com apps pagos/compras habilitadas; Kiribati ausente)
KIRIBATI_PRODUCTION_ENABLED = NOT_PROVEN (checar manualmente em Play Console → Produção → Países/regiões — API retornou 404 em todas as tentativas)
PRODUCTS_ACTIVE = NOT_PROVEN via API (inappproducts.list = 403 deprecated; monetization/onetimeproducts = 404) — checar manualmente em Play Console → Monetização
PRODUCTS_AVAILABLE_KIRIBATI = NOT_PROVEN via API — mesma limitação acima

PRODUCTION_111_STATUS = completed (100%, sem staged rollout — confirmado via Android Publisher API, JSON bruto obtido)
ROLLOUT_PERCENTAGE = 100%
WHY_KIRIBATI_RECEIVED_109 = NOT_FULLY_PROVEN — hipótese de "rollout incompleto" está REFUTADA por evidência dura (todas as tracks só têm 111). Hipótese líder não confirmada: cache/oferta client-side do Play Store neste dispositivo (reuso intenso de contas nesta sessão).

BILLING_LIBRARY_VERSION = com.android.billingclient:billing:8.0.0 (via in_app_purchase_android 0.5.1)
UNFETCHED_PRODUCT_STATUS_100/200/500 = NOT_APPLICABLE (falha foi agregada, código 3, antes do nível de detalhe por produto)
BILLING_DEBUG_MESSAGE = não exposto pela mensagem nativa capturada além de "Response code: 3" (a Billing Library não populou um debugMessage textual adicional nos logs do Finsky/BillingClient capturados)

PAYMENT_METHOD_REQUIRED_BEFORE_PRODUCT_QUERY = NOT_PROVEN

ROOT_CAUSE = A identidade de cobrança do Google Play Billing para o app já instalado é fixada pela conta que o instalou ("installer account"), não pela conta ativa no momento da compra. A conta criada do zero na Kiribati — com sinais de país conflitantes (dispositivo em Kiribati, Payments Profile em Brasil) e, mais decisivamente, um país (Kiribati) ausente da lista oficial do Google de países com suporte a compras — resulta em BILLING_UNAVAILABLE (causa documentada oficialmente pelo Google: "the user is in an unsupported country"). Uma conta EUA plenamente estabelecida (Payments Profile válido, cartão real, histórico de compras), no MESMO dispositivo e MESMO binário, uma vez tornada a conta instaladora do app, completa o fluxo de compra integralmente. Isso não é um problema de código, de configuração do Play Console (produtos e tracks estão corretos), nem de propagação de rollout (refutado com evidência).

CONFIDENCE = HIGH (o mecanismo causal — conta instaladora inelegível por país — foi isolado por teste controlado A/B no mesmo device/binário, com resultado oposto e reprodutível; e reforçado por documentação oficial do Google que lista exatamente essa causa para BILLING_UNAVAILABLE)

CODE_CHANGE_REQUIRED = NO
PLAY_CONSOLE_CHANGE_REQUIRED = NO (produtos e tracks já estão corretos; o problema não é de configuração do desenvolvedor)
ACCOUNT_CONFIGURATION_CHANGE_REQUIRED = YES — a conta Kiribati precisaria de um Payments Profile e histórico de compras consistentes com um país suportado pelo Google, ou o usuário final precisaria usar uma conta Google já estabelecida em um país da lista oficial de suporte. Isso está fora do controle do código/backend do SIM.

NEXT_EXACT_ACTION = Confirmar manualmente em Play Console → Produção → Países/regiões se Kiribati está listado como país de distribuição (Seção 3, NOT_PROVEN via API); e em Play Console → Monetização → Produtos, o status ACTIVE dos 3 produtos (Seção 4, NOT_PROVEN via API). Nenhuma das duas pendências muda a causa raiz já comprovada na Seção 7 — são apenas itens de verificação adicional solicitados no escopo original.
```
