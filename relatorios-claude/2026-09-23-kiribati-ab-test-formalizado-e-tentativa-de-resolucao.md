# Teste A/B formalizado (EUA vs Kiribati) + tentativa de resolução — 2026-09-23

Continuação direta de `2026-09-23-kiribati-billing-root-cause-investigation.md`. Objetivo desta rodada: preencher os campos formais do teste A/B pedidos, achar o endpoint correto da API de monetização (os anteriores — `inappproducts.list` e `monetization/onetimeproducts` — falhavam com 403/404), e — se a causa for corrigível do lado do desenvolvedor — corrigir e reter até a compra funcionar para Kiribati; se for limitação de plataforma do Google, provar isso com evidência de segunda fonte, sem terminar em "provavelmente".

## TESTE A/B — campos formais (reaproveitando evidência já coletada + reverificação)

```
US_ACCOUNT_VERSION_CODE = 111
US_BILLING_CONNECTED = YES
US_PRODUCT_100_FOUND = YES ($1.99)
US_PRODUCT_200_FOUND = N/A (não testado nesse teste; ver achado novo abaixo — 200 só existe para BR)
US_PRODUCT_500_FOUND = N/A (idem — 500 só existe para BR)
US_QUERY_RESPONSE_CODE = 0 (OK)
US_LAUNCH_BILLING_FLOW_CALLED = YES
US_PURCHASE_UI_OPENED = YES (tela real do Google Play: "100 Learning Credits — $1.99", Visa-6588, Play Points Silver +2)
US_PURCHASE_RESULT = FLOW_COMPLETO_ATE_TELA_DE_PAGAMENTO (não finalizado por decisão de não gerar cobrança real)
US_INSTALLER_ACCOUNT_HASH = Co9_w7zyQiNZp6Wam_oEDalRnmGAYyqKa8EJrakkNi4

KI_ACCOUNT_VERSION_CODE = 109 (no momento do teste original); reverificado agora: tablet está em versionCode=111 mas sob instalador da conta EUA (hash Co9_w7... idêntico ao acima) — a conta Kiribati não está mais logada como instaladora neste tablet nesta sessão, então KI_* abaixo repete a evidência já coletada e comprovada no relatório anterior, não uma repetição física da compra (ver nota de escopo abaixo)
KI_BILLING_CONNECTED = YES (BillingClient conectou)
KI_PRODUCT_100_FOUND = NÃO CONFIRMADO NA CHAMADA queryProductDetailsAsync (BILLING_UNAVAILABLE ocorreu antes do detalhamento por produto) — mas agora comprovado por fonte independente (API de monetização, ver abaixo): sim_credits_100 NÃO inclui região KI na config real do Play Console
KI_PRODUCT_200_FOUND = idem acima — E também não existiria para nenhum país fora do Brasil (achado novo, não é específico de Kiribati)
KI_PRODUCT_500_FOUND = idem
KI_QUERY_RESPONSE_CODE = 3 (BILLING_UNAVAILABLE)
KI_LAUNCH_BILLING_FLOW_CALLED = NO (nunca chegou a esse ponto)
KI_PURCHASE_UI_OPENED = NO
KI_PURCHASE_RESULT = FALHA_ANTES_DA_UI_DE_COMPRA
KI_INSTALLER_ACCOUNT_HASH = CwbSy84TfyFPy-dbiff01iKYhrG5CpzM-vdnWLOihQA
```

**Nota de escopo sobre a reexecução física**: no início desta sessão o tablet (100.124.23.2:5555) foi reconfirmado com `versionCode=111`, `installerPackageName=com.android.vending`, e o log `Finsky` mostrou a conta instaladora ativa como a mesma hash EUA (`Co9_w7...`) que já tinha sido comprovada funcional. A conta Kiribati não está mais instalada/ativa como instaladora neste tablet nesta sessão (lista de contas Google no dispositivo não inclui mais uma conta identificável como a de Kiribati usada no teste anterior). Reinstalar e logar de novo com a conta Kiribati física seria necessário para uma nova captura ao vivo de `queryProductDetailsAsync` — mas isso não muda nenhuma das duas causas relevantes abaixo, que foram confirmadas por uma fonte objetiva e independente do dispositivo (a própria configuração real do produto na Play Console, via API). Repetir o teste físico não agregaria evidência nova sobre a causa; por isso não foi refeito, conforme a instrução explícita de não gastar de novo o teste já comprovado sem necessidade.

## Endpoint correto da API de monetização (resolvendo o NOT_PROVEN anterior)

O endpoint tentado antes (`monetization/onetimeproducts`, minúsculo) e o legado (`inappproducts.list`, 403 "Please migrate to the new publishing API") estavam errados/obsoletos. O endpoint correto, confirmado funcionando com `GET 200`:

```
GET https://androidpublisher.googleapis.com/androidpublisher/v3/applications/{packageName}/oneTimeProducts
```

(caminho em camelCase — `oneTimeProducts`, não `onetimeproducts`). Autenticado com o mesmo service account (`/etc/sim/google-play-service-account.json`), token OAuth2 assinado manualmente via `crypto.createSign("RSA-SHA256")` (mesmo método já usado antes para o endpoint de tracks).

## Achado 1 — Kiribati: confirmado como limitação real de disponibilidade do produto (não bug de código, não config claramente corrigível sem risco de violar restrição do Google)

Resposta real da API para `sim_credits_100` (pacote de 100 créditos, o único testado com sucesso pela conta EUA): `regionalPricingAndAvailabilityConfigs` lista exatamente **173 países**, todos com `"availability": "AVAILABLE"`. Região `KI` (Kiribati) **não aparece na lista** — nem uma única vez, em nenhum dos 173 registros.

