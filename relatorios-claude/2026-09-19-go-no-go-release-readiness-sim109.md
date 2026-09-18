# GO/NO-GO — Release Readiness SIM109

Data: 2026-09-19

## Nota de transparência

Esta versão substitui uma revisão anterior deste mesmo arquivo (commit `d2ff804`) que foi escrita e publicada por um subagente meu que recebeu uma tarefa estreita (auditoria de segurança/logs) mas, por ter herdado todo o contexto da missão, acabou refazendo a missão inteira por conta própria e sobrescrevendo este relatório sem minha revisão — inclusive atribuindo incorretamente o commit `e89aa32` a "outra sessão trabalhando em paralelo" (na verdade fui eu, no fio principal, enquanto esse subagente rodava em background). Não houve nenhum commit de código adicional além do que já estava documentado; o único efeito colateral real foi este arquivo de relatório. Esta versão é a autoritativa.

## BASE

- **App**: repo `BOM`, branch `feature/nplus1-image-and-canonical-scroll`, HEAD `e89aa32`. Fast-forward limpo à frente de `origin/main` (14 commits à frente, 0 atrás). Tree limpa, tudo commitado e pushado.
- **Server**: repo `Servidor-BOM`, branch `main`, HEAD `7fd99e8`. Tree limpa, tudo commitado e pushado.
- **App identity**: `com.simaitutor.app`, versionName/versionCode `1.0.0+108`.

## BLOCKERS ENCONTRADOS E CORRIGIDOS

1. **Item de Recuperação podia sumir silenciosamente em sync multi-dispositivo** (BLOCKER — classe "Recovery não bloqueia conclusão quando deveria").
   `lib/sim/state/student_learning_state.dart`: o merge de estado pegava o `auxRooms` (onde mora a fila pendente da Recuperação) inteiro do lado que "vencesse" por progresso de aula, sem relação com o conteúdo pendente em si. Um item pendente registrado no dispositivo com progresso mais baixo podia ser descartado ao sincronizar com um dispositivo mais avançado.
   Corrigido: união item-a-item da `pendingMap` por `(marker, layer)`, preferindo a entrada mais recente (`lastUpdatedAt`). Teste de regressão criado e confirmado que falha sem a correção. Commit `e89aa32`.

2. **Preparação do próximo item podia travar por até 10 minutos** (BLOCKER, já corrigido antes desta missão de auditoria — incluído aqui pelo registro).
   Servidor: falha genérica de IA (não rate-limit) ficava sem horário de retry definido, travando o registro de falha até o TTL cheio (10min). App: nada tentava de novo depois da primeira falha.
   Corrigido nos dois lados. Commits `7fd99e8` (servidor) e `89ca488` (app). Testado ao vivo no tablet: o item que travava antes (item 11) avançou normalmente em múltiplas repetições, inclusive cruzando esse ponto até o item 14.

3. **Canal de imagem pedagógica quebrado no ambiente local** (config de dev ausente, não regressão de código).
   O mecanismo de fallback para dev (`MEDIA_OBJECT_STORAGE_INLINE_DEV_FALLBACK`) já existe no código desde 24/ago; não estava habilitado no `.env` local desta VM. Ativado e testado ao vivo — imagens voltaram a carregar. Mudança apenas em `.env` local, não versionado, sem efeito em produção.

## GO/NO-GO MATRIX

