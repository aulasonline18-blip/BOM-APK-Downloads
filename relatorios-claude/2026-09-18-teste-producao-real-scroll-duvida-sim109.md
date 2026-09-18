# Teste manual contra produção real (simaitutor.com) — scroll pedagógico + Sala de Dúvida — SIM109

**Data:** 2026-09-18
**Segue:** `2026-09-18-consolidacao-sim109itempackages-sole-authority.md` (que por sua vez seguiu a missão de reconstrução canônica do scroll + prefetch N+1 de imagem)
**Escopo:** `sim109-nplus1-scroll` (app), branch `feature/nplus1-image-and-canonical-scroll`, HEAD `80b92d2`. Servidor: nenhuma alteração nesta rodada — apenas observação dos logs de produção.

## O que foi pedido

Reconfigurar o APK de teste para apontar para produção real (`simaitutor.com`, Droplet, não o servidor local de teste), gerar/instalar novo build, e testar manualmente sozinho: as três transições de scroll e a Sala de Dúvida, acompanhando os logs de produção via SSH durante todo o teste.

## Build e instalação

`flutter build apk --debug --dart-define=SIM_SERVER_URL=https://simaitutor.com` — SHA-256 `f46bdf17...5dd1259fd125dd4589bf1c51f`, instalado via `adb install -r` no tablet físico (Galaxy Tab, Tailscale). `SimEnvironment.apiBaseUrl` prioriza `SIM_SERVER_URL` independente de modo produção/dev, então isso aponta o app de teste para o Droplet real sem precisar de `FLUTTER_APP_MODE=production`.

## Resultado dos testes

| Item | Resultado |
|---|---|
| Transição 1 (Alternativa → Indicadores) | **OK** — reveal mínimo, estável, sem soco nem reversão |
| Transição 2 (Indicador → Feedback/Ações) | **OK** — mesmo comportamento |
| Transição 3 (entrar na próxima experiência) | **OK** — ver investigação abaixo |
| Sala de Dúvida | **OK** — ver investigação abaixo |
| Logs de produção durante todo o teste (~90 min) | **Saudáveis** — ver seção de logs |

## Investigação 1 — suspeita de scroll voltando sozinho (falso alarme)

Depois de ver a Transição 3 assentar corretamente em duas capturas próximas no tempo, uma captura mais tardia pareceu mostrar o viewport voltando para trás. Antes de reportar isso como bug, rodei um teste controlado: rolei manualmente para uma posição arbitrária conhecida, aguardei 35 segundos reais sem tocar em nada, e comparei a posição antes/depois pixel a pixel — **idêntica**. Concluí que a observação original foi um engano meu: usei `sleep 6` mas o tempo real decorrido entre as capturas (por lag do ADB/sistema) foi de quase 2 minutos, e me confundi ao interpretar isso como movimento espontâneo. `adb logcat` confirmou zero atividade do app no intervalo. **Não há evidência de violação das regras de scroll passivo (B14/B19/B20).**

## Investigação 2 — resposta da Sala de Dúvida "sumida" (também falso alarme, mas com dois incidentes reais no meio)

Ao testar a Sala de Dúvida pela primeira vez, enviei uma dúvida ("não entendi porque a opção B está errada"), vi minha própria pergunta aparecer no chat, mas não vi nenhuma resposta da IA — mesmo depois de rolar a tela e reiniciar o app. Isso levantou a suspeita de um bug real no guard de "resposta obsoleta" da Sala de Dúvida (`LessonDoubtController`/`_isDoubtScopeStillCurrent` em `lab_session_warmup_flows.dart`), que já foi objeto de auditoria em sessão anterior.

Antes de reportar como bug, refiz o teste com mais cuidado e **encontrei a resposta da IA renderizada corretamente no chat** — eu só tinha rolado a tela para o lado errado e não vi. A resposta ("A opção B, 'Where is my flight?', não é a mais adequada porque é muito vaga...") estava completa, relevante e bem formatada, no lugar esperado da timeline. **A Sala de Dúvida funciona corretamente de ponta a ponta.** Confirmado também no servidor: rota `/api/complete-lesson` com `"kind":"doubt"`, 200 OK em ~3s.

## Dois incidentes operacionais causados por mim (transparência total)

1. **Logout acidental:** um erro meu de escala de coordenadas (esqueci de dobrar as coordenadas lidas da miniatura do screenshot antes de mandar o toque real no dispositivo — resolução real 1200×1920, miniatura 600×960) fez um toque cair sobre "Sair da conta" no menu de configurações. O app voltou para a tela de login. **Confirmei nos logs do servidor que não houve chamada de exclusão de conta nem qualquer perda de dado** — apenas logout local, sem nenhuma requisição destrutiva no período. Por orientação do usuário, retomei com "Continuar com Google" usando a conta `joelgomes522@gmail.com`, e o app voltou exatamente para o ponto onde a sessão de teste estava (mesma aula, mesmo item, mesma imagem carregada) — nada foi perdido.
2. **App errado por alguns minutos:** outro toque com coordenada mal calculada abriu, sem querer, um atalho do Chrome (WebAPK) instalado no tablet que se parece visualmente com o ícone do app nativo — é um atalho da página `simaitutor.com` adicionado à tela inicial, não o APK que eu tinha acabado de instalar. Por alguns minutos testei essa página web por engano, o que gerou telas de login/cadastro confusas e abas de navegador aparecendo sozinhas (Play Console, 404). Identifiquei o problema com `adb shell dumpsys window | grep mCurrentFocus` (mostrou `com.android.chrome/.../SameTaskWebApkActivity` em vez de `com.simaitutor.app`) e corrigi lançando o pacote nativo certo diretamente via `adb shell monkey -p com.simaitutor.app`.

Nenhum dos dois incidentes teve qualquer efeito destrutivo confirmado no servidor.

## Logs de produção — resumo (~90 minutos de teste)

```
200: 56   409: 1   413: 4   5xx: 0   exceptions: 0
```

- **1× 409** (`STATE_HIGH_WATER_MARK_REGRESSION`): guard de anti-regressão durante o bootstrap da conta de teste nova — comportamento esperado, autocorrigido na tentativa seguinte (200 seis segundos depois).
- **4× 413**: limite pré-existente de 1.25MB (`MAX_LIVE_STATE_BYTES`) rejeitando um payload de estado local de ~3.8MB acumulado por testes extensivos de sessões anteriores — guard de segurança correto e intencional, sem relação com o código desta missão.
- Linhas "Fontconfig error" no log: avisos benignos de um container sem cache de fontes gravável, emitidos pelo subprocesso de geração de imagem (`/api/visual-route`) — não impedem a resposta 200 que vem logo em seguida.

## Conclusão

O app funcionou corretamente contra produção real nas três transições de scroll e na Sala de Dúvida. Os dois "bugs" que suspeitei durante o teste eram, na investigação, enganos meus de navegação/tempo — não defeitos no código da missão. Os logs de produção confirmam tráfego saudável, sem nenhum erro atribuível ao código testado.

**Pendente:** geração de AAB — aguardando autorização explícita, conforme instrução da missão anterior.
