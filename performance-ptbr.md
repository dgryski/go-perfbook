# Escrevendo e otimizando o código Go

Este documento descreve boas práticas para escrever código Go de alto desempenho.

Embora existam discussões focadas em tornar os serviços individuais mais rápidos (cache, etc), projetar sistemas distribuídos de alto desempenho vai além do escopo deste trabalho. Já existem bons textos sobre monitoramento e projeto de sistemas distribuídos. Eles englobam um conjunto totalmente diferente de decisões de pesquisa e design.

Todo o conteúdo será licenciado sob o CC-BY-SA.

Este livro está dividido em diferentes seções:

1. Dicas básicas para escrever software que não é lento
    * Material de nível introdutório (CS 101)
1. Dicas para escrever software rápido
    * Veja seções específicas sobre como obter o melhor do Go
1. Dicas avançadas para escrever software *realmente* rápido
    * Para quando o seu código otimizado não é rápido o suficiente

Podemos resumir estas três seções como:

1. "Seja razoável"
1. "Seja intencional"
1. "Seja perigoso"

## Quando e onde otimizar

Estou colocando isso em primeiro lugar porque é realmente o passo mais importante. Você deveria mesmo estar fazendo isso?

Toda otimização tem um custo. Geralmente esse custo é expresso em termos de complexidade de código ou carga cognitiva - o código otimizado é raramente mais simples do que a versão sem otimizações.

Mas há outro lado que chamarei de economia da otimização. Como programador, seu tempo é valioso. Há o custo da oportunidade de trabalhar em outras coisas no projeto, por exemplo, quais erros corrigir ou quais funcionalidades adicionar. Otimizar as coisas é divertido, mas nem sempre é a tarefa certa a escolher. O desempenho é uma característica (*feature*), mas entrega e corretude também são.

Escolha a coisa mais importante para trabalhar. Às vezes não é em uma otimização da CPU, mas na experiência do usuário. Algo tão simples como adicionar uma barra de progresso, ou tornar uma página mais responsiva ao fazer cálculos no plano de fundo depois de renderizar a página.

Às vezes isso será óbvio: um relatório de hora em hora que leva três horas para ficar pronto é, provavelmente, menos útil do que aquele que é concluído em menos de uma hora.

Só porque algo é fácil de otimizar não significa que vale a pena ser otimizado. Ignorar os casos mais fáceis é uma estratégia de desenvolvimento válida.

Pense nisso como uma otimização do *seu* tempo.

Você pode escolher o que otimizar e quando otimizar. Você pode mover o controle deslizante entre "Software rápido" e "Implantação rápida".

As pessoas ouvem e repetem "a otimização prematura é a raiz de todo mal", mas eles perdem o contexto completo da citação.

"Os programadores gastam muito tempo pensando ou se preocupando com a velocidade de partes não-críticas de seus programas. Estas tentativas de conseguir eficiência tem um forte impacto negativo quando a depuração e manutenção são consideradas. Devemos esquecer pequenas eficiências, digamos cerca de 97% do tempo: a otimização prematura é a raiz de todo o mal. Porém, não devemos deixar passar nossas oportunidades nesses 3% críticos."
-- Knuth (*tradução livre*)

Adicione: https://www.youtube.com/watch?time_continue=429&v=RT46MpK39rQ
   * não ignore as otimizações fáceis
   * mais conhecimento de algoritmos e estruturas de dados torna mais otimizações "fáceis" ou "óbvias"

"Você deve otimizar? Sim, mas somente se o problema for importante, o programa é realmente muito lento e há alguma expectativa de que pode ser feito mais rápido, mantendo a corretude, robustez e clareza".
-- A prática da programação, Kernighan e Pike (*tradução livre*)

