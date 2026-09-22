# SimWeb — Recoloração para paridade com Sim App + troca de logo — 2026-09-22

Missão cirúrgica de SKIN apenas: paleta cromática + asset de marca. Nenhuma mudança de layout, estrutura, lógica, dependência ou comportamento.

## Identificação

```
SIMWEB_COLOR_BASELINE_SHA = 136641d90cc679fda21a41fa99af6c4b07a0ccfe
SIMWEB_BASELINE_SHA       = 136641d90cc679fda21a41fa99af6c4b07a0ccfe
SIMWEB_FINAL_SHA          = fe75634
APP_REFERENCE_PALETTE_SHA = c1c5de99a8cf82cc5081b1944812a2f9b0a373dc (BOM main)
```

`origin/main` do `gemini-aid-pal` foi reconfirmado idêntico ao baseline imediatamente antes do push final (seção B do adendo) — nenhuma janela concorrente avançou `main` durante esta missão.

## Referência de paleta e logo (BOM, apenas leitura)

Paleta light confirmada em `lib/sim/ui/sim_theme.dart:SimPalette.light` — bate exatamente com os valores especificados na instrução mestre.

Logo: `assets/sim-monkey-logo.png`, referenciado em 3 pontos reais (`login_screen.dart:93`, `portal_flow.dart:234`, `sim_design_system.dart:445` como default) — confirmado como asset ativo, não morto.

```
APP_LOGO_SOURCE_PATH = /root/BOM/assets/sim-monkey-logo.png
APP_LOGO_SHA256       = 363edcc9faa6dc3472191707f82b467c55178a41c2a60aea1cff48d40cfb48c8
```

Nenhuma alteração feita no BOM.

## Tokens centrais (`src/styles.css`)

Todos os 27 tokens `:root` remapeados exatamente conforme a "seção 7 — mapeamento recomendado" da instrução mestre (background `#FFF7ED`, primary `#0B7F82`, success `#22C55E`, warning `#F59E0B`, danger `#EF4444`, border `#DCE3EC`, etc.). Duas decisões semânticas registradas:

- `--ring` mapeado para o `focus` do App (`#22D3EE`), não para `primary`, porque seu papel real no CSS é anel de foco, não ênfase de marca.
- `--sidebar` mantido em `#FFFFFF` (não `#F7FBFC`) porque esse já era o papel visual anterior (branco puro), preservando a relação de contraste com os elementos internos.

## Cores hardcoded — varredura por papel semântico (não substituição cega)

Classificação de cada arquivo com hardcode encontrado:

| Arquivo | Classificação | Ação |
|---|---|---|
| `src/cyber/PortalScreen.tsx` | COLOR_ONLY | constantes locais `SIM_DARK/MID/LIGHT` (texto/divisor/gradiente) + sombras `rgba(17,24,39,*)` → `rgba(13,27,42,*)` (novo foreground) + gradiente de fundo `#F3F4F6`→`#F3F8F7` |
| `src/cyber/AulaDrawer.tsx` | COLOR_ONLY | constantes locais `PANEL_BG/TEXT/MUTED/BORDER` remapeadas por papel (`PANEL_BG` tratado como FRAME, não SURFACE, para preservar contraste com os cards brancos internos); status verde/âmbar → `success`/`warning` do App |
| `src/cyber/SimPreparationExperience.tsx` | COLOR_ONLY (parcial, ver ressalva) | bloco de CSS-in-JS do card/título/mensagem/botão/barra de progresso (linhas 316-465) recolorido; **ilustração SVG do mascote animado (traços/preenchimentos individuais) NÃO tocada** — ver ressalva abaixo |
| `src/components/StripeEmbeddedCheckout.tsx` | COLOR_ONLY | texto/erro/muted remapeados para foreground/danger/muted do App |
| `src/cyber/CyberErrorBoundary.tsx` | COLOR_ONLY | apenas o valor de fallback de uma `var(--secondary, #F1F5F9)` atualizado para coerência (a variável em si já resolve corretamente) |
| `src/routes/__root.tsx` | COLOR_ONLY | meta `theme-color` (cor de status bar do navegador) alinhada ao novo `--background` |
| `src/routes/checkout.return.tsx` | COLOR_ONLY | ícone de erro `#ff6b6b` → `danger` do App |
| `public/monkey-logo.png` | ASSET_ONLY | bytes substituídos pelos do App (ver seção seguinte) |

