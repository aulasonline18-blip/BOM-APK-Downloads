# SIM109 — Correção do bug de imagem + implementação de consumo de créditos

**Data:** 2026-09-18
**Branch:** `reform/sim109-economic-final-construction` (app e server)
**Commits:**
- Server: `a371f97` — feat(credits): add signup bonus and fix durable-ledger 402 mapping
- App: `8ffae46` — fix(classroom): show friendly credit-exhausted message on advance failure

---

## 1. Bug do canal de imagem (nenhuma imagem apareceu na aula)

**Causa raiz:** não era um bug de código. O servidor de teste não tem credenciais reais de
object storage (S3/R2), então cai no modo de armazenamento fake em memória, cujas URLs
assinadas apontam para `https://storage.invalid/...` — um host de placeholder deliberado.
O app corretamente recusa renderizar esse host (proteção contra URL inválida/insegura).

**Correção:** nenhuma mudança de código. Adicionada a variável
`MEDIA_OBJECT_STORAGE_INLINE_DEV_FALLBACK=true` no ambiente do servidor de teste
(`/root/sim109-test-server.env`, fora de qualquer repositório git). Essa flag só tem efeito
quando o backend de storage é o fake em memória — nunca em produção, que sempre configura
S3/R2 real. Com ela, o storage fake retorna também uma `data:` URL base64 real, que o app
consegue renderizar.

**Teste físico no tablet:** aula "Sistema Solar" recriada do zero, 3 imagens distintas
confirmadas renderizando corretamente em 2 itens diferentes. Suite de 98/98 testes
automatizados do servidor permaneceu verde.

**Efeito colateral descoberto durante o teste de créditos:** essa mesma flag, ao fazer o app
embutir uma `data:image/...` na resposta de visual-route, colide com uma trava arquitetural
real e intencional do servidor (`STUDENT_STATE_EMBEDDED_BLOB_FORBIDDEN` em
`student-state-lean.js`), que rejeita qualquer blob embutido persistido no estado do aluno.
Isso só acontece no ambiente de teste (que usa o storage fake) — nunca em produção. Ficou
registrado como comportamento conhecido, não corrigido (está fora do escopo do bug relatado
e não afeta produção).

---

## 2. Consumo de créditos (nova feature)

### O que o SIM Web realmente faz (verificado no código-fonte, não assumido)

Lido diretamente: `credits.functions.ts`, `route-credits.server.ts`, migrations SQL mais
recentes (`charge_lesson_generation_once`, `charge_image_generation_once`, `handle_new_user`).

- **3 créditos por aula gerada** (uma vez, idempotente por `lesson_local_id`)
- **10 créditos por imagem gerada** (idempotente por chave própria)
- **9 créditos de bônus no cadastro** (evoluiu de 3→5→9 ao longo das migrations)
- Bloqueio em HTTP 402 antes de qualquer chamada T00/T02 quando saldo ≤ 0

Isso diverge do que você descreveu ("1 crédito por item, 15 de bônus"). Perguntei via
pergunta direta qual modelo replicar — você escolheu manter o modelo que descreveu
originalmente (1 crédito por item, 15 de bônus), não o do SIM Web.

### O que já existia no servidor SIM109 (descoberto, não construído nesta sessão)

O consumo de **1 crédito por item de currículo carregado** já estava implementado e testado
(`T02_ITEM_CREDIT_COST`, `commercialItemCreditKey` em `complete-lesson-controller.js`), assim
como o bloqueio em saldo zero com erro amigável ponta a ponta (402 no servidor →
`classifyStudentExperienceError()` no app → "Seus créditos acabaram").

### O que faltava e foi implementado nesta sessão

**Bônus de 15 créditos em conta nova**, replicado nos três backends de crédito do servidor
(arquivo local, Redis distribuído, ledger durável via Supabase), concedido de forma idempotente
no primeiro toque da conta — inclusive quando esse primeiro toque é uma reserva de crédito
(não só uma leitura de saldo), para que o primeiro item de um aluno novo já possa ser cobrado.

