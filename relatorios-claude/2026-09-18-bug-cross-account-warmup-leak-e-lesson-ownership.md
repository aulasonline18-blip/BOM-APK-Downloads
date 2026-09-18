# BLOCKER: vazamento de estado entre contas (aquecimento) + erro ao abrir aula após limpeza de dados

Data: 2026-09-18
Commit da correção: `1a9cc08` (main do BOM)

## Resumo executivo

Os dois bugs relatados por Joel têm a **mesma causa raiz**. Bug 1 (aquecimento
da conta errada aparecendo após logout/login) já foi corrigido, testado e
publicado em `main`. Bug 2 (erro ao clicar na aula "Sistema Circulatório" após
limpar dados do app e logar de novo) **não é um bug separado** — é uma
consequência permanente, gravada no banco, do Bug 1 ter acontecido uma vez
antes da correção existir. A correção do Bug 1 impede que esse tipo de dado
corrompido volte a ser criado, mas o registro específico já corrompido
("cyber-1r2yb4g") continua existindo no Postgres e é inofensivo (o próprio
app já se autorrecuperou, ver evidência abaixo).

## Bug 1 — vazamento do aquecimento entre contas

**Causa raiz confirmada (não é rede, não é servidor):** `LabSession` é um
objeto único e de vida longa (`lib/features/session/lab_session.dart`). Ele
tem um handler `_onAccountChanged` que já limpava runtime, mídia, organismo
etc. ao trocar de conta — mas **nunca limpava os campos do aquecimento**
(`warmupLesson`, `warmupError`, `warmupSelectedAnswer`,
`warmupWaitingForOfficialLesson`) nem as gerações de aquecimento ainda em
andamento (`_warmupFlights`). Esses campos ficam soltos na instância e só
eram zerados quando o app era fechado e reaberto (o que recria o objeto do
zero) — exatamente o padrão que você descreveu.

**Correção:** novo método
`LabSessionWarmupController.resetForAccountChange()`
(`lib/features/session/lab_session_warmup_flows.dart`), chamado dentro do
próprio `_onAccountChanged` já existente, logo depois do reset do organismo.
Ele zera os 4 campos de aquecimento, cancela flights em andamento e reseta o
`WarmupBridgeCoordinator` (que correlaciona "aquecimento pronto" → "aula
oficial pronta").

**Prova:** `test/account_switch_state_isolation_test.dart` (novo), dois
casos:
1. Conta A com aquecimento pronto → troca para conta B → aquecimento da conta
   A precisa sumir. Confirmei que o teste falha (pega a regressão) removendo
   a correção temporariamente e rodando de novo.
2. Aquecimento da conta A ainda em geração quando a conta troca duas vezes
   seguidas (B, depois A de novo) → não pode voltar.

Suíte completa rodada depois da correção: **1434 testes, todos passando**.
`flutter analyze` limpo nos arquivos alterados.

## Bug 2 — erro ao abrir "Sistema Circulatório" depois da limpeza de dados

Sua suspeita de ligação com o problema de `sim_state_owners` fazia sentido
como hipótese, mas os logs de produção apontam para outra tabela do mesmo
sistema durável: `sim_ownership_records`. Reconstruí a cadeia inteira batendo
log de produção contra o banco (script `bom_durable_assert_ownership`, com a
service-role key, sem adivinhar nada):

1. Nos logs do `bom-api` no horário do seu teste (19:49–19:52 UTC de
   18/09), a conta exponencial tentou reabrir uma aula com
   `lessonLocalId = "cyber-1r2yb4g"`. `/api/student-state/get` respondeu
   **403** repetidamente, e o `/api/complete-lesson` logo em seguida
   respondeu **409 CREDIT_OPERATION_REQUIRES_RECONCILIATION**.
2. Consultei diretamente `bom_durable_assert_ownership` no Supabase de
   produção para esse `resource_key`: com o `userId` de exponencial dá
   `ownership_mismatch`; com o `userId` da conta `aulasonline18` (a mesma
   conta do exemplo da Cinemática) dá `owned: true`. Ou seja: esse
   `lessonLocalId` está permanentemente registrado no Postgres como
   pertencendo à OUTRA conta — não à exponencial.
3. Isso só é possível se, durante o vazamento do Bug 1 (antes da correção),
   o app tenha chegado a mandar para o servidor, autenticado como
   exponencial, uma requisição carregando esse `lessonLocalId` que na
   verdade era da sessão anterior (`aulasonline18`). O servidor criou aí um
   registro de posse (`sim_ownership_records`) preso à conta original, e ao
   mesmo tempo criou/atualizou um `student_states` para exponencial com esse
   mesmo id — daí ele aparecer no menu da exponencial como "aula que eu já
   tinha criado".
4. Quando você limpou os dados e logou de novo, o app buscou esse
   `lessonLocalId` do servidor (não era cache local — os dados tinham sido
   apagados) e tentou abri-lo. O servidor bloqueou corretamente (é dono de
   outra conta), e isso apareceu na tela como erro de conexão/servidor.
5. **O próprio app já se recuperou sozinho**: nos logs, 27 segundos depois
   do primeiro 403, o app chamou `/api/student-state/delete` para
   tombstonar esse `lessonLocalId` quebrado e recomeçou com um id novo
   (`cyber-al884f`), que funcionou perfeitamente (aquecimento OK, itens
   M0001/M0002/M0003 gerados com sucesso, 200 em tudo).

**Conclusão:** não existe um segundo bug para corrigir no código. É resíduo
de dado permanente causado pelo Bug 1 ter ocorrido uma vez, ao vivo, antes da
correção estar no ar. A correção do Bug 1 impede que esse tipo de
contaminação volte a acontecer (o `lessonLocalId`/coordenador de aquecimento
não sobrevive mais à troca de conta). O registro específico
`cyber-1r2yb4g` continua "preso" à conta antiga no banco, mas isso é
inofensivo — nenhuma conta vai gerar esse mesmo id de novo (é aleatório), e
o app já demonstrou que se recupera sozinho em segundos quando encontra um
id assim.

Sobre o texto exato do erro: não encontrei em nenhum lugar do código atual
(nem em nenhum idioma) a string literal "Servidor indisponível, tente
novamente" — essa frase existia no app até a build 75 e foi reescrita para
"Não consegui conectar agora. Tente novamente em instantes." faz tempo,
antes até da build 109 que você está testando. O sentido é o mesmo; a
transcrição deve ter aproximado o texto.

## O que falta antes de promover para produção

- [ ] Testar no tablet físico o cenário adversarial exato que você pediu:
  conta A gera aula, avança itens, logout **sem fechar o app**, login conta
  B, gerar aula nova, confirmar que aquecimento e menu mostram só o
  conteúdo de B — repetir algumas vezes.
- [ ] Gerar um novo AAB (versionCode 110) com essa correção, já que o
  build 109 publicado ainda tem o bug.
- Nenhuma ação de banco é necessária para o Bug 2.