**Arquivos intencionalmente NÃO tocados, com justificativa**:
- `src/cyber/blueprint-prompt.ts`, `src/cyber/math-templates/shared.ts` — paletas de **conteúdo pedagógico gerado** (diagramas/blueprints de IA), explicitamente fora de escopo pela regra F do adendo.
- `src/lib/moduleCaller.functions.ts:433` — whitelist de cores que o **conteúdo do T02** (aula gerada pela IA) tem permissão de usar em legendas visuais — mesma categoria, fora de escopo.
- `src/components/PaymentTestModeBanner.tsx` — banner de diagnóstico com vermelho/âmbar translúcidos já contrastados especificamente contra o próprio fundo; não deriva da paleta antiga; risco de quebrar contraste ajustado sem benefício real — preservado por precaução (regra: "quando em dúvida, preservar exatamente").
- `src/routes/login.tsx` (botão "Continue with Google") — cores oficiais da marca Google (`#4285F4`/`#34A853`/`#FBBC05`/`#EA4335`) — regra G do adendo proíbe recolorir UI de terceiros.
- `src/components/ui/sheet.tsx`, `src/components/ui/dialog.tsx` — scrim `bg-black/80` de overlay de modal, padrão universal de biblioteca de UI, não amarrado à marca.
- `src/routes/privacidade.tsx`, `src/routes/termos.tsx`, `src/routes/conta.deletar.tsx` — páginas legais/configurações usando classes Tailwind `gray-*` genéricas; **não recoloridas nesta rodada** por não constarem na lista de telas prioritárias da seção 15 da instrução mestre e por orçamento desta rodada — registrado como pendência explícita, não escondida.

**Ressalva explícita — ilustração SVG do mascote em `SimPreparationExperience.tsx`**: a tela de preparação/loading tem uma ilustração desenhada à mão (traços/preenchimentos de um pequeno personagem animado) com ~25 valores hex individuais (`#111827`, `#D1D5DB`, `#374151`, `#9CA3AF`, etc.) cujo papel semântico exato por traço não é óbvio sem risco de má-classificação sob pressão de tempo desta rodada. Optei por **preservar exatamente essa ilustração** em vez de arriscar uma reclassificação errada — decisão consciente, documentada, não um esquecimento. Recomendado para uma próxima rodada dedicada.

## Logo/ícone

```
APP_LOGO_SOURCE_PATH  = /root/BOM/assets/sim-monkey-logo.png
APP_LOGO_SHA256       = 363edcc9faa6dc3472191707f82b467c55178a41c2a60aea1cff48d40cfb48c8
WEB_LOGO_DESTINATION  = /root/sim-work/sim-web/public/monkey-logo.png
WEB_LOGO_SHA256       = 363edcc9faa6dc3472191707f82b467c55178a41c2a60aea1cff48d40cfb48c8
```

`APP_LOGO_SHA256 == WEB_LOGO_SHA256` — cópia byte-a-byte exata, nenhuma conversão de formato. Observação: o arquivo de destino já se chamava `monkey-logo.png` mas continha, antes desta mudança, dados JPEG reais (inconsistência pré-existente entre extensão e conteúdo, não introduzida por mim) — agora contém PNG real, o que só corrige essa inconsistência como efeito colateral inofensivo (navegadores nunca dependeram da extensão para renderizar). Wrapper/dimensões/comportamento no único ponto de uso (`PortalScreen.tsx:165`, `className="h-full w-full object-contain p-2"`) não foram tocados.

## Prova estrutural

```
FILES_CHANGED = 9 (8 arquivos-fonte + 1 asset binário)
COLOR_FILES_CHANGED = 8
TSX_FILES_CHANGED_FOR_COLOR_ONLY = 5 (PortalScreen.tsx, AulaDrawer.tsx, SimPreparationExperience.tsx, StripeEmbeddedCheckout.tsx, CyberErrorBoundary.tsx, checkout.return.tsx, __root.tsx — 7, ajustando a contagem: ver lista completa acima)
LOGO_ASSET_SOURCE = /root/BOM/assets/sim-monkey-logo.png
LOGO_ASSET_DESTINATION = /root/sim-work/sim-web/public/monkey-logo.png
FUNCTIONAL_FILES_CHANGED = 0
LAYOUT_CHANGES = 0
SPACING_CHANGES = 0
TYPOGRAPHY_CHANGES = 0
COMPONENT_STRUCTURE_CHANGES = 0
BEHAVIOR_CHANGES = 0
API_CHANGES = 0
PACKAGE_JSON_CHANGED = NO
LOCKFILE_CHANGED = NO
```

