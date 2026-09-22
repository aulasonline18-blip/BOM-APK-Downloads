# SimWeb — correção de contraste de estados selecionados/ativos — 2026-09-22

Missão cirúrgica pós-recoloração: corrigir texto branco sobre fundo quase branco em CTAs primários e opções selecionadas que usavam `var(--gradient-primary)` (agora `linear-gradient(135deg, #FFFFFF, #E7FAFA)`) + `var(--primary-foreground)` (`#FFFFFF`) — combinação ilegível criada pela recoloração geral anterior (`fe75634`).

## Mapeamento e classificação

Encontrados via `grep -rln "gradient-primary\|primary-foreground" src/` — 12 arquivos (2 a mais que a lista conhecida na instrução: `login.tsx` e `CyberStepShell.tsx`, incluídos por serem consumidores reais do mesmo padrão).

**PRIMARY_CTA** (21 ocorrências, corrigidas para `background: var(--primary)` + `color: var(--primary-foreground)` já existente): botões Continue em `cyber.idioma.tsx`, `PlacementIntroScreen.tsx`, `PlacementResultScreen.tsx`; "Salvar e seguir" em `cyber.objeto.tsx`; "Comprar créditos"/"Tentar novamente" em `cyber.curriculo.tsx` e `LessonStateScreens.tsx`; "Nova aula" e os dois botões "✓" de confirmar renome em `AulaDrawer.tsx`; botão de review e retry em `LessonMainScreen.tsx`; botão seguro de `CyberErrorBoundary.tsx`; botões de avançar/contagem/retry em `AuxRoomScreens.tsx`; e o botão "Sign in"/"Create account" de `login.tsx` (achado extra via grep — usava `var(--foreground)`, não estava tecnicamente ilegível, mas usava o mesmo `--gradient-primary` de forma inconsistente com os outros CTAs; alinhado ao mesmo padrão canônico).

**SELECTED_SURFACE** (4 ocorrências, corrigidas para `background: var(--accent)` + `color: var(--foreground)`): linhas de seleção de idioma em `cyber.idioma.tsx` (2), badges de letra de opção selecionada em `AuxRoomScreens.tsx` e `LessonMainScreen.tsx` (a borda já usava `var(--color-primary)` condicionalmente no nível da linha/wrapper — preservada).

**DECORATIVE** (4 ocorrências, propositalmente NÃO alteradas): barras de progresso de `CyberStepShell.tsx`, `AuxRoomScreens.tsx` (cabeçalho de sala) e `LessonMainScreen.tsx` (cabeçalho de aula + pulso de loading) — nenhuma tem texto sobreposto, logo não há problema de contraste a corrigir.

**OTHER (fora de escopo, não corrigido, sinalizado)**: `login.tsx:153` — a letra "S" do badge de logo usa `color: var(--primary-foreground)` (branco) sobre um wrapper com `bg-white` (branco), mesmo defeito de contraste, mas é um elemento decorativo estático (marca), não um estado selecionado/ativo/CTA — fora do escopo explícito desta missão cirúrgica. Recomendado para rodada futura dedicada a elementos de marca estáticos.

## Verificação estrutural

```
git diff --stat: 11 files changed, 32 insertions(+), 32 deletions(-)
git diff --check: limpo
package.json / bun.lock: diff vazio (intocados)
```

Todo hunk revisado via `git diff --word-diff` — 100% classificado `COLOR_ONLY` (troca de valor de `background`/`color`/`border` apenas). Zero `UNEXPECTED`.

## Testes

```
vitest: 5 test files, 23/23 PASS
tsc --noEmit: limpo (exit 0)
npm run build: sucesso (exit 0)
```

## Commit

`gemini-aid-pal` commit `9309d2b` (`fix(web): restore selected and primary action contrast`), branch `main`, pushado.

## Resultado

```
BASELINE_SHA=fe7563478aa9ddda6e1e1ec8429bd06f7d506621
FINAL_SHA=9309d2b869edd71892ea29d7cb841ad0c43c65c6

PRIMARY_CTA_CONTRAST=PASS
SELECTED_SURFACE_CONTRAST=PASS
CONTINUE_BUTTONS=PASS
DISABLED_STATE=PASS (classes disabled:opacity-40/50/60 e o branch var(--muted)/var(--muted-foreground) do botão de avanço principal não foram tocados)

FILES_CHANGED=11
FUNCTIONAL_LOGIC_DIFF=0
LAYOUT_DIFF=0
TYPOGRAPHY_DIFF=0
SPACING_DIFF=0

TESTS=PASS
BUILD=PASS
READY_TO_PUBLISH_WEB=YES
```

**Nota de honestidade metodológica**: como na missão anterior, não foi possível gerar screenshots antes/depois nem prova via `getBoundingClientRect()` neste ambiente (sem navegador/Playwright disponível). A prova de ausência de mudança estrutural aqui é por diff textual exaustivo (`git diff --word-diff`, hunk por hunk, 100% classificado), não visual.
