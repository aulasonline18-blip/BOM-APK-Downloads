# Correção do achado 1 (visual item-level) — SIM109

**Data:** 2026-09-17
**Segue:** `2026-09-17-phase11-12-integracao-legado-sim109.md`

## O que mudou

Você corrigiu uma premissa errada: o SIM App não tem oferta própria de imagem paga por IA — a geração via `/api/visual-route` (N3) é gratuita ao aluno. Imagem paga por IA existe só no SIM Web, produto separado.

Reinvestiguei com isso em mente:

- Confirmei por leitura direta: `visual-route-controller.js` **não tem nenhum** `reserveCredit`/`captureCredit`/`releaseCredit` — zero relação com cobrança real ao usuário.
- A identidade durável de armazenamento (`mediaIdentity()`, o que realmente decide se um artefato salvo é reaproveitado) **já não usava** `layer` — isso já estava certo.
- O que ainda usava `layer` era só a chave de single-flight/cost-guard local (`visualLogicalKey`) e uma string de documentação (`policy().keyAuthority`) — nenhuma das duas envolve dinheiro do aluno.

Com a confirmação de que era recurso gratuito, corrigi as duas para serem item-level (mesma identidade visual reaproveitada entre Experience 1 e 2 do mesmo item), como a decisão já ratificada (Phase 04, Decisão 5) sempre pediu. Um teste (`play16_financial_anti_leakage_contract.test.js`) que ainda testava o comportamento antigo por camada foi corrigido para testar o comportamento novo por item.

Deixei de fora, de propósito, os campos `layer` que ainda vão para `ai-cost-protection-gate.js` (o gate de proteção de gasto do PROVEDOR do servidor, não cobrança do aluno) — isso é um mecanismo genérico compartilhado por vários organs (T00, T02, visual, imagem), arquivo protegido, e mexer nisso é um escopo mais largo do que "a chave do visual" que você pediu para revisar.

## Autorização e verificação

Como `visual-route-controller.js` está no grupo protegido `visual-runtime-sensitive`, criei a autorização
`docs/migracao-sim-nv/autorizacoes/SIM109-VISUAL-ITEM-LEVEL-KEY-2026-09-17.json` nomeando "visual runtime" explicitamente, citando sua confirmação.

`npm test` (98/98), `npm run check:protected` (passa com autorização), suítes de visual/N3/anti-leakage financeiro individualmente — tudo verde.

## Commits

- Servidor: `4b9c939` — fix + teste + autorização.
- App: `f51dc95` — correção do relatório de ontem (achado 1 fechado, achados 2 e 3 continuam abertos como estavam).

Push e fast-forward de `main` feitos nos dois repositórios, confirmados por hash.

## Status dos achados

1. **Fechado.** Chave visual agora é item-level, sem relação com cobrança.
2. **Aberto, como estava.** `readyLessonMaterials` continua sendo autoridade de conteúdo duplicada e ativa — não mexi.
3. **Aberto, como estava.** `image-controller.js`/`/api/generate-lesson-image` continuam parecendo sem chamador do app — não mexi. Sua observação de que geração de imagem paga é exclusiva do SIM Web reforça a hipótese de que essa rota é mesmo do SIM Web, hospedada no mesmo servidor compartilhado, e não deste app — mas não confirmei isso de forma definitiva, então mantive como estava.
