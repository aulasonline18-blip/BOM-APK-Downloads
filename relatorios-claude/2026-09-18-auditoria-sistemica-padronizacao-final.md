# Auditoria Sistêmica de Engenharia — BOM + Servidor-BOM

Execução: 2026-09-18/19. Este relatório está em construção incremental nesta sessão — seções são preenchidas conforme a evidência é coletada e verificada. Nenhuma afirmação de conformidade é feita sem evidência de código/log/teste citada inline.

## 1. Identidade da execução

- **APP** — repo `/root/BOM` (canônico), `main`. HEAD ao final desta auditoria: **`190138d`** (inclui, em ordem: `1a9cc08` fix do vazamento de aquecimento entre contas, `eb26933` bump de versionCode para 110, `da62a73` log de diagnóstico do erro genérico, `5533627` `allowBackup=false`, `190138d` guarda de staleness em `speakAuxRoomContent`). Push confirmado para `origin/main` (GitHub `aulasonline18-blip/BOM`).
- **SERVER** — repo `/root/Servidor-BOM`, `main`. HEAD ao final desta auditoria: **`8ca0e91`** (inclui `7bbc8fa` fix do `sim_state_owners`, já em produção antes desta auditoria, + `8ca0e91` log de falhas de transição do ledger durável, desta sessão). Push confirmado para `origin/main` (GitHub `aulasonline18-blip/Servidor-BOM`). Suite completa do servidor rodada após a mudança: **102/102 arquivos de teste passando**.
- Ambiente de execução: VM de desenvolvimento, sem acesso root ao Play Console; acesso SSH de produção ao droplet `168.144.255.169` (`sim-api-sgp1`) usado para consultar logs/Supabase quando necessário.

## 2. Metodologia

Esta auditoria seguiu o princípio de **não alucinar desvio**: cada achado abaixo cita arquivo:linha e, quando aplicável, uma referência normativa (documentação oficial Flutter/Dart, Android Developers, Google Play Billing, OWASP MASVS, RFC/HTTP, ou o próprio contrato interno já estabelecido pelo SIM). Rótulos de confiança usados: **PROVEN** (lido o código, comportamento inequívoco), **HIGH CONFIDENCE** (evidência forte mas não 100% exaustiva), **SUSPECTED** (indício, não confirmado), **NOT VERIFIED** (fora do alcance desta passada).

Parte da investigação foi paralelizada em 4 sub-auditorias independentes (agentes de pesquisa, somente leitura, sem edição de arquivos) cobrindo: (1) billing/créditos/Redis/segredos no servidor, (2) state management/lifecycle/async no app Flutter, (3) retry/timeout/idempotência em ambos os repositórios, (4) build Android/segurança móvel/dependências. Os resultados dessas sub-auditorias estão integrados nas seções correspondentes abaixo, identificados como tal.

---

## 3. LOGIN / LOGOUT / TROCA DE CONTA (achado P0 já corrigido)

### 3.1 Padrão correto
Logout deve produzir o mesmo efeito de limpeza que uma reinstalação para todo estado *account-scoped*: nenhum dado, em memória ou em disco, pertencente à conta anterior pode ser lido, exibido ou reenviado ao servidor depois da troca — sem depender de o usuário fechar o app manualmente.

### 3.2 Como o SIM resolve hoje
`LabSession` (`lib/features/session/lab_session.dart`) é um singleton de vida longa, composto por vários subcontroladores. Existe um hook `_onAccountChanged(previousUserId, currentUserId)`, disparado por `AuthSession.applySupabaseSession()` (`lib/session/auth_session.dart:81-83`) sempre que `userId` muda — **este é o padrão certo** (comparação explícita de identidade antes/depois, não um timer nem uma heurística). O handler já fazia, antes desta sessão: invalidar operações de runtime em voo (`_invalidateRuntimeOpenOperation`, um cancellation-token clássico — `_runtimeOpenToken = Object()` capturado antes do `await` e comparado via `identical()` depois, `lab_session.dart:1328-1380`), invalidar operações de Amparo pendentes, resetar `_runtimeController`, resetar `simOrganismProvider`, limpar `lessonUiState.lessonLocalId`.

### 3.3 Desvio exato encontrado (P0, BLOCKER)
`_onAccountChanged` **não limpava os campos do fluxo de aquecimento** (`warmupLesson`, `warmupError`, `warmupSelectedAnswer`, `warmupWaitingForOfficialLesson`, mapa `_warmupFlights` de gerações em voo). Esses campos vivem soltos na instância (`lib/features/session/lab_session_warmup_flows.dart`) e só eram zerados quando o app inteiro era recriado (restart). Resultado observado ao vivo por Joel: logout de conta A (com aquecimento pronto) → login conta B → tela de aquecimento mostrava o conteúdo da conta A.

**Prova (PROVEN):** reproduzido em `test/account_switch_state_isolation_test.dart` (novo). O teste falha (`Expected: null, Actual: <Instance of 'SimWarmupLesson'>`) quando a correção é removida, e passa com ela presente — cobre tanto aquecimento já pronto quanto uma geração ainda em voo no momento da troca.

### 3.4 Correção aplicada
`LabSessionWarmupController.resetForAccountChange()` (novo método), chamado dentro do `_onAccountChanged` já existente. Commit `1a9cc08`, já em `main` do BOM, incluído no AAB `1.0.0+110`. Suite completa (1434 testes) e `flutter analyze` limpos após a mudança.

### 3.5 Efeito colateral já materializado em produção (P1, correlacionado, não é bug novo)
Este vazamento, ANTES da correção, já havia gravado um dado corrompido *durável* (não apenas em memória): uma linha em `sim_ownership_records` (tabela Postgres/Supabase que arbitra posse de `lessonLocalId` via `bom_durable_assert_ownership`, usada por `/api/bootstrap-t00`, `/api/warmup`, `/api/complete-lesson`) ficou permanentemente associada à conta `aulasonline18` para o recurso `cyber-1r2yb4g`, e a conta `exponencial` acabou com uma linha `student_states` própria referenciando esse MESMO `lessonLocalId` (via `/api/student-state/persist`, que **não** passa por `assertRequestResourceOwners` — ver seção 5.3). Quando `exponencial` tentou reabrir essa aula, `/api/student-state/get` corretamente rejeitou com 403 `RESOURCE_OWNER_MISMATCH`, e `/api/complete-lesson` com 409 `CREDIT_OPERATION_REQUIRES_RECONCILIATION` — isso é o que produziu a mensagem genérica que o usuário reportou como "Servidor indisponível, tente novamente" (ver seção 8).

**Verificado ao vivo (PROVEN):** consulta direta à RPC `bom_durable_assert_ownership` em produção, com a chave service-role, confirmou `ownership_mismatch` para o userId de `exponencial` e `owned: true` para o userId de `aulasonline18`, para o mesmo `resource_key`.

**Consequência prática:** o app já se autorrecupera (visto nos logs: ele mesmo chama `/api/student-state/delete` ~27s depois e recomeça com um `lessonLocalId` novo, que funciona). O registro específico contaminado é inofensivo daqui pra frente (não deve colidir de novo, IDs são aleatórios) e não exige limpeza manual de banco. A correção do item 3.4 impede que esse tipo de contaminação volte a ser criado.

---

## 8. TRATAMENTO DE ERRO DE REDE (achado P1, parcialmente corrigido)

### 8.1 Padrão correto
Mensagens genéricas ao usuário são aceitáveis na UI, mas a causa técnica real deve permanecer observável internamente — nunca "servidor indisponível" como resposta padrão para qualquer falha não tratada, sem rastro.

### 8.2 Como o SIM resolve hoje
`lib/sim/classroom/lesson_runtime_engine.dart:_surfaceCurrentPreparationFailureIfNeeded` extrai um `errorCode` real (ex.: `AI_RATE_LIMIT`, `CREDIT_OPERATION_REQUIRES_RECONCILIATION`, `RESOURCE_OWNER_MISMATCH`, `AI_TIMEOUT`, `T02_CONTRACT_INVALID`) de `pending.errorCode` ou de `_failedReadyWindowErrorCodeForCurrent`, mas na hora de decidir o que mostrar ao aluno colapsa TUDO em apenas duas strings: `'aula_server_unavailable'` (só se `errorCode == 'AI_RATE_LIMIT'`) ou `'aula_gen_fail'` (qualquer outra coisa).

### 8.3 Desvio exato
O `errorCode` real era descartado no processo — nenhum log, nenhum evento de diagnóstico, nada. Nesta própria investigação, a única forma de descobrir a causa real de um erro reportado pelo usuário foi acessar logs de produção via SSH e cruzar por horário — algo que não deveria ser necessário para um erro já classificado no cliente.

