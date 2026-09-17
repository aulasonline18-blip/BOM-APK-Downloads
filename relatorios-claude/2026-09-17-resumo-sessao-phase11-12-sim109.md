# Resumo da sessão — Phase 11/12 (recheck de integração + limpeza de legado) — SIM109

**Data:** 2026-09-17
**Escopo:** `sim109-economic-final-app` e `sim109-economic-final-server`, branch `reform/sim109-economic-final-construction` em ambos.

## O que foi feito nesta sessão

1. Reverifiquei as 51 entradas da matriz de integração da Phase 3 contra o código atual (pós Cut A/B/C1/C2): **zero integrações quebradas ou órfãs**. As 5 decisões de usuário pendentes (sinais, amparo, recovery, T02, visual) estão todas implementadas conforme o Phase 04 Human Decision Gate.
2. Investiguei e corrigi o que foi seguro corrigir do sistema antigo de 3 camadas.
3. Corrigi o achado 1 depois que você esclareceu que o SIM App não vende imagem por IA (isso é exclusivo do SIM Web).

## Status dos 3 achados

**Achado 1 — Chave de cache/custo do visual usando "layer" — FECHADO.**
Confirmei por leitura direta que `visual-route-controller.js` não tem nenhuma chamada de `reserveCredit`/`captureCredit`/`releaseCredit` — geração via `/api/visual-route` é gratuita ao aluno, sem relação com cobrança real. A identidade durável do cache (o que decide se um artefato salvo é reaproveitado) já era item-level; só a chave de single-flight local e uma string de documentação ainda citavam "layer". Corrigidas as duas para item-level, teste correspondente atualizado. Deixei de fora, por escopo, os campos que alimentam o gate de proteção de gasto do *provedor* do servidor (mecanismo genérico e protegido, não é cobrança do aluno).

**Achado 2 — `readyLessonMaterials`/layer — ABERTO, aguardando decisão.**
Não é um caminho morto (correção da minha própria descrição anterior): é uma autoridade de conteúdo duplicada e ativa, rodando em paralelo com `sim109ItemPackages`. Consolidar de verdade é um redesenho de arquitetura da camada de conteúdo/prepared-window, não uma remoção — não mexi sem sua decisão.

**Achado 3 — `image-controller.js`/rota `/api/generate-lesson-image` possivelmente órfã — ABERTO, aguardando decisão.**
Parece não ter chamador nenhum do app, no modelo antigo ou no novo. Isso é anterior ao trabalho de hoje, não uma regressão. Sua observação sobre o SIM Web reforça a suspeita de que essa rota pertence a ele, não a este app — mas não confirmei isso de forma definitiva, então não toquei.

## Correção segura de legado aplicada (Phase 12)

`studentStateHighWaterMark` (rank de conflito de sincronização) deixava um `layer: 3` antigo (pré-corte de 3 camadas) vencer progresso novo e legítimo de Experience 2. Corrigido travando o valor rankeado em no máximo 2. `LessonLayer.l3` em si e as travas defensivas que o tratam como inválido foram mantidas de propósito — são proteção contra crash de estado antigo no aparelho, não resíduo solto.

## Testes rodados

- App: `flutter analyze` limpo; `flutter test` completo **1426/1426**.
- Servidor: `npm test` **98/98** arquivos; `npm run check:protected` passa com autorização explícita; suítes de visual/N3/anti-leakage financeiro rodadas em separado.

## Commits

**App:**
- `ae1d36b` — fix + teste do rank de sincronização (legado layer-3) + relatório de execução.
- `f51dc95` — correção do relatório após o esclarecimento sobre o SIM Web (achado 1 fechado).

**Servidor:**
- `4b9c939` — fix da chave visual item-level + teste atualizado + autorização formal.

`main` atualizado (fast-forward) e confirmado por hash em ambos os repositórios após cada commit.

## Conclusão

Phase 11 fechada. Phase 12 parcialmente fechada: duas correções seguras aplicadas e testadas (rank de sincronização, identidade visual item-level); dois achados documentados aguardando sua decisão, nenhum deles bloqueando Billing/APK/AAB.