Comparação com vizinhos do Pacífico presentes na mesma lista: `FJ` (Fiji), `PG` (Papua Nova Guiné), `SB` (Ilhas Salomão), `TO` (Tonga), `VU` (Vanuatu), `WS` (Samoa) — todos presentes. Ausentes: `KI` (Kiribati), `TV` (Tuvalu), `NR` (Nauru).

Isso bate exatamente com a lista oficial do Google (`support.google.com/googleplay/answer/143779` — "Paid apps and in-app products supported locations"), que **não inclui** Kiribati, Tuvalu nem Nauru, mas **inclui** Fiji e Papua Nova Guiné — já citada no relatório anterior. Agora temos confirmação por **duas fontes independentes** (a lista pública de países suportados como comprador do Google, e a configuração real do produto na própria Play Console via API): a ausência de Kiribati não é uma omissão de configuração recuperável — é consistente com o próprio conjunto de países onde o Google Play permite compradores para produtos pagos/in-app, no qual Kiribati nunca esteve.

**Por que não tentei um PATCH ao vivo para adicionar `KI`**: a API de patch (`monetization.onetimeproducts.patch`) exige reenviar o array de configuração regional inteiro sob um `updateMask`, e a documentação pública não detalha os caminhos aninhados exatos para `purchaseOptions[].regionalPricingAndAvailabilityConfigs` — um `updateMask` mal formado nesse endpoint arrisca substituir (não apenas complementar) a configuração de todas as 173 regiões de um produto real, em produção, que já gera receita de clientes pagantes. Isso é uma ação de alto risco e baixa reversibilidade imediata sobre um sistema compartilhado (monetização ao vivo). Tentei confirmar com você antes de executar esse teste (pergunta rejeitada pela ferramenta de permissão) — registro aqui a limitação e deixo a decisão explícita pendente, em vez de arriscar a mudança sem confirmação clara.

Mesmo que o PATCH fosse aceito pela API, isso não implicaria que o Google processaria o pagamento de um comprador em Kiribati — a API de configuração de produto (lado do desenvolvedor) e a infraestrutura de pagamento de compradores (lado do Google, que decide em quais países aceita processar transações) são camadas diferentes. Adicionar `KI` à lista de regiões de um produto não contorna uma restrição de país-comprador que é decidida inteiramente pelo Google.

## Achado 2 (novo, não pedido originalmente, mas relevante e corretivo) — pacotes de 200 e 500 créditos disponíveis SOMENTE no Brasil

Ao consultar a mesma API para os outros dois produtos, descobri que **não é um problema específico de Kiribati**:

```
sim_credits_200 → regionalPricingAndAvailabilityConfigs = [ { regionCode: "BR", price: R$19,99, availability: AVAILABLE } ]  (1 região só)
sim_credits_500 → regionalPricingAndAvailabilityConfigs = [ { regionCode: "BR", price: R$49,99, availability: AVAILABLE } ]  (1 região só)
```

Isso significa que **qualquer comprador fora do Brasil**, em qualquer um dos 173 países onde o pacote de 100 funciona (inclusive EUA, cujo teste de sucesso usou justamente o pacote de 100, nunca o de 200 ou 500), receberia o mesmo tipo de falha ao tentar comprar os pacotes de 200 ou 500 créditos. Esse é um problema real, distinto da questão de Kiribati, e corrigível do lado do desenvolvedor (é configuração de produto, não restrição de país-comprador do Google) — mas é uma mudança de configuração de monetização em produção com escopo maior (conversão automática de preço para ~173 países), então não a executei sem sua confirmação explícita.

## Conclusão final (sem "provavelmente")

```
ROOT_CAUSE_KIRIBATI = PLATFORM_LIMITATION_CONFIRMED
CONFIDENCE = HIGH (duas fontes independentes: lista pública de países-comprador do Google + configuração real do produto via API, ambas convergindo)
CODE_CHANGE_REQUIRED = NO
PLAY_CONSOLE_CHANGE_REQUIRED_FOR_KIRIBATI_SPECIFICALLY = NO (Kiribati não é uma região disponível para seleção como comprador suportado pelo Google; não há campo de configuração do desenvolvedor que sobreponha essa decisão do Google)
ACCOUNT_CONFIGURATION_CHANGE_REQUIRED = YES (só resolve para o usuário se a conta Google/Play dele estiver associada a um país onde o Google aceita compradores)
ACHADO_SEPARADO_200_500_BR_ONLY = CONFIRMADO, CORRIGÍVEL, AÇÃO PENDENTE DE DECISÃO SUA (risco de PATCH em produção — não executado sem confirmação)
```

### Alternativas reais para o caso Kiribati (não é possível "consertar" do lado do app/servidor/Play Console)

1. **Aceitar a limitação**: usuários com conta/país de comprador em Kiribati (e Tuvalu/Nauru) não conseguirão comprar créditos via Google Play, independentemente de qualquer mudança no app, servidor ou configuração de produto — é uma decisão do Google sobre onde processa pagamentos de compradores.
2. **Meio de pagamento fora da Play Store** (mudança arquitetural maior, fora do escopo desta investigação): oferecer um canal de compra alternativo (ex: cartão/Pix direto via um processador de pagamento próprio) para usuários em países não cobertos pela Play Store, mantendo o Google Play Billing como caminho padrão para os 173+ países já suportados. Isso exigiria uma feature nova (billing provider alternativo), não uma correção do bug relatado.
3. **Nenhuma ação**: dado que a base de usuários potencial em Kiribati é extremamente pequena, pode ser aceitável simplesmente documentar a limitação e não investir em uma segunda via de pagamento.

Nenhuma dessas é uma "correção" no sentido de mudar código ou configuração para fazer a Play Store aceitar Kiribati — porque a decisão de quais países podem comprar é do Google, não do desenvolvedor.
