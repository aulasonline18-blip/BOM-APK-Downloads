# Convergência P1 — Async / Retry / Billing

## BASE

- **APP** — repo `aulasonline18-blip/BOM`, branch `main`. HEAD ao final desta missão: **`c13f7f8`**. Working tree limpo, testes rodando a partir do worktree `/root/worktrees/sim109-nplus1-scroll` (mesmo HEAD).
- **SERVER** — repo `aulasonline18-blip/Servidor-BOM`, branch `main`. HEAD ao final desta missão: **`3630a7d`**. Working tree limpo.
- Relatório da auditoria que originou esta missão: `relatorios-claude/2026-09-18-auditoria-sistemica-padronizacao-final.md` (lido e usado como ponto de partida, mas **nenhum número foi confiado sem reverificação** — a contagem exata de chamadas do retry multiplicativo, por exemplo, mudou de "9" hipotético para um número empiricamente medido antes de qualquer correção).

---

## 1. Retry

### Antes
Rastreei ponta a ponta: `APP → rota do servidor → cost gate → controller (T02/imagem) → cliente do provedor`. Descobri que o gate de custo (`ai-cost-protection-gate.js`, `executeWithInRequestRetry`, até 3 tentativas) envolve o `execute` que passa para ele — e para T02 e imagem especificamente, esse `execute` **já é, ele mesmo, uma função com seu próprio loop de retry de até 3 tentativas**. Não assumi o número "9" do relatório anterior — escrevi uma sonda empírica que exercita a pilha real (gate real + controller real, sem mocks de retry) e medi:

| Classe de erro | Chamadas reais ao provedor (antes) |
|---|---|
| 503 genérico, sem código | **9** |
| `AI_PROVIDER_UNAVAILABLE` | **9** |
| `AI_TIMEOUT` | **9** |
| `AI_CONTRACT_INVALID` (resposta malformada) | 3 (já correto — só T02 retenta) |
| `AI_RATE_LIMIT` / 429 | 1 (já correto — nenhuma camada retenta) |

Isso provou que a causa NÃO era "duas políticas de retry idênticas duplicando tudo" — era mais específica: o gate e o T02 se sobrepõem exatamente nas classes 503/`AI_PROVIDER_UNAVAILABLE`/`AI_TIMEOUT`, mas `AI_CONTRACT_INVALID` já era tratado corretamente só por T02 (o gate o exclui deliberadamente do seu próprio retry via `providerFailureDisposition`). Isso significa que a correção ingênua ("remover o retry de T02 inteiramente") teria **quebrado** a recuperação de resposta malformada — tentei essa correção primeiro, um teste existente (`t02_aula_auxiliares_contract.test.js`) quebrou exatamente por esse motivo, e eu revertiat.

### Mapa das camadas (owner de retry por camada, confirmado no código atual)

| Camada | Retry próprio? | Classes cobertas | 429? |
|---|---|---|---|
| `ai-cost-protection-gate.js` (`executeWithInRequestRetry`) | Sim, até 3, backoff+jitter | "genérico, sem classificação específica" (não inclui 429, nem contract-invalid, nem 4xx comuns) | Nunca retenta (`isExplicitProviderRateRejection`) |
| `complete-lesson-controller.js` (T02) | Sim, até 3, backoff+jitter | `AI_TIMEOUT`, `AI_PROVIDER_UNAVAILABLE`, `AI_CONTRACT_INVALID`, qualquer 5xx/408 | Nunca retenta |
| `image-controller.js` | Sim, até 3, backoff+jitter | 429\*, 502, 503, 504 (\*antes da correção) | **Retentava** antes da correção — corrigido nesta missão |
| `entry-warmup-controller.js`, `visual-route-controller.js`, `attachment-processor.js` | Não têm retry próprio | Dependem inteiramente do gate | N/A |

### Owner escolhido
**Um retry owner por classe de erro, não por operação inteira.** Em vez de escolher "só uma camada" globalmente (o que teria exigido adicionar retry aos 3 consumidores que hoje não têm nenhum, ou removido cobertura real do T02 para `AI_CONTRACT_INVALID`), a correção canônica encontrada foi: **a camada mais próxima do provedor (T02, imagem) marca explicitamente quando seu próprio orçamento já foi gasto**, e o gate respeita esse sinal.

