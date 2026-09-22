# Kiribati x Google Play — a "lista" citada, e esclarecimento sobre o teste feito agora no tablet

Resumo direto para repassar a outro agente analisar.

## 1. Qual é a "lista" que eu mencionei

Fonte oficial do Google: **https://support.google.com/googleplay/answer/143779** ("Paid app availability" — países onde o Google Play permite compras pagas, incluindo compras dentro do app).

Essa página lista os países onde a Google Play habilita venda paga. Ao consultar essa lista na investigação anterior desta sessão, confirmei que os seguintes países do Pacífico **NÃO aparecem** nela:

- **Kiribati**
- Tuvalu
- Nauru
- Tonga
- Samoa
- Vanuatu
- Ilhas Salomão

Em contraste, outros países vizinhos do Pacífico **aparecem normalmente** na lista:
- Fiji
- Papua Nova Guiné

Ou seja: não é "todos os países pequenos do Pacífico estão de fora" — é uma lista específica e explícita do Google, e Kiribati está entre os países ausentes dela.

Documentação técnica complementar consultada: **https://developer.android.com/google/play/billing/errors** (documentação oficial do Play Billing Library), que lista textualmente, entre as causas documentadas do erro `BILLING_UNAVAILABLE` (código 3): *"the user is in an unsupported country"* — exatamente o código de erro observado nos testes com a conta de Kiribati.

**Importante sobre "deveria estar aberto pra todos os países"**: essa restrição não é uma configuração que o app ou o desenvolvedor define no Play Console (não existe um botão "adicionar Kiribati" que resolva isso, até onde já foi possível confirmar — isso ainda está sendo verificado a fundo em paralelo, ver seção 4). Pelo que a documentação oficial do Google indica, é uma limitação da própria infraestrutura de pagamentos do Google Play para aquele país — o Google simplesmente não processa compras pagas para compradores identificados como estando em Kiribati, independentemente do app.

## 2. Esclarecimento sobre o teste que você acabou de fazer no tablet

Você reportou que a tela de confirmação de pagamento apareceu (sem erro) agora há pouco. Eu vi a tela — de fato apareceu a folha real da Google Play: "100 Learning Credits — $1.99", cartão Mastercard-9319, botão "Buy".

Mas ao checar o log técnico (`logcat`) no exato momento, descobri o seguinte:

- O app foi reinstalado no tablet às `10:24:49` (hoje, 23/09).
- O log do Google Play (`Finsky`) mostra que a identidade de cobrança vinculada a essa instalação é o hash `Co9_w7zyQiNZp6Wam_oEDalRnmGAYyqKa8EJrakkNi4`.
- Esse hash é **exatamente o mesmo** capturado na investigação anterior quando a conta **aulasonline18 (EUA)** foi usada para reinstalar o app.
- Você confirmou agora que a conta ativa no Play Store no tablet é a **aulasonline18 (EUA)**.

**Conclusão**: essa compra funcionando agora é consistente com o que já sabíamos — a conta EUA funciona. Não é uma evidência nova sobre Kiribati. A identidade de cobrança do Google Play para um app já instalado é fixada pela conta que estava ativa no Play Store **no momento da instalação** (fenômeno já documentado na investigação anterior, seção "installer account attribution") — trocar a conta ativa no Play Store depois, sem desinstalar e reinstalar, não muda isso.

Para testar Kiribati de fato, é necessário: desinstalar o app completamente → trocar a conta ativa no Play Store para a de Kiribati (`deoliveirajoel724@gmail.com`) → reinstalar o app pela Play Store já com essa conta ativa → só então tentar a compra.

## 3. Resultado já confirmado do teste A/B anterior (mesmo tablet, mesmo binário versionCode 111)

| | Conta EUA (aulasonline18) | Conta Kiribati (deoliveirajoel724) |
|---|---|---|
| Hash de instalador | `Co9_w7zyQiNZp6Wam_oEDalRnmGAYyqKa8EJrakkNi4` | `CwbSy84TfyFPy-dbiff01iKYhrG5CpzM-vdnWLOihQA` |
| `queryProductDetailsAsync` | Sucesso | Falhou — `BillingResponseCode = 3 (BILLING_UNAVAILABLE)` |
| `launchBillingFlow` | Chamado, abriu folha real de compra | Nunca chamado |
| Resultado | Chegou até a tela de pagamento real | Erro antes de chegar à tela de pagamento |

## 4. O que ainda está em aberto (investigação em andamento em paralelo)

Um agente está checando agora, via API do Android Publisher (Google Play Console programático):
- Se existe algum endpoint atual que mostre explicitamente a lista de "países/regiões de distribuição" configurada no Play Console para este app, e se Kiribati está incluído ou excluído dela.
- Se os 3 produtos de crédito (100/200/500) têm alguma configuração regional específica que exclua Kiribati.
- Se, de fato, isso é uma limitação de plataforma do Google (não corrigível do nosso lado) ou uma configuração no Play Console que pode ser ajustada.

Assim que esse resultado chegar, ele será integrado a este relatório com uma conclusão definitiva (resolvido, ou confirmado como limitação de plataforma do Google sem correção possível do nosso lado).