### 8.4 Correção aplicada (P1 → corrigido, baixo risco, puramente aditivo)
Adicionado `debugPrint('[SIM_OBS] category=lesson-runtime operation=preparation-failure-surfaced lessonLocalId=... itemIdx=... errorCode=...')` imediatamente antes do colapso para as 2 strings genéricas. Commit `f46bd52`. Zero mudança de comportamento visível ao usuário; suite completa (1434 testes) permanece verde.

### 8.5 Nota sobre o texto exato reportado
A string "Servidor indisponível, tente novamente" reportada não existe hoje em nenhuma tradução do app (foi o texto usado até a build 75, substituído por "Não consegui conectar agora. Tente novamente em instantes." antes da build 109 atual). Trata-se de paráfrase; a causa real do evento é a explicada na seção 3.5, confirmada por log de produção, não por suposição.

### 8.6 Achado adicional não corrigido (P2, registrado — ver seção "achados não corrigidos")
`requestId` do erro original (quando presente, extraído em `lib/sim/external_ai/sim_ai_server_config.dart:simSafeHttpException`) é descartado antes de chegar em `pending.errorCode` — só o `code` sobrevive. Incluir o `requestId` no log de diagnóstico do item 8.4 permitiria correlação direta com o log do servidor sem depender de janela de tempo. Não implementado nesta passada porque exigiria estender o schema de `AdvancePendingStatus`/`StudentLearningState` (mudança de superfície maior que o corredor local já tocado).

---

## 4. AUTENTICAÇÃO E SESSÃO

### 4.1 Cliente (app) — CONFIRMADO CANÔNICO
`lib/session/auth_session.dart` usa exatamente o padrão oficial do `supabase_flutter`: `client.auth.onAuthStateChange.listen(...)` para reagir a eventos de auth (incluindo refresh automático, que o próprio SDK do Supabase gerencia internamente), mais um fallback explícito no cold-start (`bindRealAuth`): se `currentSession.isExpired`, chama `refreshSession()` antes de aplicar; se o refresh falhar, aplica sessão nula (desloga com segurança, não trava em um estado indefinido). **Referência:** documentação oficial `supabase_flutter` (padrão `onAuthStateChange` + `refreshSession`). PROVEN, `lib/session/auth_session.dart:33-59`.

O token nunca é capturado/guardado cedo demais: `SimAiServerConfig.jsonHeaders()`/`streamHeaders()` (`lib/sim/external_ai/sim_ai_server_config.dart:30-45`) chamam `accessTokenProvider()` **no momento exato da chamada HTTP**, não antes — ou seja, o cabeçalho `Authorization` de qualquer request é sempre o token da conta ativa no instante do envio, nunca um token "congelado" de antes. PROVEN.

### 4.2 Pergunta central da missão: uma requisição da conta A pode retornar e ser aplicada depois que B já está ativa?

Resposta com evidência, não suposição:

- **Quanto à IDENTIDADE (quem autentica o request):** não. Como o token é lido fresco a cada chamada (4.1), nenhum request é *enviado* autenticado como A depois que B passou a ser a conta ativa — mesmo que o request tenha sido "iniciado" logicamente sob A.
- **Quanto ao CONTEÚDO/PAYLOAD do request:** aqui está o mecanismo real do vazamento da seção 3. Um request pode ser *montado* com dados de A (ex.: um `lessonLocalId` capturado num campo antes da troca) e só efetivamente disparado depois — nesse caso ele carrega o payload de A mas o token de B. O servidor autentica corretamente como B, mas o corpo referencia um recurso de A. Isso é exatamente o que a guarda de posse (`sim_ownership_records`, seção 3.5) existe para pegar — e pegou (403/409 nos logs). **HIGH CONFIDENCE**: o guard de posse funcionou como rede de segurança; a causa raiz (campo não limpo na troca de conta) foi corrigida na origem (seção 3.4), não apenas contida pelo guard do servidor.
- Para o fluxo de abertura de aula (`_openAulaRuntimeDirect`), existe uma guarda de stale-response completa e correta: `RuntimeOpenKey` (`lab_session.dart:1301-1316`) inclui `accountGeneration: canonicalStore.accountEpoch` — um contador que muda a cada troca de conta (`AccountScopedStudentStateLocalStorage.activateAccount`, `_accountEpoch += 1`) — combinado com `_runtimeOpenToken` comparado via `identical()`. Um resultado que chega depois da troca de conta tem `accountGeneration` desatualizado e é descartado. PROVEN, `lib/features/session/lab_session.dart:1301-1380` + `lib/sim/state/account_scoped_student_state_storage.dart:17-23`.

### 4.3 Servidor — CONFIRMADO CANÔNICO
`src/auth/jwt-verifier.js` faz verificação JWT completa e correta: valida `exp` (com tolerância explícita, `JWT_EXPIRED` se vencido), `iss` (compara contra `SUPABASE_JWT_ISSUER` configurado), `aud`, busca chaves via JWKS do próprio Supabase (`/auth/v1/.well-known/jwks.json`), e restringe algoritmos aceitos a uma lista explícita (`algorithms: [alg]`) — evita o ataque clássico de confusão de algoritmo (aceitar `alg: none` ou trocar RS256↔HS256 usando a chave pública como segredo HMAC). **Referência:** RFC 7519 (JWT) + práticas recomendadas de verificação (validação de todas as claims relevantes, allowlist de algoritmo). Já existe teste de contrato dedicado (`test/jwt_verifier_security_contract.test.js`, `test/auth_production_verifier_config.test.js`). PROVEN.

---

## 13. CACHE

### 13.1 Padrão correto
Todo cache com dado pessoal deve ter escopo explícito por conta; nenhuma chave de cache pode ser genérica o bastante para colidir entre duas contas diferentes no mesmo dispositivo.

### 13.2 Inventário e veredito por cache encontrado

| Cache | Arquivo | Escopo real | Veredito |
|---|---|---|---|
| Estado de aprendizagem local (progresso/curriculum por aula) | `lib/sim/state/account_scoped_student_state_storage.dart` | Chave física `account-v1:<base64 userId>:lesson:<base64 lessonLocalId>`; `writeState`/`writeEvents` validam que `state.userId` bate com a conta ativa, lançando `STATE_ACCOUNT_MISMATCH`/`EVENT_ACCOUNT_MISMATCH` caso contrário | **CANONICAL** — escopo por conta embutido na própria chave física + validação de posse na escrita. PROVEN (`account_scoped_student_state_storage.dart:1-222`). |
| Texto de aula gerado (warm/cold, `LessonMaterialCache`) | `lib/sim/lesson/lesson_material_cache.dart` + `lessonKeyFor` em `lib/sim/lesson/lesson_models.dart:145-167` | Persistido sob UMA chave SharedPreferences global (`sim-lesson-text-cache-v1`), mas a chave lógica de cada entrada é composta e inclui `params.accountId` explicitamente como segundo componente (antes de `lessonLocalId`) | **ACCEPTABLE CUSTOM** — mesmo que `lessonLocalId` colida entre contas (como colidiu na incidência da seção 3.5), a chave completa não colide porque `accountId` é parte dela. PROVEN, condicionado a `params.accountId` estar sempre populado corretamente em todo call site (não verificado exaustivamente — NOT VERIFIED para 100% dos call sites de `putForParams`). |
| Fila de sincronização com a nuvem | `lib/sim/cloud/shared_prefs_cloud_queue_storage.dart` | Chaves com sufixo de conta (`_activeQueueKey => '$_queueKey$_accountSuffix'`) | **CANONICAL**, escopo por conta confirmado na própria declaração da chave. PROVEN. |
| Migração legada SharedPreferences → Drift | `lib/sim/state/shared_prefs_state_storage.dart` | Chaves globais SEM prefixo de conta (`sim-student-learning-state-v1:lesson:<id>`, índice global `sim-student-learning-state-v1:index-v2`) | Por si só seria uma **DEVIATION** grave (índice compartilhado entre contas) — mas confirmado que esta classe só é usada como fonte de migração ÚNICA e não-recorrente (`legacy: stateStorage` em `lib/main.dart:106-109`), nunca como storage ativo (o storage ativo é sempre o Drift envolto em `AccountScopedStudentStateLocalStorage`). **Classificado como LEGACY/DEAD para uso corrente, não como bug ativo.** PROVEN via `main.dart:97-113`. |
| Aquecimento (warmup) | `lib/features/session/lab_session_warmup_flows.dart` | Campos soltos na instância `LabSession`, sem qualquer escopo de conta | Era **DEVIATION P0** — já corrigido na seção 3. |
| Retorno de pagamento (Play Billing) | `lib/sim/billing/payment_return_store.dart` | NOT VERIFIED nesta passada direta — delegado à sub-auditoria de billing/Android (seções 15/16). | Pendente de consolidação. |