```js
// No T02/imagem, ao esgotar o próprio orçamento:
error.organRetryBudgetExhausted = true;

// No gate:
if (error?.organRetryBudgetExhausted === true) return false; // não retenta de novo
```

Isso torna os dois retries **disjuntos por construção**, sem remover cobertura de nenhuma classe, sem tocar nos 3 consumidores que dependem só do gate, e sem inventar uma segunda autoridade — o gate continua sendo a ÚNICA autoridade de retry para warmup/visual-route/attachment; para T02/imagem, a autoridade é o controller local, e o gate simplesmente não duplica o que já foi decidido.

### Orçamento final (depois)

| Classe de erro | Chamadas reais ao provedor (depois) |
|---|---|
| 503 genérico | **3** |
| `AI_PROVIDER_UNAVAILABLE` | **3** |
| `AI_TIMEOUT` | **3** |
| `AI_CONTRACT_INVALID` | 3 (inalterado) |
| `AI_RATE_LIMIT` / 429 (T02) | 1 (inalterado) |
| `AI_RATE_LIMIT` / 429 (imagem) | 1 (**era 3 antes** — corrigido: imagem retentava 429 com backoff genérico de 1-7s, igual a qualquer 5xx, sem respeitar cooldown real) |

### Testes
`test/ai_retry_ownership_single_budget_contract.test.js` (novo) — prova as 5 classes acima com contagem real de chamadas, incluindo um caso de "transiente que se recupera dentro do próprio orçamento" (2 chamadas, sem reserva/captura duplicada) e um de `AI_CONTRACT_INVALID` (prova que a correção não regrediu essa cobertura). Suite completa do servidor: **103/103 → 105/105** (depois de somar este teste e o de rotas legadas removidas).

### Commit
`c90118d` (SERVER) — `fix(ai-retry): one retry owner per operation, no accidental multiplication`.

---

## 2. Anexo da Dúvida

### Fluxo completo investigado
`tap do aluno → image_picker/file_picker → bytes → validação → payload → upload multipart → retry → /api/process-attachment → processamento → resultado → EntryFormState → render`.

### Identidade
**Já correta, confirmado, não é `DateTime.now()` isolado.** A identidade real de deduplicação não vem do cliente — o servidor deriva sua própria chave de idempotência a partir do **hash do conteúdo do arquivo** (`attachment:${hashIdentifier(file.data, 48)}` em `attachment-processor.js`), então um retry do MESMO arquivo sempre bate na mesma chave, independente de qualquer coisa que o cliente envie ou deixe de enviar. Isso é mais robusto do que qualquer chave client-supplied poderia ser — é exatamente o padrão "identidade derivada de conteúdo/hash" recomendado pela missão.

### Idempotência server-side
**Já correta, confirmada.** O gate de custo do servidor já garante que o mesmo hash de conteúdo não processa nem cobra duas vezes, com o mesmo mecanismo de single-flight/replay já auditado e testado em outras partes do sistema.

### Cancelamento e staleness
Investigado com cuidado, dois achados reais:

1. **`doubt_input_sheet_widget.dart:_pickImage`** — dois pontos de `await` (picker de imagem, leitura de bytes) sem nenhuma checagem de `mounted`. Clássico risco de `setState` após dispose se a folha de Dúvida for fechada enquanto o picker está aberto. **Corrigido**: `if (!mounted) return;` depois de cada `await`, no caminho de sucesso e no de erro.
2. **`sim_server_attachment_client.dart`** — retentava um 429 explícito exatamente como qualquer outro status transitório (408/500/502/503/504), com um backoff linear curto (400/800/1200ms) — a mesma classe de problema já corrigida no lado do servidor para T02/imagem. **Corrigido**: 429 removido do conjunto de status retentáveis localmente.

