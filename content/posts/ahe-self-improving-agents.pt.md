---
title: "Agente de código que evolui a cada execução"
date: 2026-05-16T00:00:00-03:00
draft: false
tags:
  - ia
  - agentes
  - harness-engineering
  - llm
summary: "Ler o paper AHE me fez repensar onde a melhoria de agentes realmente acontece. Os maiores ganhos não vieram de prompts melhores. Vieram de memória, ferramentas e middleware. Aqui está o que isso significa para times reais de engenharia."
---

Nos últimos meses venho experimentando algumas técnicas de codificação agêntica e percebi que em alguns cenários evoluir o prompt não é suficiente. Foi em uma conversa descontraída sobre fluxo de execução que cogitei criar um arquivo markdown de log para guardar as ações executadas. Até que essa abordagem funciona, mas não atende todos os cenários. Lendo o paper [AHE: Agentic Harness Engineering](https://arxiv.org/abs/2604.25850), publicado no arXiv em abril de 2026, e conferindo os resultados dos benchmarks, passei a ter uma outra visão sobre como aplicar um modelo de memória para o meu harness. Além disso, esse artigo confirmou uma percepção que eu já tinha: memórias são mais valiosas que otimização de prompt. Se você tem investido a maior parte do seu esforço de otimização em prompt engineering, esse paper tem algo a dizer para você também. Continua lendo. Consolidei as ideias e faço alguns paralelos da experiência que venho tendo ao tentar construir um harness corporativo.

## O que é um harness e por que isso importa

Antes de chegar ao resultado, é preciso entender o que o paper está otimizando.

Um harness é tudo que envolve um LLM e o faz interagir com o mundo: o system prompt, sim, mas também as ferramentas disponíveis para o agente, como essas ferramentas são implementadas, o middleware que intercepta e valida ações, a memória que persiste entre tarefas, e as regras que governam como o agente decide que terminou.

Se você já trabalhou com LangChain, LangGraph, Cursor ou Claude Code, você já constrói harnesses ou os customiza. Os prompts e definições de ferramentas fazem parte. Mas o loop de execução também faz, assim como a lógica de retry, o tratamento de erros, a forma como outputs são processados e realimentados. O harness é o ambiente onde o modelo opera, não apenas o que o modelo lê.

Essa distinção importa mais do que parece. Você pode ter um system prompt perfeitamente escrito dentro de um harness quebrado, e o agente ainda vai falhar. O modelo é uma entrada. O harness determina o que o modelo pode observar, o que pode fazer, e se consegue se recuperar quando algo dá errado.

## Como o Agentic Harness Engineering (AHE) funciona

O framework AHE automatiza a evolução do harness por meio de um loop fechado com três atores distintos.

O **Code Agent** é o agente que está sendo otimizado. Começa como um agente mínimo que usa apenas bash. Ele executa tarefas de benchmark e gera trajetórias de execução.

O **Agent Debugger** lê essas trajetórias, que podem chegar a dezenas de milhões de tokens, e as destila em relatórios estruturados de falha. Ele não despeja logs brutos. Produz evidências em camadas: padrões resumidos, análises por tarefa, causa raiz, tudo rastreável de volta à trajetória original. O paper faz uma distinção clara aqui: logging diz o que aconteceu; observabilidade diz por que aconteceu, em uma forma que outro agente consegue agir.

O **Evolve Agent** lê os relatórios do Debugger e edita os arquivos do harness. Mas opera sob uma restrição fundamental: cada edição deve vir acompanhada de uma previsão. Antes de mudar qualquer coisa, o agente declara o que espera melhorar na próxima rodada. Depois que a rodada executa, o sistema verifica se a previsão se confirmou.

O paper chama isso de contrato falsificável. É o mecanismo que separa evolução sistemática de tentativa e erro. Sem ele, um agente modificando o próprio ambiente está apenas chutando no escuro. Com ele, cada mudança é uma hipótese, cada resultado é dado, e o agente aprende não só o que funciona, mas o que não funciona e por quê.

O harness é decomposto em sete componentes independentes com representação em arquivo: system prompt, descrições de ferramentas, implementações de ferramentas, middleware, skills, configurações de sub-agentes e memória de longo prazo. Cada um é versionado e revertível individualmente. Se uma mudança degrada a performance, você sabe exatamente o que mudou e pode reverter sem afetar o restante.

## O resultado que me incomodou

O paper indica que após 10 iterações, o harness evoluído pelo AHE melhorou o pass@1 no Terminal-Bench 2 de 69.7% para 77.0%, superando um harness projetado por humanos e dois baselines de auto-evolução. Os pesquisadores isolaram cada componente e mediram quanto cada um contribuiu individualmente para a melhoria:

| Componente | Contribuição isolada |
|---|---|
| Memória de longo prazo | +5.6 pp |
| Ferramentas | +3.3 pp |
| Middleware | +2.2 pp |
| System prompt | **-2.3 pp** |

Por incrível que pareça, o system prompt editado isoladamente piorou os resultados.

O maior ganho veio da memória: a capacidade de acumular experiência entre tarefas e carregá-la adiante. O segundo maior veio das ferramentas: tornar os mecanismos de execução mais robustos. O terceiro veio do middleware: interceptar ações arriscadas ou incorretas antes que causassem falhas.

Nenhum desses é trabalho de prompt. São trabalho de engenharia.

Isso não significa que prompts não importam. Importam, e no harness evoluído completo contribuem junto com todo o resto. Mas sugere que podemos estar tratando a melhoria de agentes como um problema principalmente de prompting, quando provavelmente estamos trabalhando na camada errada.

As falhas que mais importam em tarefas complexas e multi-etapa de coding não se resolvem com instruções melhores. Se resolvem com mecanismos melhores: memória que sabe o que foi verificado, ferramentas que tratam casos extremos com robustez, middleware que captura o agente antes que ele declare sucesso prematuramente.

## A pergunta que não consegui parar de pensar

Lendo esse paper como alguém que vem implementando e coordenando criação e uso de harness com desenvolvimento por especificação, fiquei preso em uma pergunta que não é o foco do paper, logo ele não responde. Para fazer sentido, vou contextualizar como cheguei nela.

O AHE usa benchmarks com código open source complexo, mas são repositórios maduros com vários cenários de teste. Nesse cenário, o benchmark garante que a implementação passa ou falha. Com isso, o agente pode evoluir sem supervisão humana porque existem critérios com alta qualidade definida.

Mas em um time real de engenharia, com requisitos ambíguos e critérios de aceitação que existem porque um analista de negócios os negociou com um cliente, a verificação automatizada não cobre tudo. O agente pode passar em todos os testes e ainda assim construir a coisa errada. Algumas garantias exigem julgamento humano que nem sempre tem todo o contexto para a tomada de decisão, mas precisa decidir.

Então o AHE se desfaz nesse contexto? Não acho. Acho que o modelo muda, e quero compartilhar algumas reflexões sobre esse cenário.

Em um time com desenvolvimento orientado por especificação, o loop se torna: o agente observa, propõe, e o humano aprova. O agente executa tarefas, o debugger destila o que falhou, o agente de evolução rascunha uma entrada de memória: "esse tipo de requisito mal especificado gera consistentemente falhas nessa camada." O tech lead ou arquiteto revisa se o padrão é real, se a regra proposta faz sentido, se deve virar um ADR, um template de spec, ou uma atualização no prompt do agente de execução.

O agente contribui para o conhecimento institucional do time sem ter autoridade final sobre garantias que exigem julgamento humano. É um tipo diferente de evolução, mais lento e mais restrito, mas aplica os mesmos princípios: componentes explícitos, experiência destilada, previsões que são verificadas.

O contrato falsificável não precisa ser automatizado para ser valioso. Um time que anota o que espera que uma mudança de spec vá melhorar, e depois verifica se melhorou de fato, está fazendo decision observability de forma manual. A maioria dos times não faz isso, e é por isso que os mesmos tipos de falhas continuam se repetindo.

## O que isso significa na prática

Não estou sugerindo que você abandone tudo e implemente um harness auto-evolutivo para o seu agente de coding amanhã. O framework AHE é um protótipo de pesquisa que exige benchmarks, sandboxes e configuração cuidadosa de experimentos.

Mas os princípios são imediatamente aplicáveis.

Torne o harness do seu agente endereçável por arquivo. Se os prompts, definições de ferramentas, seeds de memória e regras de runtime do seu agente existem como constantes escondidas espalhadas pelo codebase, você não consegue evoluí-los de forma sistemática. Precisam ser artefatos explícitos, inspecionáveis, versionados e rastreáveis.

Trate ferramentas e memória como alvos de otimização de primeira classe, não como secundários. O resultado dos benchmarks é um lembrete concreto de que os maiores ganhos de performance podem não vir do prompt.

Quando você mudar algo no ambiente do seu agente, escreva o que espera melhorar. Depois verifique. Esse hábito é o que estou preparando para adicionar nos meus harnesses, porque o valor de previsões falsificáveis é entregar intenção para os artefatos. Sabemos também que muitas vezes o time gera os artefatos via IA sem revisar. Esse hábito vai te dizer mais sobre o seu sistema do que qualquer quantidade de iteração casual.

E se você lidera um time: os mesmos princípios se aplicam ao processo de desenvolvimento. Component observability significa que o conhecimento compartilhado do time, specs, guidelines, decisões arquiteturais, prompts dos agentes, deve ser explícito e rastreável. Experience observability significa que as retrospectivas devem produzir padrões estruturados, não apenas sentimentos. Decision observability significa que as mudanças de processo devem vir acompanhadas de previsões que são verificadas.

A ideia deste post não é sugerir um modelo novo a partir do paper, mas fazer um paralelo de algumas técnicas muito usadas e provocar uma reflexão de como podemos incluir um processo intencional de verificação da intenção dos artefatos dos nossos harnesses. Encerro concordando ainda mais que temos muita coisa para explorar além de esperar modelos maiores. Esse paper mostra que ainda temos espaço para melhorar ambientes, evidências e disciplina de engenharia em torno dos sistemas que fazem os modelos agirem.

## Referências

- [Paper AHE: arXiv:2604.25850](https://arxiv.org/abs/2604.25850)
- [Repositório oficial do AHE](https://github.com/china-qijizhifeng/agentic-harness-engineering)
- [Terminal-Bench 2.0](https://www.tbench.ai/)
- [SWE-bench Verified](https://www.swebench.com/verified.html)