### 13.3 Classe de bug identificada (família)
A causa raiz de todo achado P0 desta sessão (seção 3) é a mesma família: **identidade de recurso (`lessonLocalId`) tratada como se fosse naturalmente única entre contas, quando na verdade só é garantida única DENTRO de uma conta.** Qualquer cache/campo que usa `lessonLocalId` sozinho como chave está sujeito à mesma classe de risco caso o valor sobreviva a uma troca de conta. Verificado horizontalmente nesta seção: dos caches pessoais catalogados, apenas o de aquecimento (já corrigido) e o legado SharedPreferences (inerte) usavam `lessonLocalId` sem `accountId` embutido; os demais (estado local via Drift, texto de aula, fila de sync) já incluem a conta na chave física ou lógica.

---

## 14. SINCRONIZAÇÃO (SYNC)

### 14.1 Padrão correto
Last-write-wins ingênuo é uma anti-prática para estado rico; a política correta depende do campo (máximo monotônico, append-only, merge de eventos, autoridade do servidor, etc.), com resolução determinística e testável, nunca sobrescrevendo estado mais avançado com um mais pobre.

### 14.2 Como o SIM resolve — CONFIRMADO MADURO
Toda escrita de estado de aluno passa por um único ponto: `store.transact(...)` dentro de `src/student-state/student-state-controller.js:persist()` (linha 302) e `delete()` (linha 412) — não há nenhum outro caminho de escrita a `student_states` no servidor (confirmado por grep: `grep -rn "store.transact" src/` só retorna esses dois pontos). Dentro dessa transação existe uma cadeia extensa e explícita de guardas, cada uma com um código de erro próprio, na ordem: tombstone ativo (`STATE_TOMBSTONE_ACTIVE`) → perda de causalidade em compactação → **regressão de high-water-mark** (`STATE_HIGH_WATER_MARK_REGRESSION`, confirmado disparando em produção nos logs desta sessão) → revisão causal ausente/incompatível (`STATE_EXPECTED_REVISION_REQUIRED`/`_MISMATCH`, `STATE_REVISION_NOT_CAUSAL`) → preservação explícita de currículo mais rico (`preserveRicherRemoteCurriculum`) e de eventos existentes (`preserveExistingEvents`) antes de aceitar a escrita. Isso é precisamente "nunca sobrescrever estado rico com estado pobre" implementado como código, não como intenção. PROVEN, `src/student-state/student-state-controller.js:271-370`.

### 14.3 Consistência confirmada
Não foi encontrado nenhum segundo caminho de escrita que pule essas guardas (grep exaustivo por `store.transact`/`store.write` em `src/`). **Isso responde diretamente à exigência da missão de confirmar que os princípios foram aplicados em TODO o sync, não só nos pontos já corrigidos anteriormente**: a arquitetura é centralizada por construção (um único controller, uma única função de transação), então não há corredor paralelo a divergir.

### 14.4 Achado correlato (P2, já registrado na seção 3.5)
`/api/student-state/persist` não passa pelo guard de posse (`sim_ownership_records`) que `/api/student-state/get`, `/api/bootstrap-t00`, `/api/warmup` e `/api/complete-lesson` usam. Isso não é uma falha de sync em si (o rich-vs-poor merge continua correto por conta, já que a linha é sempre `(userId, lessonLocalId)`), mas é uma assimetria de defesa em profundidade: um `lessonLocalId` contaminado por um bug de cliente pode ser silenciosamente aceito por `persist` e só ser detectado bem mais tarde, em um endpoint diferente, com uma mensagem de erro confusa para o usuário (ver seção 8.5). Registrado como achado, não implementado nesta sessão — requer decidir a semântica de `accountGeneration`/`create` para o primeiro persist de um recurso novo e medir o custo de uma chamada RPC durável extra no caminho mais quente do produto (persist é chamado a cada avanço de item). Ver seção "Achados não corrigidos".

---

## 29. SCROLL — CONFIRMADO SEM VIOLAÇÃO

Contrato ratificado nesta mesma sessão (autoridade canônica única + scroll livre). Verificação transversal: `grep -rn "\.jumpTo(\|\.animateTo(" lib --include=*.dart | grep -v test` retorna resultados **apenas** em `lib/features/classroom/chat_aula_widgets.dart` — o arquivo que já é a autoridade sancionada. Nenhum outro arquivo do produto chama esses métodos diretamente. PROVEN, verificado nesta sessão após a auditoria mega ter sido solicitada.

## 32. IMAGENS — NÃO REVERIFICADO A FUNDO NESTA PASSADA, SEM REGRESSÃO DETECTADA

`git log --since="2026-09-17" -- "lib/sim/media/**"` não retorna nenhum commit — ou seja, nenhum código tocou o pipeline de imagem desde a última reconstrução (Parte A: prefetch N+1) auditada em sessões anteriores. Não há evidência de violação, mas também não houve reverificação profunda de estados/race conditions nesta sessão especificamente — **NOT VERIFIED** para além da ausência de mudança.

---

## 11. STATE MANAGEMENT — arquitetura Flutter (sub-auditoria dedicada)

Sub-auditoria independente confirmou que o padrão de "guarda contra resposta obsoleta" (stale-response guard) já identificado nesta sessão para o runtime de aula (`_runtimeOpenToken`) **é aplicado de forma consistente em praticamente todo async sensível do app**, através de pelo menos 4 variantes idiomáticas equivalentes: contador de geração comparado via `identical()`, releitura de estado vivo em vez de valor capturado, comparação por tupla `(userId, epoch)`, e identidade+geração passadas por closure. **Nenhuma instância foi encontrada de um resultado assíncrono aplicado sem qualquer guarda.** Isso é uma confirmação importante: o vazamento do aquecimento (seção 3) foi a EXCEÇÃO à disciplina do resto do código, não evidência de um padrão sistêmico ausente.

Achados específicos (nenhum P0/P1 novo):

- **[P3]** `lib/sim/auxiliary/lesson_doubt_controller.dart` define uma classe `LessonDoubtController` completa que **nunca é instanciada em lugar nenhum** — o controlador de Dúvida real é `LabSessionDoubtController` (dentro de `lab_session_warmup_flows.dart:770`). Código morto que pode induzir um engenheiro futuro a editar o arquivo errado. PROVEN.
- **[P2]** `LabSessionDoubtController` (o real) não tem um `resetForAccountChange()` explícito — mas sua guarda de staleness (`_isDoubtScopeStillCurrent`) já releitura estado vivo (`host.lessonLocalId`), que É zerado no `_onAccountChanged`, então o resultado prático é seguro; o risco é que essa proteção é **incidental** (depende de um reset feito por outro motivo), não local e auto-contida. Mesma observação se aplica a `LabSessionFirstLessonGate`, `LessonRecoveryController`/`ReviewController` — todos corretos hoje, todos dependendo de um campo alheio ser zerado. HIGH CONFIDENCE.
- **[P4]** `LabSessionClassroomInteractionController._expectedAdvanceLessonLocalId` não tem reset explícito nem comparação de geração local — não foi provado exploit, mas é uma assimetria a normalizar por consistência. SUSPECTED.

**Navegação (seção 30):** o app não usa uma pilha de navegação imperativa no nível superior — `MaterialApp(home: ...)` é recalculado a cada rebuild a partir de `session.route`/`session.authed`, então no logout a árvore de widgets inteira é desmontada (não há tela órfã de outra conta possível por esse mecanismo). PROVEN. Único ponto duvidoso, não confirmado: um modal de Dúvida aberto no exato instante de um logout concorrente poderia, em tese, sobreviver à troca de tela por viver na `Navigator` raiz — não reproduzido, SUSPECTED, P3.

**Lifecycle de widgets (seção 27) e pureza de `build()` (seção 28):** revisão de todos os `initState`/`dispose` em `lib/features/` e `lib/sim/ui/` não encontrou nenhum vazamento de `AnimationController`/`TextEditingController`/`FocusNode` — todos pareados corretamente. Nenhum efeito colateral não-guardado dentro de `build()` foi encontrado; os 3 side-effects existentes em `ChatAulaScreen.build()` são todos protegidos por chave composta + `mounted`. CONFIRMADO CANÔNICO, PROVEN.

**Dúvida — infra (seção 65):** confirmado que `submitDoubt` valida DOIS eixos de staleness (identidade da pergunta E assinatura de conteúdo), cancela áudio anterior incondicionalmente antes de tocar um novo, e o histórico de conversa é escopado por conta via `account:$userId:` como prefixo de chave, com uma guarda extra (`_provesLegacyOwnership`) para dados de uma migração legada não-escopada. PROVEN, GOOD — nenhum defeito de infraestrutura novo encontrado neste módulo.

---

## 45/46. PROCESS RESTART / MÚLTIPLAS INSTÂNCIAS / REDIS