**Correção real encontrada no caminho de produção/ledger durável:** `durable-ledger.js`
mapeava *toda* resposta de erro do RPC do Supabase para códigos genéricos
(`DURABLE_LEDGER_UNAVAILABLE`/500), inclusive quando o erro real era `insufficient_credits`.
Isso quebraria a detecção de crédito esgotado especificamente no backend de produção (durável).
Corrigido para mapear corretamente para `INSUFFICIENT_CREDITS`/402.

**Correção de UX no app:** ao tentar avançar para o próximo item de uma aula em andamento e a
cobrança falhar por falta de crédito, o app ficava preso indefinidamente em um botão
"Preparando próximo passo" desabilitado, sem nenhuma mensagem — o aluno não tinha como saber
o que aconteceu nem como agir. Corrigido para mostrar a mensagem amigável existente
("Seus créditos acabaram. Adicione créditos para continuar estudando.") com botão de tentar
novamente, reaproveitando a bolha de erro do sistema já existente na timeline do chat.

### Testes automatizados

- Servidor: novo arquivo `test/credits_signup_bonus_contract.test.js` cobrindo bônus (uma vez,
  por conta), consumo por item (idempotente, não cobra duas vezes no mesmo item), e bloqueio em
  saldo zero (402, `INSUFFICIENT_CREDITS`, mensagem menciona "credit"). Suite completa do
  servidor: **99/99 arquivos passando**.
- App: suite completa **1414 testes passando**, sem regressão, após a correção de UX.

### Teste físico no tablet (conta real `ccrfoodgy1@gmail.com`, sem saldo de teste)

1. Login com Google em conta nova → badge de créditos mostrou **⚡ 15** imediatamente.
2. Aula "Ciclo da água" criada → item 1 carregado → saldo caiu para **14** (confirmado na
   tela de Créditos do app).
3. Item 2 carregado → saldo caiu para **13** (confirmado no ledger no servidor; a tela de
   créditos do app mostrou valor levemente desatualizado nessa checagem específica — cache
   de UI, não é um bug de cobrança, o valor real no servidor estava correto).
4. Saldo zerado deliberadamente (via ledger, para acelerar o teste em vez de esgotar 13
   itens manualmente) → tentativa de carregar o próximo item → servidor respondeu
   `402 INSUFFICIENT_CREDITS` corretamente → após a correção de UX, o app mostra a mensagem
   amigável em vez de travar.

**Pendência conhecida (não corrigida nesta sessão):** o mesmo tipo de "trava sem mensagem"
que corrigi existe em pelo menos mais um lugar do app — o fluxo de entrada da *primeira* aula
(tela de aquecimento "Aguardando a primeira aula..."), que usa um subsistema diferente
(`lab_session_warmup_flows.dart`) do que corrigi (`lesson_material_controller.dart`, usado para
avançar dentro de uma aula já em andamento). Não having sido possível reproduzir e corrigir
esse caminho especificamente dentro do tempo desta sessão; fica registrado para uma próxima
correção pontual.

---

## Arquivos alterados

**Server** (`Servidor-BOM`, commit `a371f97`):
- `src/config/env.js` — nova config `SIGNUP_BONUS_CREDITS`
- `src/credits/credits-store.js` — bônus de cadastro nos 3 backends
- `src/durable/durable-ledger.js` — mapeamento correto de `insufficient_credits` → 402
- `src/errors/human-error.js` — mensagens humanas dedicadas para status 402
- `test/credits_signup_bonus_contract.test.js` — novo teste
- `test/mandatory-tests.manifest`, `test/server-contract.test.js` — atualizados

**App** (`BOM`, commit `8ffae46`):
- `lib/sim/classroom/lesson_material_controller.dart` — erro amigável em vez de trava
- `lib/sim/ui/sim_i18n.dart` — nova chave `aula_credits_exhausted`

Co-Authored-By: Claude Sonnet 5 <noreply@anthropic.com>