[BitFunnel performance estimation] (http://bitfunnel.org/strangeloop) tem alguns números que tornam esta decisão mais explícita. Imagine uma máquina de busca hipotética que precisa de 30.000 máquinas em vários *datacenters*. Essas máquinas tem um custo de aproximadamente US$ 1.000 por ano. Se você pode dobrar a velocidade do software, isso pode economizar US$ 15 milhões por ano. Até mesmo um único desenvolvedor gastando um ano inteiro para melhorar o desempenho em apenas 1% irá se pagar.

Na grande maioria dos casos, o tamanho e a velocidade de um programa não são uma preocupação. A otimização mais fácil é não ter que fazê-la. A segunda otimização mais fácil está apenas comprando hardware mais rápido.

Uma vez decidido que você irá mudar seu programa, continue lendo.

### Alterações de dados

Alterar seus dados significa adicionar ou alterar a representação dos dados que você está processando. Do ponto de vista do desempenho, alguns deles acabarão mudando o O() associado a diferentes aspectos da estrutura de dados.

Idéias para melhorar a sua estrutura de dados:

* Campos extras

Por exemplo, armazene o tamanho de uma lista ligada em vez de iterar quando o tamanho for solicitado. Ou armazene ponteiros adicionais para nós frequentemente acessados (por exemplo, links "invertidos" em uma lista duplamente ligada para tornar a remoção O(1)). Esses tipos de alterações são úteis quando os dados de que você precisa são baratos para serem armazenados e mantidos atualizados.

* Índices de pesquisa adicionais

A maioria das estruturas de dados é projetada para um único tipo de consulta. Se você precisar de dois tipos de consulta diferentes, ter uma "visão" adicional de seus dados pode ser uma grande melhoria. Por exemplo, `[]struct` referenciada por ID, mas as vezes `string` -> `map[string]id` (ou `*struct`).

* Informações extras a cerca elementos

Por exemplo, um filtro de _bloom_. Estes precisam ser pequenos e rápidos para não sobrecarregar o resto da estrutura de dados.

* Se as consultas forem caras, adicione um _cache_

Estamos todos familiarizados com o _memcache_, mas existem _caches_ internos ao processo.

  * Ao longo do fio, os custos de transferência + serialização vão doer.
  * _Caches_ internos ao processo, mas agora você precisa se preocupar com a expiração e a pressão adicional do GC.
  * Mesmo um único item pode ajudar (por exemplo, interpretação do tempo em um arquivo de log).

TODO: o "cache" não precisa ser chave-valor, mas apenas um ponteiro para onde você estava trabalhando. Isso pode ser tão simples quanto um "dedo de busca" (_search finger_)*.

Esses são exemplos claros de "fazer menos trabalho" no nível de estrutura de dados. Todos eles tem um custo de espaço. Na maioria das vezes, se você estiver otimizando para CPU, seu programa usará mais memória. Este é o clássico [compromisso de espaço-tempo](https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff).

Se o seu programa usa muita memória, também é possível ir para o outro lado. Reduza o uso do espaço em troca de um maior custo computacional. Em vez de armazenar as coisas, calcule-as sempre que precisar. Você também pode compactar os dados na memória e descompactá-los no momento que precisar.

[Small Memory Software](https://gamehacking.org/faqs/Small_Memory_Software.pdf) é um livro disponível online que cobre técnicas para reduzir o espaço usado por seus programas. Embora tenha sido originalmente escrito visando desenvolvedores de sistemas embarcados, as idéias são aplicáveis ​​a programas que lidam com grandes quantidades de dados excutando em hardware moderno.

* Reorganize seus dados

Elimine o preenchimento de estruturas (_structure padding_). Remova campos extras. Use um tipo de dados menor.

* Mudar para uma estrutura de dados mais lenta

Estruturas de dados mais simples freqüentemente possuem requisitos de memória menores. Por exemplo, sair de uma estrutura de árvore com muitos ponteiros para usar listas e buscas lineares.

* Formato de compactação personalizado para seus dados

[]byte (snappy, gzip, lz4), ponto flutuante (go-tsz), inteiros (delta, xor + huffman), existem muitos recursos para compactação. Você precisa inspecionar os dados ou podem permanecer compactados? Você precisa de acesso aleatório ou apenas _streaming_? Comprimir blocos com índice extra. Se não forem apenas internos ao processo, mas gravados em disco, devemos pensar em migração ou adição/remoção de campos. Agora você estará lidando com byte[] em vez de tipos de Go estruturados.

Nós falaremos mais sobre os _layouts_ de dados mais tarde.

Computadores modernos e a hierarquia de memória tornam o equilíbrio espaço/tempo menos claro. É muito fácil que as tabelas de consulta fiquem "distantes" na memória (e, portanto, de acesso caro), tornando mais rápido a recomputação de um valor sempre que for necessário.

Isso também significa que o _benchmark_ frequentemente mostrará melhorias que não são realizadas no sistema de produção devido à contenção de _cache_ (por exemplo, as tabelas de consulta estão no cache do processador durante o _benchmark_, mas sempre substituídas por "dados reais" quando usadas em um sistema real. O artigo [Jump Hash](https://arxiv.org/pdf/1406.2294.pdf) da Google abordou isso diretamente, comparando o desempenho em um _cache_ de processador com e sem contenção (consulte os gráficos 4 e 5 no artigo Jump Hash).

TODO: como simular um _cache_ com contenção, mostrar custo incremental

Outro aspecto a considerar é o tempo de transferência de dados. Geralmente, o acesso à rede e ao disco são muito lentos e, portanto, acessar um fragmento compactado será muito mais rápido do que o tempo extra de CPU necessário para descompactar os dados, uma vez que tenham sido carregados. Como sempre, confirme usando _benchmarks_. Um formato binário geralmente será menor e mais rápido de analisar que um formato textual, mas ao custo de não ser legível para humanos.

Para transferência de dados, mova para um protocolo menos tagarela ou melhore a API para permitir consultas parciais. Por exemplo, uma consulta incremental em vez de forçar a buscar o conjunto de dados inteiro a cada vez.

\* Mais informações sobre "dedos de busca" em https://en.wikipedia.org/wiki/Finger_search