### Como o SIM resolve — CONFIRMADO CORRETO
`src/ai/ai-cost-protection-gate.js:536` seleciona `createRedisAiCostProtectionGate` quando `config.IS_PRODUCTION` é verdadeiro — ou seja, o gate de dedup/single-flight de chamadas de IA (`AI_COST_SINGLE_FLIGHT_RUNNING`, visto disparando em produção nos logs desta sessão) é **apoiado em Redis**, não em memória de processo. Isso significa que o dedup funciona corretamente entre múltiplas instâncias do servidor atrás de um load balancer — se fosse um `Map` em memória, duas instâncias diferentes poderiam processar (e cobrar) a mesma operação simultaneamente sem se verem. PROVEN, `src/ai/ai-cost-protection-gate.js:974-1613`.

Igualmente importante: o próprio módulo se autodocumenta como **não autoritativo** para dinheiro (`policy: () => ({distributed: true, durableLedger: Boolean(durableLedger), redisAuthoritative: false, ...})`, linha 1613) — Redis é usado como cache/gate de velocidade na frente do ledger durável em Postgres/Supabase (`bom_durable_reserve_effect`/`bom_durable_capture_credit`), nunca como substituto dele. Isso bate com o princípio já documentado internamente pelo próprio projeto ("Ledger durável indisponível impede nova operação paga; não usa Redis como substituto"). Este é exatamente o padrão correto para "o que acontece se o processo reiniciar / houver duas instâncias": o dado que sobrevive de verdade está no Postgres, Redis é descartável sem perda de integridade financeira (pode, na pior hipótese, permitir uma reexecução que o guard de ownership/high-water-mark do Postgres pegaria depois).

---

## 37/38/39. GOOGLE PLAY BILLING, CRÉDITOS, CHAMADAS PAGAS DE IA (sub-auditoria + verificação própria)

Uma sub-auditoria dedicada ao servidor levantou dois achados classificados como P0. Verifiquei cada um pessoalmente antes de aceitar a classificação — o resultado, com evidência, está abaixo. Isso é exatamente o tipo de disciplina que a missão pede (seção 5, "proibição de alucinação de problema" — não aceitar severidade sem prova).

### 37.1 Verificação/servidor nunca confia no cliente para conceder crédito — CONFIRMADO CORRETO
`src/play-billing/play-billing-controller.js:verifyGooglePlayProductPurchase` chama a API oficial do Google (`androidpublisher/v3/.../purchases/products/{productId}/tokens/{purchaseToken}`) com um token OAuth de service-account; `consumeCreditPack` sempre chama essa verificação antes de conceder crédito, e o campo `localVerificationData` enviado pelo cliente nunca é lido pelo controller. Idempotência é garantida pelo `orderId` verificado pelo Google (ou hash do `purchaseToken`), com constraint única no banco (`unique(account_id, idempotency_key, movement_kind)`). PROVEN.

### 37.2 Achado original do sub-auditor (P0): "nenhum acknowledgement de compra é enviado ao Google, violando a janela mandatória de 3 dias"
Confirmei via grep que `play-billing-controller.js` de fato só **lê** `acknowledgementState` da resposta de verificação — nunca chama a API de acknowledge/consume do Google. PROVEN, isso é fato.

**Correção da minha própria verificação, reclassificando a severidade:** o contrato do Google Play Billing permite que o acknowledgement aconteça no CLIENTE ou no SERVIDOR, contanto que aconteça dentro de 3 dias. Verifiquei `lib/sim/billing/play_billing_functions.dart:265-288`: o app Flutter **já chama** `_store.consumePurchase(purchase)` e `_store.completePurchase(purchase)` (via `in_app_purchase_android`, que por sua vez aciona `acknowledgePurchase`/`consumeAsync` nativos do Google) **depois** de o servidor confirmar e conceder o crédito (`grantGateway.grantCreditPack` é aguardado primeiro). Ou seja, o acknowledgement/consumo REALMENTE ACONTECE hoje, só que do lado do cliente, não do servidor.

**Risco residual real (rebaixado para P1, não P0):** se o app travar/perder rede exatamente entre "servidor já concedeu o crédito" e "cliente chama consumePurchase/completePurchase", a compra fica concedida no SIM mas nunca reconhecida no Google — Google reembolsa automaticamente depois de alguns dias, e não existe nenhuma varredura de reconciliação no servidor para detectar e estornar esse crédito já concedido. A própria documentação do Google recomenda o acknowledgment server-side justamente por ser mais resiliente a esse cenário de processo morto no meio do fluxo (ver seção 48 — "crash entre etapas"). **Ação recomendada, não implementada nesta sessão** (exige desenho de uma tarefa de reconciliação periódica cruzando compras concedidas vs. reembolsos do Google — redesenho, não fix local): registrado como P1.

### 37.3 Consumíveis nunca são "consumidos" via API do Google (P1, mesma causa)
Mesma ausência de chamada — mas como o CLIENTE já chama `consumePurchase` (ver 37.2), este ponto específico também está coberto na prática atual, com o mesmo risco residual de processo morto. Não é uma segunda falha independente.

### 37.4 Achado original do sub-auditor (P0): endpoints públicos `/api/credits/reserve|capture|refund` aceitam custo/operationId arbitrários do cliente, sem checagem de posse
Confirmei o código (`src/credits/credits-controller.js:12-45`): de fato, `reserve` usa `data.cost`/`data.operationId` vindos diretamente do corpo JSON do cliente, sem derivar de nenhum produto real. **Isso é fato, PROVEN.**

**Correção da minha própria verificação, reclassificando a severidade:** tracei o RPC durável por trás (`bom_durable_release_credit`/`bom_durable_capture_credit`, `supabase/migrations/202608300002_phase_b2_durable_ledger.sql:248-276`) e confirmei que:
- Toda operação é sempre escopada a `p_account_id = req.auth.userId` — nunca controlável pelo cliente (vem do JWT verificado, não do corpo da requisição). Um cliente não pode mexer no saldo de outra conta por essa via.
- `release`/refund **recusa explicitamente** reverter uma reserva já em estado `captured`/`accepted`/`failed_terminal`/`cancelled` (`if o.status in (...) then raise exception 'release_invalid_terminal_state'`) — ou seja, o cenário mais perigoso que eu cogitei (pagar de verdade via o fluxo interno do T02, depois chamar o `/refund` público com o mesmo `reservationId` para reaver o crédito sem devolver o conteúdo já entregue) **está bloqueado no banco**, independente da falta de checagem na camada HTTP.
- Rastreei o único chamador real desse endpoint no app (`SimServerCreditsClient.chargeLessonGeneration`, `lib/sim/billing/sim_server_billing_clients.dart:136-153`) e confirmei, por busca exaustiva, que **`chargeLessonGeneration` nunca é chamado em lugar nenhum do app** (`grep` por `chargeLessonGeneration(` fora da própria definição/interface não retorna nada). Isso é resquício de uma arquitetura de cobrança mais antiga (cliente decide o custo) que foi substituída pela atual (custo derivado no servidor dentro do próprio `/api/complete-lesson`, via `config.T02_ITEM_CREDIT_COST`) — mas nunca foi removido nem do cliente nem do servidor.

**Veredito ajustado: P1, não P0.** O endpoint é alcançável por qualquer cliente autenticado (curl, app modificado) e permite criar reservas/capturas com rótulos arbitrários — o que é uma falha real de "vinculação a produto real" e um ponto cego de auditoria de ledger — mas o dano verificável está limitado à própria conta do chamador (não há como afetar outra conta) e o banco impede o cenário de "pagar e depois estornar de graça". **Não desliguei nem alterei este endpoint nesta sessão**: é uma API financeira ao vivo, e não tenho certeza suficiente de que nenhuma versão mais antiga do app ainda em uso dependa dela para cobrança de aula — desligar às cegas durante o freeze de release poderia quebrar clientes antigos pagantes, o que seria pior que o risco atual. Registrado como achado P1 com plano de correção (ver seção "Achados não corrigidos").

### 37.5 Compras PENDING são rejeitadas sem acompanhamento (P2)
Confirmado: `purchaseState !== 0` (não-comprado) é rejeitado com 409 direto, sem nenhum listener de Real-time Developer Notifications do Google para reprocessar quando uma compra pendente eventualmente é aprovada. Nenhuma evidência de isso já ter causado perda real (não há relato de usuário nesse sentido). Registrado, não corrigido (exige integração nova com webhook do Google — redesenho).

### 38. Ledger de créditos — idempotência confirmada madura
Rastreado ponta a ponta em dois call sites reais (item de aula via T02, salas auxiliares via `auxRoomCreditOperationKey`) mais um terceiro (geração de imagem) — todos convergem no mesmo padrão `reserve→capture→release` com RPC Postgres fazendo `select ... for update` (lock de linha) e retorno idempotente em estados terminais. PROVEN. Consistente com o padrão correto de sistemas de ledger (nenhuma duplicação, reserva identificável, capture/release, idempotência) exigido pela seção 38 da missão.