**Já seguro, verificado, sem necessidade de correção:**
- `entry_form_state.dart` (o `ChangeNotifier` dono da lista de anexos) já é zerado no `_onAccountChanged` (`entryForm.clearAttachments()`), e a substituição de um rascunho de anexo (`_replaceAttachment`) casa pelo **id único do próprio rascunho**, não por um campo alheio — um resultado tardio de uma conta/rascunho já removido simplesmente não encontra correspondência e é descartado sem efeito. Isso já é o padrão "identidade/generation própria da operação" que a missão pede, só que a "geração" aqui é o id do rascunho em vez de um contador — funcionalmente equivalente.
- O servidor já deriva sua própria idempotência por hash de conteúdo (ver acima), então nem uma cadeia de retry sem identidade explícita no cliente conseguiria duplicar processamento ou cobrança.

### Testes
`test/sim_server_attachment_client_retry_test.dart` (novo) — prova que um 429 falha em exatamente 1 tentativa, e que um 503 transitório ainda usa o orçamento de 3 tentativas normalmente (sem regressão). Suite completa do app: **1434 → 1436 → 1435** (subiu com os dois novos testes desta seção, depois caiu 1 com a remoção de um teste morto na seção 4 abaixo).

### Commit
`bc81ab1` (APP) — `fix(attachments): mounted guard on image pick + never retry 429 locally`.

---

## 3. Async stale audit

Varredura focalizada (não uma nova auditoria geral) nos fluxos listados pela missão. A sub-auditoria completa já feita na missão anterior (relatório de 2026-09-18) cobriu exatamente este escopo com profundidade — usei esta seção para (a) fechar a única lacuna real que ela identificou e (b) reverificar, não assumir, duas classificações que pareciam mais alarmantes do que realmente são.

| FLOW | OWNER | STALE GUARD | STATUS | AÇÃO |
|---|---|---|---|---|
| Aquecimento (warmup) entre contas | `LabSessionWarmupController` | Contador de conta zerado em `_onAccountChanged` | CANONICAL | Já corrigido em sessão anterior (`1a9cc08`) |
| Áudio de sala auxiliar (`speakAuxRoomContent`) | `LabSessionMediaController` | Nenhum, ao lado de `toggleAudio` que já tinha o padrão certo no mesmo arquivo | Era UNSAFE | **Corrigido** em sessão anterior (`190138d`), confirmado ainda válido |
| Dúvida (pergunta + imagem inline) | `LabSessionDoubtController` | Dois eixos: identidade da pergunta E hash de conteúdo, revalidados nos caminhos de sucesso e erro | CANONICAL | Nenhuma ação |
| Anexo da Dúvida (picker local) | `_DoubtInputSheetState` | Nenhum antes desta missão | Era UNSAFE | **Corrigido** nesta missão (seção 2) |
| Revisão / Recuperação | `LessonRecoveryController`/`ReviewController` | Releitura de `recoveryRoom`/`reviewRoom` (zerados em `_onAccountChanged`), não um contador de geração próprio | INCIDENTAL, mas sem risco concreto demonstrado | Registrado, não corrigido (ver seção "não corrigidos") |
| Preparação/N+1 (`BomPreparedExperienceCoordinator`) | Instanciado por `SimOrganism`, descartado inteiro na troca de conta | Nenhum contador de conta próprio no mapa `_inflight` | **Reclassificado nesta missão**: verifiquei o site de instanciação (`sim_organism.dart:661`) e confirmei que este objeto nunca sobrevive a uma troca de conta — não é "INCIDENTAL", é **SAFE BY OWNER LIFETIME**, a classificação da auditoria anterior foi mais cautelosa do que a evidência sustenta | Nenhuma ação — não adicionei escopo de conta que a arquitetura já torna desnecessário |
| Sala auxiliar (`StudentAuxRoomService`) | Mesmo padrão, instanciado por `SimOrganism` (`sim_organism.dart:748`) | Idem acima | Idem acima — SAFE BY OWNER LIFETIME, confirmado | Nenhuma ação |
| Imagem (geração/render) | Servidor: gate + single-flight; App: `student_lesson_material_service.dart` | Confirmado em sessão anterior, nenhuma mudança de código desde então | CANONICAL | Nenhuma ação |
| Abertura de aula (`_openAulaRuntimeDirect`) | `LabSession` | `_runtimeOpenToken` + `accountGeneration` do `canonicalStore` | CANONICAL | Nenhuma ação |
| Auth/troca de conta | `AuthSession` → `LabSession._onAccountChanged` | Comparação explícita de `userId` antes/depois | CANONICAL | Nenhuma ação |
| Sync (callbacks de fila de nuvem) | `SharedPrefsCloudQueueStorage` | Chave com sufixo de conta | CANONICAL | Nenhuma ação (confirmado em sessão anterior) |