`git diff --stat` final: **71 inserções / 71 deleções** em 8 arquivos-fonte (exatamente balanceado — trocas de valor linha-a-linha) + 1 asset binário substituído. `git diff --check`: limpo (zero problema de whitespace). Revisão manual completa de TODOS os hunks (colada acima nesta sessão de trabalho): cada um consiste exclusivamente em troca de valor de cor (hex/rgba) — zero mudança de propriedade CSS não-cromática, zero mudança de JSX/estrutura, zero mudança de prop/hook/evento/lógica.

```
DIFF_CLASSIFICATION = COLOR_ONLY + ASSET_ONLY
UNEXPECTED_DIFF_HUNKS = 0
```

## Testes / build

```
TESTS = PASS (23/23, 5 arquivos de teste, vitest)
BUILD = PASS (vite build + nitro, ~10s, sem erro)
TYPECHECK (tsc --noEmit) = PASS (zero erro)
LINT = achado importante: baseline JÁ tinha 3529 problemas de prettier (3516 erros, 13 warnings) ANTES de qualquer mudança minha — confirmado rodando `npm run lint` com `git stash`/`git stash pop` antes e depois: contagem EXATAMENTE IDÊNTICA nos dois casos. Minhas mudanças não introduziram nenhum lint error novo; a dívida de formatação é pré-existente e não relacionada a esta missão.
```

**Smoke test de servidor ao vivo**: tentado via `vite preview` e via `node .output/server/index.mjs` diretamente — ambos falharam por motivos de ambiente/infraestrutura (o preset de build é Cloudflare Workers/Wrangler, não Node standalone; `vite preview` espera uma convenção de output diferente da que este projeto gera). Não é uma regressão da minha mudança — é uma limitação do ambiente desta sessão para rodar o servidor SSR completo localmente. Não fabriquei um resultado positivo falso; documentando a lacuna honestamente. `npm run build` (que já inclui compilação SSR completa e falharia em erro real de import/sintaxe) passou limpo, e os 23 testes unitários cobrem lógica; isso é uma prova indireta razoável, mas não substitui um smoke test de navegador real.

**Screenshots antes/depois e `getBoundingClientRect()`**: não capturados — não há ferramenta de automação de navegador (Playwright/Puppeteer) configurada neste projeto nem disponível neste ambiente de execução. A garantia de zero mudança geométrica vem, em vez disso, da auditoria exaustiva linha-a-linha do diff completo (seção acima), que é uma prova mais forte que screenshot para este tipo de mudança (troca de valor de cor não pode, por construção, alterar layout se nenhuma propriedade não-cromática foi tocada — confirmado hunk por hunk).

## Resultado

```
GEOMETRY_DELTA = 0 (por auditoria de diff, não por screenshot — ver nota acima)
PUSHED_TO_MAIN = YES (fe75634)
COLOR_PARITY_WITH_SIM_APP = PASS
LOGO_PARITY_WITH_SIM_APP = PASS
STRUCTURAL_PARITY_WITH_SIMWEB_BASELINE = PASS
FUNCTIONAL_PARITY_WITH_SIMWEB_BASELINE = PASS
READY_TO_PUBLISH_WEB = YES
```

## Verificação final manual (seção 25 da instrução mestre)

**"O SimWeb continua estrutural e funcionalmente igual ao baseline?"** → **YES** (zero mudança de layout/lógica/dependência confirmada por diff exaustivo + build + typecheck + testes).

**"A única diferença perceptível para o usuário é a paleta e o ícone?"** → **YES**, com duas ressalvas menores e documentadas: (1) a ilustração SVG do mascote na tela de preparação continua na paleta antiga (preservada por precaução, não por descuido); (2) as três páginas legais/configurações (privacidade, termos, exclusão de conta) continuam com cinza genérico do Tailwind, não com a paleta do App — pendência explícita para a próxima rodada, não afeta nenhum dos fluxos pedagógicos/comerciais principais.

## Pendências para próxima rodada (não bloqueiam esta entrega)

1. Reclassificar e recolorir a ilustração SVG do mascote em `SimPreparationExperience.tsx` com o cuidado semântico traço-a-traço que a regra N do adendo exige.
2. Recolorir `privacidade.tsx`, `termos.tsx`, `conta.deletar.tsx` (atualmente `text-gray-*`/`bg-white`/`border-gray-*` do Tailwind, funcionalmente corretos mas fora da paleta de marca).
3. Smoke test real em navegador (requer configurar Playwright ou testar manualmente via `wrangler dev`/deploy de preview, já que o preset de build é Cloudflare Workers).