**[P1]** Um `catch (_) {}` em `src/ai/ai-cost-protection-gate.js:1588-1594` engole silenciosamente falhas ao marcar uma reserva como ambígua/falha no ledger durável — 8 linhas acima, o caminho irmão (`financial.release` falhando) É logado (`AI_FINANCIAL_RELEASE_FAILED`). Isso é uma inconsistência real dentro do PRÓPRIO padrão do arquivo (não uma comparação externa): uma reserva pode ficar presa sem nenhum sinal operacional, só aparecendo depois como `CREDIT_OPERATION_REQUIRES_RECONCILIATION` para o usuário. PROVEN, fácil de corrigir (adicionar `safeWarn`), mas não corrigido nesta sessão por estar num arquivo extenso e crítico (`ai-cost-protection-gate.js`) que não foi tocado ainda nesta auditoria — registrado para correção pontual futura de baixíssimo risco.

## 46. REDIS — CONFIRMADO CORRETO (fail-closed, não autoritativo para dinheiro)

Confirmado (por mim e pelo sub-auditor, independentemente): produção usa um cliente Redis real (`redis` v4, não mock), com lock distribuído via `SET NX PX` + Lua compare-and-delete (padrão oficial do Redis para locks), rate-limit via `EVAL` atômico. Indisponibilidade do Redis **falha fechado** (propaga erro, não deixa passar silenciosamente nenhum gate). Redis nunca é a fonte de verdade para dinheiro — é um cache/gate de velocidade na frente do ledger durável em Postgres (ver seção 45/46 acima, já confirmado por mim diretamente ao inspecionar `ai-cost-protection-gate.js`). **[P1, contingente]** não existe uma asserção explícita de boot travando `DURABLE_LEDGER_MODE` para fora de um modo só-Redis em produção — o padrão do próprio código (checagens de invariantes no boot para outras configurações críticas) sugere que essa checagem deveria existir também aqui, por consistência, mesmo que hoje dependa de má configuração ativa para ser um problema real.

## 44. SERVER ARCHITECTURE — achados adicionais

- **[P0, verificado, ver 37.4]** Duplicação real de responsabilidade: `/api/credits/reserve|capture|refund` reimplementa como rota pública um primitivo que deveria ser só interno — rebaixado a P1 após minha verificação de blast radius, mas é o único caso GENUÍNO de duplicação de rota/autoridade encontrado (nenhuma outra duplicação de leitor de corpo HTTP ou parser de auth foi encontrada).
- **[Confirmado forte]** `src/logs/safe-log.js` tem redação por padrão (chave conhecida) E por valor (string suspeita mesmo sob chave inofensiva), produzindo tokens derivados de SHA-256 — atende ou excede o OWASP Logging Cheat Sheet. Nenhum `console.*` cru encontrado em nenhum caminho de auth/billing/créditos. PROVEN.
- **[P2]** Uma flag de recuperação legada (`AI_COST_T02_LEGACY_RESULT_RECOVERY`) lê diretamente de `process.env`, ignorando a camada central de config (`src/config/env.js`) e não é validada no boot — hoje comprovadamente desligada por padrão (`false`) e sem nenhuma referência que a ligue em produção, mas é um ponto cego de configuração. Registrado, não alterado (fora do escopo desta sessão mexer em `ai-cost-protection-gate.js` para isso).
- **Zero ocorrências de `TODO`/`FIXME`** em todo o `src/` fora de testes — nenhuma pendência de código sinalizada in-line.

---

## 50/51. BUILD ANDROID, SEGURANÇA MÓVEL E DEPENDÊNCIAS (sub-auditoria dedicada)

### Corrigido nesta sessão (P2 → aplicado, baixo risco)
`android:allowBackup` estava no padrão da plataforma (`true`, implícito) enquanto a sessão de autenticação do Supabase é persistida em `SharedPreferences` padrão (sem `flutter_secure_storage`) — ou seja, o token de sessão era elegível para o Auto Backup do Android para a conta Google do usuário. **Corrigido**: adicionado `android:allowBackup="false"` no `AndroidManifest.xml` (commit `911ccfc`). Mudança de um atributo, sem dependência nova, sem risco funcional (o único efeito é que dados locais não são restaurados após desinstalar/reinstalar — o usuário loga de novo, o que é aceitável já que o estado real vive no servidor).

### Confirmado correto (sem ação necessária)
- **R8/minify**: `isMinifyEnabled=false` no build de release — **[P1, registrado, não corrigido]**. Habilitar R8 exige regras ProGuard testadas contra Play Billing/Supabase (reflexão) e um build+teste físico completo antes de ir para produção — não é seguro fazer às cegas durante o freeze de release. Recomendado como próxima missão dedicada, não implementado agora.
- **`applicationId` de produção**: mecanismo de override (`-PSIM_ANDROID_APPLICATION_ID`) tem dupla trava (`validateReleaseSafety()` no Gradle + guarda bash no script de build oficial) que impede um upload de produção com o applicationId errado — **controle positivo, já correto**, confirmado nesta sessão.
- **targetSdk 36 / minSdk 24**: acima do mínimo exigido atualmente pelo Google Play — conforme.
- **network_security_config.xml de produção**: `cleartextTrafficPermitted="false"`, sem overrides de domínio — correto. Existe um segundo arquivo (`..._cleartext.xml`) usado apenas por um caminho de build de teste explicitamente gateado (`SIM_ANDROID_ALLOW_CLEARTEXT`) e bloqueado para `FLUTTER_APP_MODE=production` — **[P2]** esse arquivo fica embutido em todo artefato mesmo assim; recomendação registrada (não implementada) de excluí-lo fisicamente do variant de release via resource set dedicado.
- **Segredos no nativo Android**: nenhuma chave/credencial hardcoded encontrada em `build.gradle.kts`, manifests, `MainActivity.kt`; tudo indireto via `key.properties` (já confirmado gitignored) ou variáveis de ambiente.
- **`payment_return_store.dart`**: investigado por hipótese de guardar token de pagamento sensível — na prática só guarda um *path* interno de retorno pós-pagamento, validado contra path traversal/redirecionamento (`isSafeInternalPath`). Não sensível, confirmado benigno.
- **Nenhum Firebase/`google-services.json`** presente no projeto — item da missão não aplicável a este app (stack de auth é 100% Supabase).

### Registrado, não corrigido
- **[P2]** Sessão do Supabase persistida em `SharedPreferences` padrão, não em `flutter_secure_storage`/Keystore. Padrão comum em apps Supabase Flutter, mas mais fraco que o ideal do OWASP MASVS-STORAGE. Migrar exigiria adicionar dependência nova e implementar um `LocalStorage` customizado para o `supabase_flutter` — mudança de superfície, não "local e óbvia o bastante" para decidir sem o usuário.
- **[P2]** `checkSimMandatoryUpdate()` falha aberto (permite o app prosseguir) em qualquer erro/timeout na checagem de versão mínima — é uma decisão de produto legítima (evitar bloquear todo mundo por uma falha transitória do endpoint de versão), mas está implícita num `catch` genérico, não documentada como escolha consciente. Registrado para decisão explícita futura, não alterado.
- **`sqlite3_flutter_libs: 0.6.0+eol`**: confirmado que o sufixo `+eol` é um marcador intencional do próprio mantenedor (pacote não faz mais nada, existe só como placeholder) — recomendação de higiene (remover a dependência morta), não um risco.

---

## 41. SEGREDOS — CONFIRMADO LIMPO (verificação direta, ambos os repositórios)

- **APP**: varredura por padrões de chave (Google API key `AIza...`, chave estilo OpenAI `sk-...`, blocos `PRIVATE KEY`) em `lib/`, `android/` não encontrou nenhum segredo hardcoded. `android/key.properties` (credenciais reais de assinatura) **não está rastreado no git** — coberto por `android/.gitignore` (`key.properties`, `**/*.keystore`, `**/*.jks`, padrão oficial gerado pelo próprio template Flutter). Confirmado via `git status --ignored`. PROVEN.
- **SERVER**: mesma varredura em `src/` não encontrou segredos hardcoded. Nenhum arquivo `.env`/`.env.*` rastreado no git (`.gitignore` cobre `.env` e `.env.*`, com exceção explícita de `.env.example`). PROVEN.

---

## 17/18/19/20. CANCELAMENTO, RETRIES, TIMEOUTS, IDEMPOTÊNCIA (sub-auditoria dedicada, ambos os repositórios)

### Retries — catálogo
Nenhum padrão proibido encontrado em nenhum dos dois repositórios: **nenhum retry instantâneo de 429 sem backoff**, **nenhum loop de retry sem teto**. Todo caminho que retenta 429/`AI_RATE_LIMIT` aplica um atraso fixo ou delega ao `Retry-After` do servidor; a rota T02 e o cost-gate se recusam categoricamente a retentar automaticamente 429.