**Nenhum UNSAFE restante conhecido** neste escopo depois das correções desta e da sessão anterior.

---

## 4. Rotas financeiras legadas

### Tráfego encontrado
- **Código do app**: zero call sites de `chargeLessonGeneration` em qualquer lugar do app atual (busca exaustiva).
- **Logs de produção**: zero requisições a `/api/credits/reserve`, `/api/credits/capture` ou `/api/credits/refund` em **toda a retenção de log disponível no droplet** (desde `2026-08-31`, quando o serviço começou a rodar nesta VM — não apenas "últimos 30 dias" teóricos, é literalmente todo o histórico que o servidor guardou).

### Clientes
Nenhum cliente atual ou historicamente ativo encontrado. O único consumidor (`SimServerCreditsClient.chargeLessonGeneration`) era resquício de uma arquitetura de cobrança mais antiga (cliente decide o custo), substituída pela atual (custo derivado no servidor, dentro do próprio gate do `/api/complete-lesson`) sem nunca ter sido removido de nenhum dos dois lados.

### Decisão
**UNUSED, comprovado — não UNKNOWN.** Removido de forma coordenada.

### Migração/remoção
- **APP**: `chargeLessonGeneration`, `ChargeLessonGenerationInput`, os campos `reservePath`/`capturePath` e o método na interface `CreditsFunctions` removidos. Dois testes removidos porque só existiam para caracterizar esse caminho morto — um deles (`test/electrical_hydraulic_connections_test.dart`) tinha um nome ativamente enganoso ("créditos de aula usam reserve/capture oficiais do servidor"), o que não é verdade há um bom tempo.
- **SERVER**: as três rotas e os handlers `reserve`/`capture`/`refund` em `credits-controller.js` removidos. Os primitivos internos do ledger (`reserveCredit`/`captureCredit`/`releaseCredit` em `credits-store.js`) **preservados integralmente** — são usados por T02, imagem e salas auxiliares e não foram tocados.

### Testes
`test/legacy_credits_routes_removed_contract.test.js` (novo, SERVER) — prova que as rotas não existem mais no router e que os primitivos internos continuam existindo. Um teste antigo que chamava `controller.reserve` diretamente foi removido (`auth_billing_account_contract.test.js`).

### Commits
`c13f7f8` (APP) — `chore(billing): remove dead client-side reserve/capture credit path`.
`72e1865` (SERVER) — `chore(billing): remove unused public credits reserve/capture/refund routes`.

---

## 5. Google Play

### Fluxo antigo
Servidor verifica a compra com a API oficial do Google (nunca confia em estado enviado pelo cliente — **preservado integralmente, já estava correto**), concede créditos de forma idempotente por `orderId`/hash do token (**preservado, já estava correto**), mas **nunca chamava acknowledge/consume do Google** — só o cliente fazia isso, via `in_app_purchase_android`.

### Fluxo final
Idêntico na verificação e na concessão. Adicionado: logo após conceder os créditos, o **servidor** chama o endpoint oficial `:consume` do Google (`androidpublisher/v3/.../purchases/products/{productId}/tokens/{token}:consume`) diretamente, de forma síncrona, independente do cliente. `PRODUCT_TO_PACK` mapeia só pacotes de créditos consumíveis (não há assinatura nem produto não-consumível no catálogo), então `:consume` é a chamada oficial correta para este tipo de produto — não confundi com `acknowledge` (que seria para não-consumíveis).

