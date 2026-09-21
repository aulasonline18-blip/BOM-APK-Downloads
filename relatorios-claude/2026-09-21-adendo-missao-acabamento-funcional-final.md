# 2026-09-21 — Adendo de missão: acabamento funcional final do SIM

## Natureza da missão

Esta missão não é uma auditoria passiva.

Esta missão não é apenas olhar um checklist.

Esta missão é o acabamento funcional final do SIM.

O agente responsável deve pegar cada corredor funcional designado, executar no uso real, corrigir o que estiver quebrado, validar novamente e avançar para o próximo corredor.

## Função correta do executor

Para cada sala, fluxo, função ou mecanismo:

1. testar fisicamente;
2. observar o comportamento real;
3. se funcionar, marcar como concluído e avançar;
4. se não funcionar, descobrir a causa raiz;
5. corrigir app, servidor ou integração, conforme necessário;
6. rodar teste automatizado proporcional ao risco;
7. commit e push;
8. build/deploy quando necessário;
9. retestar contra produção real;
10. atualizar handoff;
11. avançar para o próximo corredor.

A missão é:

`TESTAR -> ARRUMAR -> VALIDAR -> DEIXAR PRONTO -> AVANÇAR`

## O que não conclui trabalho

Não conclui trabalho:

- encontrar um erro;
- escrever um relatório;
- fazer um commit;
- corrigir apenas um bug;
- gerar um APK;
- declarar suspeita;
- marcar um item como pendente;
- terminar uma subtask.

Se encontrou problema, deve corrigir quando estiver dentro da autoridade técnica e operacional disponível.

## Definição de corredor pronto

Um corredor só está pronto quando o aluno consegue usá-lo normalmente de ponta a ponta, sem intervenção técnica.

Isso exige, conforme o caso:

- entrada correta;
- conteúdo correto;
- estado correto;
- interação correta;
- saída correta;
- retorno correto;
- persistência correta;
- restart seguro;
- integração app/servidor correta;
- zero crash;
- zero tela branca;
- zero loading eterno;
- zero loop;
- zero perda de progresso;
- zero cobrança indevida;
- zero estado econômico preso.

## Ordem funcional preferencial

A execução deve caminhar pela experiência real do aluno:

1. conta/login;
2. anexos;
3. criação de aula;
4. nivelamento/placement;
5. aula normal;
6. scroll/experiências;
7. Amparo;
8. Dúvida;
9. Revisão;
10. Recuperação;
11. finalização;
12. menu/drawer/rename;
13. restart/resume;
14. offline/reconnect;
15. account isolation;
16. microcrédito;
17. billing;
18. CG1/currículo grande;
19. qualquer corredor restante.

A ordem pode mudar por dependência ou oportunidade, mas a missão é cobrir tudo que for tecnicamente possível.

## Política de avanço

Se um item bloquear depois de investigação séria:

1. preservar evidência;
2. registrar `STATUS`, `EVIDENCE`, `ROOT CAUSE KNOWN?`, `NEXT ACTION` e `BLOCKER`;
3. marcar `IN_PROGRESS` ou `BLOCKED`;
4. avançar para outro corredor independente;
5. voltar depois.

O acabamento global não pode parar por um único corredor.

## Proibições

É proibido:

- bypass;
- fallback falso;
- esconder erro;
- resetar estado para fazer passar;
- desativar proteção;
- mascarar cobrança;
- apagar dado financeiro;
- pular lógica pedagógica;
- declarar pronto sem prova real.

Correção válida é correção de causa raiz.

## Autoridade final

Produção real é a prova final:

`https://simaitutor.com`

A VM pode ajudar no debug, mas `OK_PRODUCTION` só existe com validação contra produção real.

## Regra final

O agente deve fazer um arrastão de acabamento:

- está bom: preserve e avance;
- está ruim: ajuste;
- está quebrado: conserte;
- está inconsistente: alinhe;
- está travando: resolva;
- está duplicando: elimine a causa;
- está perdendo estado: corrija.

Missão final:

`PEGAR TODAS AS SALAS E FUNÇÕES DESIGNADAS E DEIXAR CADA UMA FUNCIONANDO DE PONTA A PONTA.`