**[P1, PROVEN]** Amplificação multiplicativa real de custo: o retry externo do `ai-cost-protection-gate.js` (3 tentativas) envolve o retry interno do próprio `complete-lesson-controller.js` (3 tentativas) — até **9 chamadas reais ao provedor de IA por um único crédito reservado**, mesmo padrão para imagem. A reserva/liberação de crédito acontece uma única vez ao redor de todo o aninhamento (não há cobrança duplicada), mas é uma exposição real de amplificação de custo perante o provedor de IA, sem teto agregado.

**[P2, PROVEN]** Loop de "regeneração" de reserva em `credits-store.js:217` (até 20 iterações) não tem nenhum backoff entre tentativas — não multiplica custo de IA, mas é um loop sem atraso sob contenção de lock no banco.

### Timeouts — classificação correta na maioria dos casos
**[Confirmado correto, PROVEN]** Timeout de chamada ao provedor de IA (Gemini/OpenAI) é tratado como **resultado ambíguo** (`AI_TIMEOUT`, `providerRequestStarted: true, providerOutcomeKnown: false`), roteado para reconciliação, nunca como "com certeza não aconteceu". Falha antes de qualquer chamada de rede é corretamente marcada `providerRequestStarted: false`. Consumo de compra do Google Play só acontece depois da concessão de crédito ter sucesso, nunca antes — mesmo em timeout.

**[P2, HIGH CONFIDENCE]** O timeout do T02 (140s no app, timeout de rota no servidor) é tratado como `retryable: true` incondicionalmente em ambos os lados — para uma operação cujo resultado real no servidor pode estar indeterminado. A segurança desse retry depende inteiramente da chave de idempotência estável ser respeitada; não foi verificado ponta a ponta que essa deduplicação realmente acontece no servidor para uma requisição retentada especificamente por timeout (distinto de uma falha explícita).

**[P2, PROVEN]** As chamadas HTTP de verificação de compra do Google Play (`play-billing-controller.js`) não têm timeout explícito nenhum — risco de disponibilidade (travar), não de cobrança duplicada (a concessão já é idempotente por `orderId`/hash do token).

### Idempotência — praticamente livre do anti-padrão "DateTime.now() como identidade"
Varredura completa dos dois repositórios por chave de idempotência derivada de `Date.now()`/`DateTime.now()` **não encontrou nenhuma ocorrência nas superfícies economicamente sensíveis do servidor** (cobrança de item T02, sala auxiliar, Google Play, imagem). No app, duas exceções menores, ambas de baixo risco:
- **[P3, HIGH CONFIDENCE]** `operationId` de Amparo (`student_aux_room_service.dart:725`) é derivado de `DateTime.now().millisecondsSinceEpoch` — um retry da mesma tentativa lógica gera uma identidade nova, derrotando a deduplicação do servidor para esse fluxo específico.
- **[P4, PROVEN, baixo risco]** Um campo chamado `idempotencyKey` no fluxo de Dúvida é na verdade um ID de telemetria local baseado em tempo — a deduplicação real de custo de IA para Dúvida passa por outra chave (`T02RequestCoordinator`, estável, por hash). Nome enganoso, sem efeito prático.

### Cancelamento — forte na maior parte, duas lacunas concretas encontradas
`lib/sim/lesson/lesson_orchestrator.dart` e `chat_aula_screen.dart` têm disciplina de cancelamento em camadas (token vivo verificado antes/depois de cada `await`, contador de geração, comparação de identidade de conteúdo) — modelo de referência.

**[P1, PROVEN, verificado pessoalmente]** Cadeia sem guarda alguma, provavelmente ligada a uma operação cobrada: `lib/sim/external_ai/sim_server_attachment_client.dart` retenta uploads de anexo (3x, em 408/429/5xx) **sem nenhuma chave de idempotência** no multipart; o chamador, `doubt_input_sheet_widget.dart:_pickImage` (linhas 65-95), tem dois `await` (picker de imagem, leitura de bytes) **sem nenhuma checagem de `mounted`** — confirmei diretamente que o único `mounted` do arquivo está numa função diferente (linha 118); e o `ChangeNotifier` que processa o arquivo depois (`entry_form_state.dart`) não tem nenhuma guarda de staleness/cancelamento. É uma cadeia completa, do toque do usuário à resposta do servidor, sem cancelamento e sem deduplicação.

**[P2, PROVEN]** Em `lab_session_media_controller.dart`, `toggleAudio()` tem uma guarda própria completa (contador `_audioOperation` checado antes/depois de cada `await` e no `finally`), mas as funções irmãs `speakDoubt()`/`speakAuxRoomContent()` no MESMO arquivo não têm guarda nenhuma — um resultado tardio pode escrever `audioError`/notificar ouvintes contra um contexto de aula que o usuário já deixou. É a mesma classe de bug do vazamento do aquecimento (resultado assíncrono aplicado sem checagem), num arquivo que já tem a correção certa duas funções ao lado.

### Single-flight / mapas de operação em voo
O gate de custo de IA do servidor foi testado quanto a casos extremos: quando a requisição que um "joiner" está esperando falha, o joiner sai do loop de espera de forma limpa e recebe um erro genérico e limitado — nunca trava, nunca aplica dado errado silenciosamente. **[P1, NOT VERIFIED, item aberto]** não foi confirmado se o registro de replay de idempotência do controlador de imagem (`src/app/media-cache.js`) escopa sua chave por `userId` ou só pela `idempotencyKey` fornecida pelo cliente — se for só a segunda, duas contas enviando a mesma string poderiam colidir em teoria. Fica como item para uma leitura de acompanhamento, não como bug confirmado.

No app, `T02RequestCoordinator._inflight` usa uma chave que já inclui `accountId`/`accountGeneration` (correto, imune à classe de bug do vazamento). **[Achado de família, mitigado, PROVEN]** `BomPreparedExperienceCoordinator`/`StudentAuxRoomService` têm seus próprios mapas de operação em voo escopados **apenas** por `lessonLocalId` — sem componente de conta — mas não são exploráveis hoje porque `SimOrganismProvider.resetAccountContext()` descarta o organismo inteiro (e esses mapas junto) na troca de conta. É segurança por acidente arquitetural, não por uma chave própria — um risco latente se algum dia existir uma segunda referência de vida mais longa a esses objetos (não encontrada nesta auditoria).

### Achados positivos confirmados nesta seção
Nenhum retry instantâneo de 429; nenhum loop infinito; praticamente nenhum uso de `DateTime.now()` como identidade em superfícies pagas; ordem de consumo de compra do Google Play sempre depois da concessão ter sucesso, em ambos os caminhos de erro; `ensureNextAulaAdvancePrepared` mantém o limite/cooldown já corrigido em sessão anterior (4 tentativas, 20s, por item).

---

## Correções realizadas nesta sessão

| # | Achado | Antes | Depois | Arquivos | Testes | Commit |
|---|---|---|---|---|---|---|
| 1 | Vazamento de aquecimento entre contas (P0/BLOCKER) | `_onAccountChanged` não zerava `warmupLesson`/`warmupError`/`warmupSelectedAnswer`/`warmupWaitingForOfficialLesson`/flights em voo | Novo `LabSessionWarmupController.resetForAccountChange()`, chamado em `_onAccountChanged` | `lib/features/session/lab_session.dart`, `lib/features/session/lab_session_warmup_flows.dart` | Novo `test/account_switch_state_isolation_test.dart` (2 casos, falha sem o fix); suite completa 1434/1434 | `1a9cc08` (APP) |
| 2 | Erro genérico "servidor indisponível" sem rastro interno | `errorCode` real descartado antes de virar 1 de 2 strings genéricas | `debugPrint` estruturado com `lessonLocalId`/`itemIdx`/`errorCode` real antes do colapso | `lib/sim/classroom/lesson_runtime_engine.dart` | Suite completa 1434/1434 (aditivo, sem mudança de comportamento) | `da62a73` (APP) |
| 3 | Sessão Supabase elegível a Auto Backup do Android | `allowBackup` no padrão da plataforma (implícito `true`) | `android:allowBackup="false"` explícito | `android/app/src/main/AndroidManifest.xml` | `flutter analyze` limpo, manifesto validado como XML | `5533627` (APP) |
| 4 | Falha de transição do ledger durável (`markEffectAmbiguous`/`failEffect`) engolida silenciosamente em 2 pontos | `catch (_) {}` sem log, ao lado de um caminho irmão (`financial.release`) que já loga | `safeWarn('AI_DURABLE_EFFECT_TRANSITION_FAILED', {...})` nos 2 pontos, espelhando o padrão já existente | `src/ai/ai-cost-protection-gate.js` | Suite completa do servidor: **102/102 arquivos passando** | `8ca0e91` (SERVER) |
| 5 | `speakAuxRoomContent` escreve `audioError`/notifica sem checar se a operação ainda é a atual (mesma família do vazamento do aquecimento) | Catch sem nenhuma guarda de staleness, ao lado de `toggleAudio` no mesmo arquivo, que já tem a guarda correta | Guarda `_audioOperation` capturada antes do `await`, checada no catch antes de mutar estado — mesmo padrão de `toggleAudio` | `lib/features/session/lab_session_media_controller.dart` | Suite completa 1434/1434 | `190138d` (APP) |