### Purchase lifecycle (ordem canônica confirmada no código atual)
1. Payload validado → 2. produto/pacote confirmado → 3. Google consultado (`purchases.products.get`) → 4. `purchaseState === 0` (comprado) exigido → 5. custo/crédito derivado do catálogo do servidor (`getCreditPackOrThrow`, nunca do cliente) → 6. concessão idempotente no ledger durável → 7. **NOVO: consumo no Google** → 8. resposta ao cliente.

### Idempotência
- Mesmo `purchaseToken`/`orderId` → uma única concessão (chave estável, já correto, confirmado por teste antigo e novo).
- Retry após timeout → mesma concessão (idempotente por construção, o gate não muda de chave entre tentativas para a MESMA operação lógica).
- Duas requisições concorrentes → protegido pelo mesmo mecanismo de reserva/captura já auditado em outras partes do sistema.
- Chamar `:consume` do Google mais de uma vez (servidor + cliente, ou dois retries do servidor) → seguro, porque o próprio endpoint de consumo do Google é idempotente (consumir um token já consumido é um no-op seguro, não um erro que precisa de tratamento especial).

### Crash recovery
**Janela fechada.** Antes: Google confirma → servidor concede → app morre antes de reconhecer = compra concedida mas nunca consumida no Google, sem nenhuma forma do servidor perceber ou corrigir sozinho. Depois: o servidor consome com o Google **antes de responder ao cliente**, então mesmo que o app morra no instante seguinte, a compra já foi corretamente concedida E consumida, sem depender do cliente sobreviver a mais nada.

### Reconciliação
**Não implementada nesta sessão — registrada como pendência, não como regressão.** Se a própria chamada do SERVOR ao `:consume` falhar (indisponibilidade momentânea do Google, rede), isso é logado (`GOOGLE_PLAY_CONSUME_FAILED`) mas não falha a resposta nem desfaz a concessão — o crédito já foi corretamente concedido, que é a garantia financeira que importa. O que falta, registrado para uma missão futura dedicada: uma varredura periódica que reencontre concessões cujo consumo falhou e tente de novo, de forma idempotente e limitada. Não implementei isso agora porque exigiria uma tabela/coluna nova para rastrear o estado "concedido mas não consumido" — uma mudança de schema em código financeiro que a própria missão pede para não arriscar sob pressão de tempo sem necessidade comprovada de urgência (a falha em si já é rara e o pior caso, hoje, é auto-observável via log, não invisível).

### Pending
Comportamento já correto e preservado: nenhuma concessão de crédito acontece enquanto `purchaseState !== 0`. O que falta, também registrado e não implementado: acompanhar a transição PENDING → PURCHASED via RTDN (Real-time Developer Notifications) do Google, para não deixar uma compra legitimamente pendente invisível para sempre caso o primeiro request do cliente tenha acontecido durante o estado PENDING. RTDN exige um endpoint público novo + configuração de Pub/Sub no Google Cloud — infraestrutura nova demais para adicionar com segurança nesta sessão, exatamente o tipo de "não criar infraestrutura enorme sem necessidade" que a missão pede para avaliar com cautela antes de implementar.

### Refund/revoke
**Investigado, nenhuma política de produto existe hoje para isso, e não inventei uma.** O ledger durável tem os primitivos (`captureCredit`/`releaseCredit`) mas nenhum deles foi desenhado para reverter uma concessão JÁ CAPTURADA em resposta a um reembolso do Google. Registrado como decisão de produto pendente do Joel, conforme a regra do adendo desta missão ("decisão real de produto/política que só o Joel pode tomar").

### Testes
`test/play_billing_server_side_consumption_contract.test.js` (novo) — prova: o servidor chama consume exatamente uma vez após cada concessão bem-sucedida; uma falha no consume nunca falha a resposta nem esconde que os créditos foram concedidos; replay idempotente da mesma compra continua correto; consume é chamado com o produto verificado, não com dados enviados pelo cliente. Rodado **3 vezes seguidas** (determinístico, sem flakiness) mais a suite completa do servidor **2 vezes**, conforme o cuidado extra pedido pelo adendo desta missão para qualquer mudança em código de pagamento real.