| Área | Status | Evidência | Ação |
|---|---|---|---|
| Git/base (app+server) | PASS | HEADs acima, trees limpas, tudo pushado | — |
| Build/signing (gate) | PASS | `validateProductionRelease` rodado com o keystore real de produção; aprovou applicationId, callback e assinatura (não-debug) | — |
| Config produção vs dev | PASS com 1 ressalva | Nenhum segredo hardcoded; `isProduction` fail-safe confirmado nos padrões corretos (DURABLE_LEDGER_MODE, MEDIA_OBJECT_STORAGE_MODE). Ressalva: `TEST_CREDIT_EMAILS` sem gate de produção | Confirmar com Joel (pendência 1) |
| Auth/identidade de conta | PASS | Escopo por lessonLocalId/accountId confirmado, sem cache genérico não-escopado | — |
| Billing (Google Play) | PASS | Integração real (`in_app_purchase`, não mock), sequência de confirmação correta (credita antes de confirmar no Play), sem log de token. 1 MAJOR não-bloqueante: `restorePurchases` só dispara ao abrir a tela de compra, não no boot do app | Registrado para depois, não bloqueia |
| Credits ledger / idempotência | PASS | Bônus único, compra idempotente por token/orderId, reserve/capture/release sem duplicar, zero-saldo bloqueia antes da IA nas 5 salas. O retry automático adicionado nesta sessão não introduz cobrança dupla (capture ocorre 1x fora do retry interno) | — |
| T02 / lição principal | PASS | Testado ao vivo até o item 14 sem travar; scroll, imagem, indicadores e avanço funcionando | — |
| Imagem / N+1 | PASS | Prefetch com `promoteToCurrent:false` preservado, sem cobrança duplicada (auditoria de código); canal de imagem restaurado localmente | — |
| Progresso/estado | PASS (após fix) | Merge monotônico confirmado para progresso/eventos/truth; pendingMap corrigido nesta missão | — |
| Sync multi-device | PASS (após fix) | Ver blocker #1 | — |
| Dúvida | PASS | Vazamento de explicação entre itens já corrigido e commitado antes desta missão (`bd89546`), confirmado ancestral do HEAD atual | — |
| Scroll/navegação (app todo) | PASS | Única tela com scroll programático é a aula (já coberta pelos testes C1-C8); demais telas só têm controllers passivos, nenhuma tela sem saída encontrada | — |
| Logs/privacidade | PASS | Nenhum log de payload completo, token, ou foto em base64 encontrado nas áreas de maior mudança (Dúvida, créditos) | — |
| Manifest Android | PASS | Só INTERNET e CAMERA, ambas justificadas; nenhuma permissão excedente | — |
| Revisão / Recuperação (lógica) / Amparo / Nivelamento / Anexos | NOT RE-VERIFIED AO VIVO NESTA PASSADA | Auditados em profundidade em missões anteriores desta mesma sessão (tasks #24, #26-30); sem mudança de código desde então exceto o fix de merge (blocker #1, coberto por teste dedicado); suíte automatizada completa (1432 testes) verde | — |
| Caminho de atualização / instalação via Play / compra real | EXTERNAL VALIDATION REQUIRED | Não testável sem publicar no Play Internal Testing | Pendente Play |
| Servidor de produção real (deploy) | EXTERNAL VALIDATION REQUIRED | Sem acesso SSH à produção (`root@simaitutor.com`: permission denied) | Pendente Joel |

## TESTES

**APP**: `flutter analyze` limpo. `flutter test` completo: **1432/1432 passando** (rodado do zero após a correção do merge de auxRooms).

**SERVER**: suíte completa: **101/101 arquivos passando** (rodado duas vezes, incluindo o novo teste de regressão do retry).

## RELEASE

- versionName/versionCode: `1.0.0+108`, applicationId `com.simaitutor.app`.
- AAB de produção: **não gerado nesta missão** — falta apenas a URL HTTPS real da API de produção (pendência 2). Todo o resto do pipeline de assinatura (`scripts/build-bom-production-aab.sh`, keystore real) já foi validado.

## PENDÊNCIAS PARA O JOEL (nada travou a auditoria; decisão mais conservadora já tomada em cada uma)

1. `TEST_CREDIT_EMAILS`/`TEST_CREDIT_BALANCE` sem gate de produção (`src/config/env.js:99`) — confirmar se o `.env` real de produção tem isso setado e se é intencional.
2. URL HTTPS real da API de produção, para gerar o AAB final.
3. `sim-api.service` nesta VM (porta 3000, hostname `sim-oracle`) declara `NODE_ENV=production` no systemd mas roda em modo dev de fato, porque `/root/Servidor-BOM/.env` tem `APP_MODE=development` e o código prioriza essa variável. Não sei se esse serviço tem alguma relação com a produção real ou é resquício local.
4. Confirmar se a produção real já está no SERVER HEAD `7fd99e8` — sem acesso SSH para verificar.
5. Infra de staging: `sim109-economic-final-server` (porta 3012) recomendado como staging ativo para reformas futuras do SIM109; `sim-two-experience-server` (porta 3010, serviço `bom-api-staging.service`) recomendado para aposentadoria formal — decisão final é sua. Nenhum worktree ou branch foi apagado.

## VERDICT

**GO FOR INTERNAL TEST** — condicionado a resolver as pendências 1, 2 e 4 antes de submeter ao Play Console. Nenhuma delas é um bloqueio de código; são confirmações operacionais que só o Joel pode dar.

Nenhum BLOCKER de código permanece em aberto. Os dois blockers reais encontrados nesta sessão (perda de pending em sync multi-device, e travamento de preparação de item) foram corrigidos, testados e commitados.

Não declarar GO FOR PRODUCTION antes da validação real do Internal Testing e da compra de créditos via Google Play.