## Achados não corrigidos (exigem decisão/redesenho maior)

| # | Achado | Severidade | Por que não corrigi agora | Arquitetura recomendada | Prioridade sugerida |
|---|---|---|---|---|---|
| 1 | `/api/student-state/persist` não passa pelo guard de posse (`sim_ownership_records`) que `get`/`bootstrap-t00`/`warmup`/`complete-lesson` usam | P2 | `persist` é o caminho mais quente do produto (chamado a cada avanço de item); adicionar uma chamada RPC durável extra nesse caminho exige medir custo de latência e decidir a semântica de `accountGeneration`/`create` para o primeiro persist de um recurso novo — mudança de comportamento, não just log | Espelhar a mesma chamada `assertRequestResourceOwners`/`ensureLessonOwnerFromAuthoritativeStudentState` já usada em `get`, com `create: true` apenas na primeira escrita de um `lessonLocalId` | P2 |
| 2 | Endpoints públicos `/api/credits/reserve\|capture\|refund` aceitam custo/operationId do cliente sem vínculo a produto real | P1 (rebaixado de P0 do achado original após verificação: dano confirmado limitado à própria conta do chamador, ledger bloqueia reverter reserva já capturada) | É uma API financeira ao vivo; o único chamador conhecido no app atual (`chargeLessonGeneration`) está confirmadamente morto (nunca invocado), mas não tenho certeza de que nenhuma versão mais antiga do app, ainda potencialmente em uso por algum usuário, dependa dela — desligar às cegas durante o freeze de release arrisca quebrar um cliente pagante antigo | Antes de restringir: (a) confirmar nos logs de produção se essa rota já recebeu tráfego real nos últimos ~90 dias; (b) se não, remover a rota e o código morto do cliente juntos; (c) se sim, migrar para custo derivado no servidor por `reason` (mesmo padrão do T02) antes de travar | P1 — investigar uso real em produção primeiro |
| 3 | Nenhum acknowledgement/consumo de compra feito pelo **servidor** (só o cliente faz, via `in_app_purchase_android`) | P1 | Acknowledgement client-side já cobre o caso comum; implementar reconciliação server-side exige uma tarefa periódica nova cruzando compras concedidas vs. status no Google — redesenho, não fix local | Job periódico de reconciliação: para toda concessão de crédito via Play Billing sem confirmação de acknowledge dentro de X horas, reverificar o purchaseToken contra a API do Google e (se já reembolsado) estornar o crédito | P1 |
| 4 | Compras `PENDING` do Google Play são rejeitadas sem acompanhamento (sem RTDN) | P2 | Exige integração nova com Real-time Developer Notifications do Google — redesenho | Webhook RTDN + fila de reprocessamento quando uma compra pendente transiciona para `PURCHASED` | P2 |
| 5 | R8/ProGuard desabilitado no build de release Android | P1 | Habilitar exige regras ProGuard testadas contra Play Billing/Supabase (reflexão) + build físico completo — arriscado fazer às cegas durante o freeze de release | Adicionar `proguard-rules.pro` com as regras conhecidas do Flutter + Play Billing, testar build de release completo num dispositivo físico antes de habilitar `isMinifyEnabled=true` | P1 — próxima missão dedicada |
| 6 | Sessão Supabase em `SharedPreferences` padrão, não em armazenamento seguro (`flutter_secure_storage`) | P2 | Exige nova dependência + `LocalStorage` customizado para o `supabase_flutter` — mudança de superfície de auth, não local o bastante para decidir sozinho | Implementar `LocalStorage` customizado usando `flutter_secure_storage`, mantendo compatibilidade com sessões já persistidas (migração) | P2 |
| 7 | `checkSimMandatoryUpdate()` falha aberto em qualquer erro/timeout | P2 | É uma escolha de produto (fail-open evita bloquear todo mundo por instabilidade transitória do endpoint de versão) que precisa ser uma decisão consciente documentada, não um efeito colateral de um `catch` genérico | Decidir explicitamente e documentar a política; se decidido manter fail-open, documentar o porquê no próprio código | P2 |
| 8 | `requestId` do erro original descartado antes de chegar em `pending.errorCode` (só o `code` sobrevive) | P2 | Exigiria estender o schema de `AdvancePendingStatus`/`StudentLearningState` — mudança de superfície maior que o corredor já tocado nesta sessão | Adicionar campo `requestId` opcional ao lado de `errorCode` em toda a cadeia (ready-window → advancePending → log de diagnóstico) | P3 |
| 9 | Flag `AI_COST_T02_LEGACY_RESULT_RECOVERY` lida direto de `process.env`, fora da camada central de config, sem validação no boot | P3 | Hoje comprovadamente desligada por padrão e sem nenhuma referência ativando-a — baixo risco imediato, mexer no arquivo `ai-cost-protection-gate.js` além do já feito nesta sessão amplia o raio de mudança sem necessidade | Mover para `src/config/env.js` como qualquer outra config, validar no boot | P3 |
| 10 | Upload de anexo (Dúvida): retry sem chave de idempotência + `_pickImage` sem guarda `mounted` + `entry_form_state.dart` sem guarda de staleness — cadeia inteira sem cancelamento nem deduplicação | P1 | Envolve 3 arquivos em 2 camadas (rede + widget + `ChangeNotifier`); exige desenho de uma chave de idempotência para upload multipart e um teste de regressão para o cenário completo (retry + navegação no meio) — não é uma mudança de uma linha | Adicionar `requestId`/hash-de-conteúdo ao multipart; adicionar guarda `mounted` em `_pickImage`; adicionar contador de geração em `entry_form_state.dart` seguindo o padrão já usado em `lesson_orchestrator.dart` | P1 |
| 11 | Amplificação multiplicativa de custo: retry externo do cost-gate (3x) envolve o retry interno do T02/imagem (3x) — até 9 chamadas reais ao provedor por crédito reservado | P1 | Não duplica cobrança (reserva/liberação acontece uma vez ao redor de tudo), mas mudar a política de retry em qualquer uma das duas camadas exige entender o motivo histórico de terem sido feitas independentes — risco de regressão em resiliência a falhas transitórias do provedor se removido às cegas | Decidir explicitamente qual camada é dona do retry (provavelmente a interna, já que a externa deveria só orquestrar o gate) e desativar o aninhamento; ou aceitar como orçamento de custo consciente e documentar o teto agregado | P1 |
| 12 | Não verificado se o registro de replay de idempotência do controlador de imagem (`src/app/media-cache.js`) escopa por `userId` ou só pela `idempotencyKey` do cliente | P1 (NOT VERIFIED, requer leitura de acompanhamento) | Não foi confirmado nem como bug nem como seguro nesta sessão — está fora do corredor já verificado (`ai-cost-protection-gate.js`), exige leitura dedicada de `media-cache.js` antes de qualquer ação | Ler `src/app/media-cache.js` e confirmar se a chave de replay inclui `userId`; se não incluir, adicionar | P1 — próxima leitura, antes de qualquer decisão |
| 13 | `operationId` de Amparo derivado de `DateTime.now()` — retry da mesma tentativa lógica gera identidade nova | P3 | Baixo risco isolado, mas mexer no formato da chave de idempotência de um fluxo de billing exige o mesmo cuidado dos outros itens de idempotência — não é uma troca de uma linha sem risco de regressão em replay já em produção | Trocar para chave content-derived (lessonLocalId + amparoLevel + hash do conteúdo), como já é feito no warmup e no T02 | P3 |
| 14 | `credits-store.js` loop de regeneração de reserva (até 20 tentativas) sem nenhum backoff entre tentativas | P2 | Não amplifica custo de IA, mas mexer no timing desse loop sem entender a causa da contenção de lock que o motivou exige investigação própria | Adicionar backoff curto com jitter entre tentativas de regeneração | P2 |

## Famílias de bugs encontradas (comprovadas, não hipotéticas)