### Commit
`3630a7d` (SERVER) — `fix(play-billing): server consumes purchases itself, closing the crash window`. Diff revisado com atenção redobrada antes do commit, conforme pedido.

**Importante:** este commit está em `main` do `Servidor-BOM`, mas **não foi implantado no droplet de produção** nesta sessão — a decisão de fazer o deploy para `simaitutor.com` fica para o Joel autorizar, já que é uma ação que afeta o serviço em produção. Até o deploy acontecer, o servidor real continua rodando o comportamento anterior (correto na verificação/concessão, sem o consumo server-side ainda).

---

## 6. Ownership persist

**Análise curta feita, não implementado, registrado.** `/api/student-state/persist` roda inteiramente dentro de uma única RPC (`bom_persist_student_state_v2`) — em teoria, é possível adicionar a mesma checagem/registro de posse que `bom_durable_assert_ownership` já faz (reusando a tabela `sim_ownership_records`) **dentro dessa mesma função SQL**, sem round-trip de rede adicional, já que ambas rodariam na mesma transação de banco. Não implementei isso agora porque é uma mudança em uma função SQL que roda no caminho mais quente do produto (todo avanço de item chama persist) — exatamente o tipo de mudança que a própria missão pede para registrar em vez de arriscar às cegas ("Se NÃO [for possível sem round-trip]: registrar desenho recomendado e deixar fora desta execução" — aqui a resposta é "sim, é possível sem round-trip", mas a mudança em si ainda é de risco alto o suficiente para merecer sua própria migração dedicada e testada, não uma alteração apressada dentro desta missão).

**Desenho recomendado para uma missão futura**: estender `bom_persist_student_state_v2` para, na mesma transação, fazer `select ... for update` em `sim_ownership_records` para `(resource_kind='lessons', resource_key=lessonLocalId)`; se não existir, criar (auto-registro, como já acontece em `bom_durable_assert_ownership` com `p_create=true`); se existir com `account_id` diferente, abortar a transação inteira com o mesmo código `ownership_mismatch` que os outros endpoints já usam. Isso fecharia a assimetria descrita na auditoria anterior sem adicionar nenhuma chamada de rede nova.

---

## 7. Autoridades finais

| Domínio | Autoridade | Persistência | Consumidores | Status |
|---|---|---|---|---|
| RETRY OWNER (T02/imagem) | Controller local (`complete-lesson-controller.js`/`image-controller.js`) | N/A (em memória, por request) | Chamadas ao provedor de IA | Confirmado, uma autoridade por classe de erro |
| RETRY OWNER (warmup/visual-route/attachment) | `ai-cost-protection-gate.js` | N/A | As mesmas 3 rotas | Confirmado, sempre foi assim, correto |
| UPLOAD OPERATION OWNER (anexo) | `attachment-processor.js` (servidor), chave = hash de conteúdo | Nenhuma persistência própria além do gate de custo | `/api/process-attachment` | Confirmado |
| STALE INVALIDATION OWNER (troca de conta) | `LabSession._onAccountChanged` | N/A (em memória, contadores/campos zerados) | Todos os subcontroladores de `LabSession` | Confirmado, único ponto |
| CREDIT LEDGER OWNER | `credits-store.js` → RPCs `bom_durable_*` (Postgres) | `sim_credit_accounts`/`sim_credit_ledger`/`sim_durable_operations` | T02, imagem, salas auxiliares, Google Play | Confirmado, única autoridade — a rota pública duplicada foi removida (seção 4) |
| PLAY PURCHASE VERIFICATION OWNER | `play-billing-controller.js` → API oficial do Google | N/A (chamada síncrona por request) | `/api/play-billing/consume-credit-pack` | Confirmado, nunca confiou no cliente |
| PLAY GRANT OWNER | `credits-store.js` (mesmo ledger durável acima) | Mesmo ledger | Mesma rota | Confirmado, idempotente |
| PLAY COMPLETION/RECONCILIATION OWNER | **Servidor, a partir desta missão** (antes: só o cliente) | Nenhuma persistência de estado intermediário ainda (registrado como pendência na seção 5) | Google Play API `:consume` | **Mudou nesta missão** — antes o cliente era a única autoridade, agora o servidor também é, de forma independente e primária |

