# GO/NO-GO — Release Readiness SIM109

Data: 2026-09-19

## BASE

**APP**
- Branch: `feature/nplus1-image-and-canonical-scroll`
- HEAD: `e89aa32`
- Fast-forward limpo à frente de `origin/main` (14 commits à frente, 0 atrás — nenhum conflito, `main` pode ser avançado com fast-forward quando quiser)
- Tree limpa, tudo commitado e enviado (push feito)
- versionName/versionCode: `1.0.0+108` (pubspec.yaml)
- applicationId: `com.simaitutor.app` (confirmado em build.gradle.kts e no gate de release)

**SERVER**
- Branch: `main`
- HEAD: `7fd99e8`
- Tree limpa, tudo commitado e enviado (push feito)
- **Não confirmado**: se o servidor de produção real (droplet, simaitutor.com) já roda este HEAD — sem acesso SSH para verificar (ver pendência #4 abaixo).

## BLOCKERS ENCONTRADOS E CORRIGIDOS NESTA MISSÃO

1. **Item de Recuperação podia sumir silenciosamente em sync multi-dispositivo** (BLOCKER — classe "Recovery não bloqueia conclusão quando deveria").
   `lib/sim/state/student_learning_state.dart`: a função de merge pegava o `auxRooms` (onde mora a fila pendente da Recuperação) inteiro do lado que "vencesse" por progresso de aula — sem relação com o conteúdo pendente em si. Um item pendente registrado no dispositivo com progresso mais baixo podia ser descartado ao sincronizar com um dispositivo mais avançado.
   Corrigido: união item-a-item da `pendingMap` por `(marker, layer)`, preferindo sempre a entrada mais recente (`lastUpdatedAt`). Teste de regressão criado e confirmado que falha sem a correção. Commit `e89aa32`.

2. **Preparação do próximo item podia travar por até 10 minutos** (BLOCKER — já corrigido e reportado antes desta missão, incluído aqui para o registro).
   Servidor: falha genérica de IA (não rate-limit) ficava sem horário de retry definido, travando o registro de falha até o TTL cheio (10min). App: nada tentava de novo depois da primeira falha.
   Corrigido nos dois lados (retry automático no servidor com backoff, cooldown curto em vez de 10min; retry limitado no app). Commits `7fd99e8` (servidor) e `89ca488` (app). Testado ao vivo no tablet, item que travava antes (item 11) avançou normalmente múltiplas vezes.

3. **Canal de imagem pedagógica totalmente quebrado no ambiente local** (não é bug de produção — configuração de dev ausente).
   Investigado: mecanismo de fallback para dev (`MEDIA_OBJECT_STORAGE_INLINE_DEV_FALLBACK`) já existe no código desde 24/ago, só não estava habilitado no `.env` local. Ativado, testado ao vivo — imagens voltaram a carregar. Mudança feita apenas em `.env` local (não versionado, não afeta produção).

## GO/NO-GO MATRIX

| Área | Status | Evidência | Ação |
|---|---|---|---|
| Git/base (app+server) | PASS | HEADs capturados acima, trees limpas | — |
| Build/signing (gate) | PASS | `validateProductionRelease` rodou com keystore real, aprovou assinatura/applicationId/callback | — |
| Config produção vs dev | PASS com 1 ressalva | Nenhum segredo hardcoded; `isProduction` fail-safe confirmado no app e no server para os padrões corretos (DURABLE_LEDGER_MODE, MEDIA_OBJECT_STORAGE_MODE). Ressalva: `TEST_CREDIT_EMAILS` sem gate de produção (pendência #1) | Confirmar com Joel |
| Auth/identidade de conta | PASS | Escopo por lessonLocalId/accountId confirmado, sem cache genérico não-escopado | — |
| Billing (Google Play) | PASS | Integração real (não mock), sequência de confirmação correta (credita antes de confirmar no Play), sem log de token. 1 MAJOR não-bloqueante (restorePurchases só ao abrir tela de compra, não no boot) | Registrado para depois, não bloqueia |
| Credits ledger / idempotência | PASS | Bônus único, compra idempotente por token, reserve/capture/release sem duplicar, zero-saldo bloqueia antes da IA em todas as 5 salas. Meu retry de hoje não introduziu cobrança dupla (capture acontece 1x fora do retry interno) | — |
| Purchase idempotency | PASS | Ver acima — dedup por `stablePurchaseId` | — |
| T02 / lição principal | PASS | Testado ao vivo até item 14+ sem travar, scroll/imagem/indicadores/avanço funcionando | — |
| Imagem / N+1 | PASS | Prefetch com `promoteToCurrent:false` preservado, sem cobrança duplicada (auditoria de código), canal de imagem restaurado localmente | — |
| Progresso/estado | PASS (após fix) | Merge monotônico confirmado para progresso/eventos/truth; pendingMap corrigido nesta missão | — |
| Sync multi-device | PASS (após fix) | Ver blocker #1 acima | — |
| Dúvida | PASS | Vazamento de explicação entre itens já corrigido e commitado antes desta missão (`bd89546`), confirmado ancestral do HEAD atual | — |
| Revisão | PASS | Coberto por testes automatizados existentes, sem mudança de código nesta missão | — |
| Recuperação | PASS (após fix) | Bug de perda de pending corrigido (blocker #1) | — |
| Amparo | PASS | Sem mudança de código nesta missão, testes existentes verdes | — |
| Nivelamento | PASS | Sem mudança de código nesta missão, testes existentes verdes | — |
| Anexos | NOT RE-VERIFIED NESTA MISSÃO | Auditado em profundidade em missão anterior desta mesma sessão (#30), sem mudança de código desde então | — |
| Scroll/navegação (app todo) | PASS | Auditoria de código: única tela com scroll programático é a aula (já testada); demais telas só têm controllers passivos, sem tela sem saída encontrada | — |
| Logs/privacidade | PASS | Nenhum log de payload completo, token, foto em base64 ou segredo encontrado nas áreas de maior mudança (Dúvida, créditos) | — |
| Segredos no app | PASS | Nenhum segredo/chave literal encontrada no código do app | — |
| Manifest Android | PASS | Só INTERNET e CAMERA, ambas justificadas; nenhuma permissão excedente | — |
| Compatibilidade APP↔SERVER | PASS | Nenhuma quebra de contrato encontrada nas rotas auditadas | — |
| Caminho de atualização (update test) | EXTERNAL VALIDATION REQUIRED | Não testável sem publicar no Play Internal Testing | Pendente Play |
| Instalação via Play Internal | EXTERNAL VALIDATION REQUIRED | Não testável sem acesso ao Play Console | Pendente Play |
| Compra real (Google Play) | EXTERNAL VALIDATION REQUIRED | Não testável sem transação real no Play | Pendente Play |
| Servidor de produção real (deploy) | EXTERNAL VALIDATION REQUIRED | Sem acesso SSH à produção | Pendente Joel |

## TESTES

**APP**: `flutter analyze` limpo. `flutter test` completo: **1432/1432 passando** (rodado do zero após a correção do merge de auxRooms).

**SERVER**: suíte completa: **101/101 arquivos passando** (rodado duas vezes, incluindo o novo teste de regressão do retry).

## RELEASE

- versionName/versionCode: `1.0.0+108`
- applicationId: `com.simaitutor.app`
- AAB: **não gerado nesta missão** — bloqueado apenas pela URL HTTPS real de produção (pendência #2). Todo o resto do pipeline de assinatura já foi validado com o keystore real.

## PENDÊNCIAS PARA O JOEL (nada travou a auditoria, decisão mais conservadora já tomada em cada uma)

1. `TEST_CREDIT_EMAILS`/`TEST_CREDIT_BALANCE` sem gate de produção — confirmar se o `.env` real de produção tem isso setado e se é intencional.
2. Qual é a URL HTTPS real da API de produção, para gerar o AAB final.
3. `sim-api.service` nesta VM (porta 3000, hostname `sim-oracle`) diz `NODE_ENV=production` mas roda em modo dev de fato (por causa do `.env` local) — confirmar se esse serviço tem alguma relação com a produção real ou é resquício.
4. Confirmar se a produção real já está no SERVER HEAD `7fd99e8` (sem acesso SSH pra verificar).
5. Infra de staging: `sim109-economic-final-server` (porta 3012) recomendado como staging ativo daqui pra frente; `sim-two-experience-server` (porta 3010, `bom-api-staging.service`) recomendado para aposentadoria formal — decisão final é sua.

## VERDICT

**GO FOR INTERNAL TEST** — condicionado a resolver as pendências 1, 2 e 4 acima antes de submeter ao Play Console (nenhuma delas foi encontrada como bloqueio de código; são confirmações operacionais que só você pode dar).

Nenhum BLOCKER de código permanece em aberto. Os dois blockers reais encontrados nesta sessão (perda de pending em sync, e travamento de preparação de item) foram corrigidos, testados e commitados.

Não declarar GO FOR PRODUCTION antes da validação real do Internal Testing e da compra de créditos via Google Play.