1. **Identidade de recurso tratada como globalmente única quando só é única por conta** — causa raiz do achado P0 principal (aquecimento) e do efeito colateral já materializado em produção (`sim_ownership_records`/`student_states` cross-account para `cyber-1r2yb4g`). Auditada horizontalmente: dos caches/campos pessoais catalogados nesta sessão, apenas os dois relacionados a esse mesmo incidente estavam vulneráveis; os demais (estado local via Drift, cache de texto de aula, fila de sync, doubt/review/recovery via releitura de estado vivo) já incorporam a conta na chave ou na validação. **Não é um padrão sistêmico do produto — foi uma exceção real, já corrigida na origem.**
2. **Proteções de staleness incidentais em vez de auto-contidas** — vários controladores (Dúvida, Recuperação, Revisão, first-lesson-gate) só evitam aplicar resultado obsoleto porque um CAMPO ALHEIO é zerado por outro motivo no `_onAccountChanged`, não por uma checagem de geração local e explícita. Funciona hoje, é frágil a uma futura refatoração do reset central. Registrado, não corrigido (baixo risco imediato, correção ampla e sistemática seria gold-plating fora do escopo de segurança desta missão).
3. **Catch silencioso ao lado de um caminho irmão já logado, dentro do mesmo arquivo** — os dois pontos corrigidos em `ai-cost-protection-gate.js` são o mesmo padrão duas vezes (inconsistência local, não um problema espalhado pelo servidor — a varredura exaustiva de `catch (_)`/`catch (e) {}` em `src/` não encontrou um terceiro caso comparável fora de fallbacks de parsing legítimos).
4. **Primitivo interno duplicado como rota pública sem os mesmos guardas** — único caso genuíno de "duas autoridades para a mesma responsabilidade" encontrado no servidor (créditos: caminho interno orquestrado vs. rota pública crua). Não encontrado padrão equivalente em auth, parsing de corpo HTTP, ou cache.
5. **Resultado assíncrono aplicado sem checar se a operação ainda é a atual — a mesma classe do bug do aquecimento, encontrada de novo e corrigida de novo** — `speakAuxRoomContent` em `lab_session_media_controller.dart` tinha exatamente o mesmo defeito do aquecimento (bug original desta sessão), só que em áudio de sala auxiliar em vez de aquecimento, e a correção certa (contador de operação) já existia duas funções ao lado no MESMO arquivo (`toggleAudio`). Isso confirma que vale a pena procurar horizontalmente por uma família de bug mesmo depois de "resolvida" — a causa raiz (falta de guarda) pode reaparecer em outro lugar do mesmo arquivo/módulo sem ser a mesma linha de código.
6. **Cadeia sem cancelamento nem idempotência do toque do usuário até a resposta do servidor** — encontrada uma vez, na cadeia de upload de anexo da Dúvida (rede sem chave de idempotência + widget sem guarda `mounted` + `ChangeNotifier` sem guarda de staleness, em 3 arquivos diferentes). Não encontrada uma segunda ocorrência completa dessa cadeia tripla em outro fluxo de upload/anexo do produto — mas não foi feita uma varredura exaustiva de todo `lib/sim/media/` para confirmar ausência total.

## Testes executados

**APP** (`/root/worktrees/sim109-nplus1-scroll` → `/root/BOM`, HEAD `190138d`):
- `flutter analyze` — limpo em todos os arquivos tocados.
- `flutter test` — suite completa, **1434/1434 testes passando**, rodada de novo após CADA uma das 4 mudanças de código desta sessão (fix do aquecimento, log de diagnóstico, `allowBackup`, guarda de `speakAuxRoomContent`) — nenhuma regressão introduzida em nenhuma etapa.
- Validação de manifesto Android como XML bem-formado após o `allowBackup`.

**SERVER** (`/root/Servidor-BOM`, HEAD `8ca0e91`):
- `node --check` no arquivo editado.
- 4 testes de contrato mais diretamente relevantes rodados isoladamente primeiro (`ai_cost_protection_mandatory_law`, `generic_transient_ai_failure_recovery_contract`, `rwr001_economic_provider_failure_taxonomy_contract`, `t02_reservation_release_on_provider_failure_contract`) — todos passando.
- Suite completa: `npm test` → **102/102 arquivos de teste passando**.

## Evidência física
Nenhuma validação em dispositivo físico foi realizada NESTA sessão de auditoria (o teste no tablet Samsung Galaxy Tab A9 SM-X216B do fix do vazamento de conta já havia sido solicitado e um APK de debug já foi instalado em sessão anterior, aguardando confirmação do Joel — não repetido aqui). **NOT VERIFIED** em produção real para as mudanças desta sessão especificamente.

## Riscos remanescentes

- Os 9 achados não corrigidos listados acima permanecem no produto como estão hoje.
- O endpoint `/api/credits/reserve|capture|refund` continua acessível publicamente sem vínculo a produto real — risco P1 vivo até a investigação de uso em produção ser feita.
- Ausência de reconciliação server-side para compras Google Play não reconhecidas por falha do cliente — risco P1 vivo, mitigado apenas pelo comportamento cooperativo do cliente atual.
- Nenhuma validação em dispositivo físico das mudanças desta sessão.

## Estado de release

**Não declaro esta base como pronta para produção.** O AAB `1.0.0+110` já gerado em sessão anterior contém o fix do vazamento de conta mas NÃO contém as 4 correções desta auditoria (log de diagnóstico, `allowBackup`, fix do servidor). O fix de servidor (`8ca0e91`) já está em `main` mas não foi implantado no droplet de produção nesta sessão — produção continua rodando `7bbc8fa`. Recomendo: (1) confirmar o teste adversarial de troca de conta no tablet físico (já solicitado ao Joel, pendente), (2) decidir sobre os achados P1 registrados antes de gerar um novo AAB, (3) fazer deploy do fix de servidor quando conveniente (é aditivo/observabilidade pura, baixo risco, pode ir independente do app).

## Backlog técnico recomendado (ordenado)

**P0** — nenhum item aberto (o único P0 desta sessão, vazamento de conta, foi corrigido e testado).

**P1**:
1. Ler `src/app/media-cache.js` e confirmar se o registro de replay de idempotência do controlador de imagem escopa por `userId` — ação mais barata e mais urgente desta lista, é só uma leitura de acompanhamento.
2. Investigar uso real em produção de `/api/credits/reserve|capture|refund` antes de decidir travar ou remover.
3. Implementar reconciliação server-side de compras Google Play não reconhecidas.
4. Habilitar R8/ProGuard no build de release Android (com regras testadas e validação em dispositivo físico).
5. Consumir compras consumíveis via API do Google no servidor, como defesa em profundidade ao lado do client-side já existente.
6. Corrigir a cadeia de upload de anexo (Dúvida): idempotência ausente + `_pickImage` sem `mounted` + `entry_form_state.dart` sem guarda de staleness.
7. Decidir e resolver a amplificação multiplicativa de retry (gate externo × T02/imagem interno, até 9 chamadas por crédito).

**P2**:
1. Adicionar guard de posse (`sim_ownership_records`) em `/api/student-state/persist`.
2. Migrar sessão Supabase para `flutter_secure_storage`.
3. Decidir e documentar explicitamente a política fail-open do `checkSimMandatoryUpdate()`.
4. Integrar Real-time Developer Notifications do Google Play para compras pendentes.
5. Asserção de boot travando `DURABLE_LEDGER_MODE` para fora de modo só-Redis em produção.
6. Remover `network_security_config_cleartext.xml` do variant de release via resource set dedicado.
7. Adicionar backoff ao loop de regeneração de reserva em `credits-store.js`.

**P3**:
1. Adicionar `requestId` ao log de diagnóstico de erro de preparação de aula.
2. Mover `AI_COST_T02_LEGACY_RESULT_RECOVERY` para a camada central de config.
3. Remover dependência morta `sqlite3_flutter_libs`.
4. Remover classe morta `LessonDoubtController` (não instanciada em lugar nenhum) de `lib/sim/auxiliary/lesson_doubt_controller.dart`.
5. Normalizar as proteções de staleness incidentais (Dúvida, Recuperação, Revisão) para checagens de geração locais e explícitas, por consistência — não por haver bug hoje.
6. Trocar `operationId` de Amparo para chave content-derived em vez de `DateTime.now()`.

## Conclusão

Esta auditoria confirmou, com evidência direta de código/log/teste/RPC de banco, que a arquitetura do produto é **majoritariamente madura e alinhada com padrões estabelecidos** nos pontos mais sensíveis já auditados: autenticação (JWT com validação completa de claims + JWKS), gestão de estado assíncrono (guarda de staleness aplicada de forma consistente em praticamente todo o app), armazenamento local (escopo por conta na própria chave física), sincronização (transação centralizada com múltiplas guardas de regressão), e uso de Redis (fail-closed, não autoritativo para dinheiro). O único desvio P0 real encontrado — o vazamento de aquecimento entre contas — já estava sendo corrigido quando esta auditoria começou, e foi confirmado como uma EXCEÇÃO à disciplina do resto do código, não um padrão sistêmico.

A auditoria também encontrou uma cadeia de achados genuínos, com evidência, na camada de billing/créditos (endpoint público sem vínculo a produto, ausência de reconciliação de compras) que não existiam antes desta missão e que exigem decisão do Joel antes de qualquer ação — não foram corrigidos às cegas, conforme a regra de ouro desta missão.

**Não declaro o produto "100% padronizado" nem "pronto para produção"** — essas afirmações exigiriam validação física e uma decisão humana sobre os achados P1 registrados que esta sessão não teve autoridade para tomar sozinha.