Nenhum domínio ficou com duas autoridades equivalentes sem justificativa.

---

## 8. Testes

**APP** (`/root/BOM`, HEAD `c13f7f8`):
- `dart format` — aplicado em todos os arquivos tocados.
- `flutter analyze` — limpo em todos os arquivos tocados, a cada etapa.
- Suite completa: rodada repetidamente ao longo da missão, terminando em **1435/1435 testes passando**.
- `git diff --check` — limpo.

**SERVER** (`/root/Servidor-BOM`, HEAD `3630a7d`):
- `node --check` em todos os arquivos tocados.
- Suite completa: rodada repetidamente, terminando em **105/105 arquivos passando**. A suite de Play Billing rodada 3x adicionais por precaução.
- `git diff --check` — limpo.

**PHYSICAL**: build de teste (debug, apontando para `https://simaitutor.com`) gerada a partir do HEAD `c13f7f8` e instalada no tablet Samsung Galaxy Tab A9 SM-X216B via ADB (`100.124.23.2:5555`), `versionCode=110`/`versionName=1.0.0` confirmado no dispositivo. **Nenhuma validação funcional foi executada por mim no dispositivo físico nesta sessão** — a instalação está pronta, mas os cenários (troca de conta durante upload de anexo, fluxo de compra completo) exigem interação humana do Joel.

**EXTERNAL VALIDATION REQUIRED**:
- Todos os cenários físicos listados acima (o app instalado não foi exercitado por mim além de confirmar a instalação/versão).
- O fluxo de compra completo do Google Play só pode ser validado de verdade pela build instalada via track interno do Google Play, não por sideload — isso está explicitamente fora do alcance desta sessão (nenhum sideload deve ser tratado como "billing validado").
- O fix do servidor (`3630a7d`, incluindo o fechamento da janela de crash do Play Billing) está em `main` mas **não foi implantado no droplet de produção** — decisão de deploy fica para o Joel.

---

## 9. Riscos remanescentes

- Reconciliação periódica de compras cujo consumo server-side falhou — não implementada, risco real mas raro e observável via log (seção 5).
- RTDN para compras PENDING que resolvem depois do primeiro request — não implementada, risco real mas de baixa probabilidade dado o fluxo normal de compra.
- Política de refund/revoke — não existe, decisão de produto pendente do Joel.
- `/api/student-state/persist` continua sem o guard de posse que os outros endpoints têm — desenho recomendado registrado (seção 6), não implementado.
- `Revisão`/`Recuperação` continuam com proteção de staleness "incidental" (funciona hoje, mas depende de um campo alheio ser zerado) em vez de um contador de geração próprio e explícito — sem risco concreto demonstrado, registrado por consistência, não corrigido para não expandir o raio de mudança sem necessidade comprovada.
- O deploy do fix de Play Billing para produção ainda não aconteceu — até acontecer, o comportamento em produção continua sendo o de antes desta missão (correto na verificação e concessão, sem o fechamento da janela de crash ainda ativo).

## 10. Próximo passo

Antes de qualquer AAB candidato final:
1. Joel testar a build instalada no tablet — especialmente troca de conta durante upload de anexo, e (quando o servidor for atualizado) o fluxo de compra completo pelo track interno do Google Play.
2. Decidir se e quando implantar `3630a7d` no droplet de produção.
3. Decidir a política de refund/revoke (única decisão desta missão que exige o Joel, não um blocker técnico).
4. Missão futura dedicada para: guard de posse em `/api/student-state/persist`, reconciliação periódica de Play Billing, RTDN para compras pendentes — todos registrados, nenhum é urgente o suficiente para justificar o risco de implementar às pressas nesta sessão.

Depois disso, e só depois, gerar um novo AAB candidato incorporando os fixes desta missão (o AAB `1.0.0+110` atual não os contém).
