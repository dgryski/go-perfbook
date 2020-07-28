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

## Como otimizar

### Fluxo de otimização


Antes de entrarmos nos detalhes, vamos falar sobre o processo geral de otimização.

Otimização é uma forma de refatoração. Entretanto, em vez de melhorar algum aspecto do código-fonte (duplicação de código, clareza, etc), ela melhora algum aspecto de desempenho como, por exemplo, reduzir o uso da CPU, reduzir a ocupação da memória, reduzir a latência, etc. Essas melhorias, geralmente, são implementadas a troco de alguma perda na legibilidade do código. Isso significa que além de um conjunto de testes unitários (para garantir que suas mudanças não irão quebrar nada), você também precisará de um bom conjunto de _benchmarks_ para garantir que suas mudanças estão, de fato, entregando o ganho de desempenho desejado. Você deve ser capaz de verificar se a alteração realmente está reduzindo o uso da CPU. Às vezes uma alteração que você pensava que iria melhorar o desempenho, na verdade não gera nenhum impacto ou, até, causa um impacto negativo. Nesses casos, sempre desfaça suas alterações.

<cite>[Qual é o melhor comentário que você já encontrou em um código? - Stack Overflow](https://stackoverflow.com/questions/184618/what-is-the-best-comment-in-source-code-you-have-ever-encountered)</cite>:
<pre>
//
// Caro mantenedor:
//
// Assim que desistir de tentar "otimizar" essa rotina,
// e perceber que terrível engano você cometeu,
// por favor, incremente o contador a seguir como uma forma de aviso
// à próxima pessoa:
//
// total_hours_wasted_here = 42
//
</pre>

Os _benchmarks_ que você decidir usar devem ser precisos e devem oferecer números reproduzíveis em cargas relevantes. Se execuções individuais tiverem uma variância muito alta, isso tornará mais difícil a detecção de pequenas melhorias. Assim, você precisará usar o [benchstat](https://golang.org/x/perf/benchstat) ou uma solução equivalente para realizar testes estatísticos já que não conseguirá verificar as melhorias apenas via observação. (Note que a utilização de testes estatísticos é uma boa ideia em qualquer cenário). Os passos para executar os _benchmarks_ devem estar documentados e quaisquer scripts e/ou ferramentas adicionais devem ser incluídas no repositório com instruções de como utilizá-los. Esteja atento a grandes conjuntos de _benchmark_ que requerem muito tempo para sua execução: isso irá tornar o processo de desenvolvimento mais lento.

Lembre-se, também, que tudo que pode ser medido pode ser otimizado. Tenha certeza de que está medindo a coisa certa.

O próximo passo é decidir qual é o seu objetivo com a otimização. Se o objetivo é melhorar o uso da CPU, qual velocidade é aceitável? Você quer melhorar o desempenho em 2x ou em 10x? Você pode definir isso como "um problema grande como N que precisa ser resolvido num tempo menor que T"? Você está tentando reduzir o uso de memória? Em quanto? Para uma determinada redução de uso de memória, quão mais lento é aceitável? Do que você está disposto a abrir mão em troca de menos exigência de espaço?

Otimização com foco em latência de serviços é uma proposta mais complicada. Livros inteiros foram escritos sobre como testar o desempenho de servidores web. A principal questão é que, para uma única função, o desempenho é bastante consistente para um problema de determinado tamanho. Para _web services_, você não tem um único número. Um bom conjunto de _benchmark_ para _web services_ fornecerá uma distribuição de latência para um dado nível de requisições por segundo. Esta palestra dá uma boa visão geral de alguns dos problemas:["How NOT to Measure Latency" by Gil Tene](https://youtu.be/lJ8ydIuPFeU)

TODO: Veja a seção a seguir sobre otimização de _web services_.

As metas de desempenho devem ser específicas. (Quase) sempre você será capaz de fazer algo ser mais rápido. Otimização é, frequentemente, um jogo de retornos decrescentes. Você precisa saber a hora de parar. Quanto esforço a mais você vai fazer para obter aquela pequena melhora? Quão disposto você está a fazer um código mais feio e mais difícil de manter?

A palestra de Dan Luu mencionada anteriormente em [BitFunnel performance estimation](http://bitfunnel.org/strangeloop) apresenta um exemplo do uso de cálculo aproximados para determinar se as metas de desempenho estimadas são razoáveis.

TODO: Programming Pearls tem "Problemas de Fermi". Conhecer os slides de Jeff Dean ajuda.

Para o desenvolvimento de novos projetos, você não deve deixar o a avaliação de desempenho para o fim. É fácil dizer "depois eu faço", mas se o desempenho é realmente importante, isso deve ser considerado desde a concepção do projeto. Quaisquer alterações relevantes na arquitetura para consertar problemas de desempenho serão ainda mais arriscadas quando tiverem que ser feitas próximas do prazo final. Perceba que, *durante* o desenvolvimento, o foco deve ser um desenho coerente, algoritmos e estrutura de dados. Otimização em níveis mais baixos da estrutura devem aguardar uma fase mais avançada do ciclo de desenvolvimento, quando houver uma visão mais completa do desempenho do sistema. Qualquer perfil completo de sistema que você faz enquanto o sistema está incompleto oferecerá uma visão distorcida de onde os gargalos estarão, de fato, no sistema acabado.

TODO: Como evitar/detectar "Morte por mil cortes (Lingchi)" por _software_ mal escrito.

O _benchmarking_ como parte do CI é difícil devido a interferências causadas pelo compartilhamento de recursos. Difícil, também, de ser ativado em métricas de desempenho. Um bom meio termo é ter _benchmarks_ executados pelo desenvolvedor (em hardware apropriado) e incluídos nos _commits_ que abordam, especificamente, o desempenho. Para aqueles que são apenas patches gerais, tente identificar as potenciais degradações de desempenho "a olho nu", na revisão de código.

TODO: Como acompanhar o desemepnho ao longo do tempo?

Escreva código que você pode comparar. Você pode fazer perfilamento em sistemas maiores, porém em _benchmarking_ você quer testar partes isoladas. Você precisa ser capaz de extrair e configurar o contexto necessário para que os _benchmarks_ executem testes representativos e suficientes.

A lacuna entre sua meta e o desempenho atual também te darão uma orientação de por onde começar. Se você precisa de apenas 10% a 20% de melhoria de desempenho, provavelmente você consegue alcançar isso com pequenos ajustes. Se você precisa de uma melhoria da ordem de 10x ou mais, isso vai exigir mudanças maiores em sua estrutura.

Um bom trabalho em aprimoramento de desempenho exige conhecimentos dos mais variados níveis, desde desenho de sistemas, rede, _hardware_ (CPU, caches, armazenamento), algoritmos, ajustes e _debugging_. Com tempo e recursos limitados, considere aquele que lhe dará o maior ganho: nem sempre será o algoritmo ou um ajuste fino no programa.

Em geral, as otimizações devem ocorrer de cima para baixo. Otimizações em nível de sistema terão mais impacto que aquelas em nível de código. Certifique-se de que você está resolvendo o problema no nível apropriado.

Esse livro irá tratar, em sua maior parte, sobre redução de uso da CPU, redução de uso da memória e redução de latência. É interessante destacar que, raramente, você fará os três ao mesmo tempo. Talvez o tempo de CPU esteja mais rápido, mas agora seu programa usa mais memória. Talvez você precise reduzir o espaço de memória, mas agora o programa levará mais tempo.

[Lei de Amdahl](https://en.wikipedia.org/wiki/Amdahl%27s_law) diz para nos concentrarmos nos gargalos. Se você dobra a velocidade da rotina que toma 5% do tempo de execução, houve um ganho de apenas 2,5% no tempo total. Por outro lado, aumentar a velocidade da rotina que toma 80% do tempo em apenas 10% oferece um ganho real de 8%. Perfilamento irá ajudar a identificar onde o tempo é realmente gasto.

Quando se está otimizando, você quer reduzir o trabalho que a CPU precisa fazer. _Quicksort_ é mais rápido que _bubble sort_ porque resolve o mesmo problema em menos passos. É um algoritmo mais eficiente. Você reduziu o trabalho que a CPU tem para executar a mesma tarefa.

O ajuste do programa, como as otimizações do compilador, geralmente trazem uma pequena melhora no tempo total de execução. Grandes vitórias quase sempre vêm de uma mudança algorítmica ou uma mudança na estrutura de dados ou uma mudança fundamental na forma como o seu
programa é organizado. A tecnologia de compiladores melhora, mas lentamente. A [Lei de Proebsting](http://proebsting.cs.arizona.edu/law.html) diz que compiladores melhoram seu desempenho em 2x a cada 18 *anos*, um contraste gritante com a Lei de Moore que diz que o desempenho dos processadores dobra a cada 18 *meses*. Melhorias em algoritmos funcionam em magnitudes maiores. Algoritmos para programação inteira mista [melhoraram por um fator de 30.000 entre 1991 e 2008](https://agtb.wordpress.com/2010/12/23/progress-in-algorithms-beats-moore%E2%80%99s-law/). Para um exemplo mais concreto, considere [essa decisão](https://medium.com/@buckhx/unwinding-uber-s-most-efficient-service-406413c5871d) de substituir de um algoritmo geo-espacial de força bruta descrito em um post do blog do Uber por um mais especializado e mais adequado para a tarefa apresentada. Não há mudança de compilador que lhe dará um aumento equivalente no desempenho.

Um perfilador pode mostrar que muito tempo é gasto em uma rotina específica. Pode ser que esta seja uma rotina cara ou uma rotina barata
porém que é chamada muitas vezes. Em vez de imediatamente tentar aumentar a velocidade de uma rotina, veja se você pode reduzir o número de vezes que ela é chamada ou, até mesmo, eliminá-la completamente. Vamos discutir estratégias de otimização mais concretas na próxima seção.

As três perguntas da otimização:

* Nós precisamos fazer isso mesmo? O código mais rápido é aquele nunca executado.
* Se sim, esse é o melhor algoritmo?
* Se sim, essa é a melhor *implementação* desse algoritmo?

## Dicas concretas sobre otimização

O trabalho de Jon Bentley em 1982, "Writing Efficient Programs", abordou a otimização de programas como um problema de engenharia: Benchmark. Analisar. Melhorar. Verificar. Iterar. Várias de suas dicas agora são feitas automaticamente por compiladores. Um dos trabalhos dos programadores é  usar as "Transformações" que um compilador *Não pode* realizar.


Há resumos deste livro:

* <http://www.crowl.org/lawrence/programming/Bentley82.html>
* <http://www.geoffprewett.com/BookReviews/WritingEfficientPrograms.html>

e as regras de ajuste de programas:

* <https://web.archive.org/web/20080513070949/http://www.cs.bell-labs.com/cm/cs/pearls/apprules.html>

Ao pensar em mudanças que você pode fazer no seu programa, existem duas opções básicas:
você pode alterar seus dados ou alterar seu código.

### Alterações nos dados

Alterar seus dados significa adicionar ou alterar a representação dos dados que você está processando. Do ponto de vista de desempenho, alguns desses vão acabar mudando a complexidade O() que é associada a diferentes aspectos das estruturas de dados.

Idéias para melhorar sua estrutura de dados:

* Campos extras

  O exemplo clássico disso é armazenar o tamanho de uma lista encadeada em um campo
  do nó raiz. Temos um pouco mais de trabalho para mantê-la atualizada, porém, em seguida, consultar
  o comprimento se torna uma pesquisa de campo simples em vez de um percurso O (n). Sua estrutura de dados
  pode apresentar um ganho semelhante: um pouco de *bookkeeping* durante algumas
  operações em troca de um desempenho melhor em um caso de uso comum.

  Da mesma forma, armazenar ponteiros para nós frequentemente necessários em vez de executar
  pesquisas adicionais. Isso abrange coisas como os links "para trás" em uma
  lista duplamente ligada para fazer a remoção do nó ter complexidade O (1). Algumas Skip lists mantêm uma "search
  finger ", onde você armazena um ponteiro de onde você estava em sua estrutura no pressuposto de que é um bom ponto de partida para a sua próxima operação.

* Índices extras de pesquisa

  A maioria das estruturas de dados é projetada para um único tipo de consulta. Se você precisar de dois
  tipos de consulta diferentes, ter uma "visualização" adicional nos seus dados pode ser uma grande
  melhoria. Por exemplo, um conjunto de estruturas pode ter um ID primário (inteiro)
  que você usa para procurar em uma fatia, mas às vezes precisa procurar com um
  ID secundário (string). Em vez de iterar sobre a fatia, você pode incrementar
  sua estrutura de dados com um mapa de string para ID ou diretamente para própria estrutura.

* Informação extra sobre elementos

  Por exemplo, manter um bloom filter de todos os elementos inseridos pode fazer com que você retorne rapidamente consultas sem resultado. Estes precisam ser pequenos e rápidos para não sobrecarregar o resto da estrutura de dados. (Se uma pesquisa em seus dados principais é barata, o custo do *bloom filter* superará qualquer economia.)

* Se as consultas forem caras, adicione um cache.

  Em um nível maior, um cache interno ou externo (como o memcache) pode ajudar. Isso pode soar excessivo para somente uma estrutura de dados. Falaremos mais sobre cache abaixo.

Esses tipos de alterações são utéis quando os dados necessários são baratos para armanezar e fáceis de se manterem atualizados.

Estes são exemplos claros de "Tenha menos trabalho" pensando no nível de estrutura de dados. Todos eles custam espaço. Na maior parte do tempo se você está otimizando pensando em uso de CPU, seu programa usará mais memória esta é a clássica [compensação espaço-temporal](https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff).

É importante pensar como essa compensação pode afetar as suas soluções -- de maneira indireta. Às vezes, uma pequena quantidade de memória pode resultar em uma melhoria significativa de velocidade, em outras situações este tradeoff é linear (2x o uso da memória == 2x a melhora de desempenho), em outras vezes é significativamente pior: uma enorme quantidade de memória fornece apenas uma pequena melhora de desempenho. Onde você precisa estar nesta curva de memória/desempenho podem afetar quais opções de algoritmos são razoáveis. Nem sempre é possível somente ajustar um parâmetro de um algoritmo. Diferentes usos de memória pode ter abordagens algorítimicas completamente diferentes.
 

Tabelas de pesquisa também se enquadram nessa compensação espaço-temporal. Uma tabela de pesquisa simples
pode ser apenas um cache de cálculos que foram solicitados anteriormente.


Se o domínio for pequeno o suficiente, o conjunto * inteiro de resultados poderá ser
pré-computado e armazenado na tabela. 

Como exemplo, essa poderia ser a abordagem adotada para uma implementação rápida de popcount, em que pelo número de bits ativos em um byte são armazenados em uma tabela de 256 entradas. Uma tabela maior pode armazenar os bits necessários para todas as palavras de 16 bits. Nesse caso, eles estão armazenando resultados exatos.


Vários algoritmos para funções trigonométricas usam tabelas de pesquisa como um
ponto de partida para realizar um cálculo.


Se o seu programa usa muita memória, também é possível seguir  outro caminho.
Reduza o uso de espaço em troca do aumento da computação. Em vez de armazenar
coisas, calcule-as sempre. Você também pode compactar os dados na memória
e descompactar rapidamente quando precisar.

Se os dados que você está processando estiverem em disco, em vez de carregar tudo na memória
RAM, você pode criar um índice para as peças necessárias e mantê-las
memória ou pré-processe o arquivo em pequenos pedaços viáveis.

[Small Memory Software](http://smallmemory.com/book.html)  é um livro disponível online que cobre técnicas utilizadas para o reduzir o espaço usado por seus programas.
Embora tenha sido originalmente escrito para desenvolvedores de software embarcado, as idéias são
aplicáveis a programas que rodam em hardware moderno que lidam com grandes quantidades de dados.

* Reorganize seus dados

  Elimine o preenchimento da estrutura. Remova campos extras. Use um tipo de dados menor.

* Mude para uma estrutura de dados mais lenta

  Estruturas de dados mais simples freqüentemente têm requisitos de memória mais baixos. Por exemplo, mudar de uma estrutura de árvore pesada com ponteiro para usar busca linear e slice em arrays.
  
* Formato de compactação personalizado para seus dados

  Algoritmos de compressão dependem muito do que está sendo compactado. É
  melhor escolher um que combine com seus dados. Se você tiver [] byte, algo
  como snappy, gzip, lz4, se comporta bem. Para dados de ponto flutuante, existe go-tsz
  para séries temporais e fpc para dados científicos. Muita pesquisa foi feita
  compactar números inteiros, geralmente para recuperação de informações em motores de pesquisa. Exemplos incluem codificação delta e varints para esquemas mais complexos envolvendo Huffman códificado com diferenças de OU-exclusivo. Você também pode criar formatos de compactação otimizados para seus tipos exatos de dados.

Você precisa inspecionar os dados ou eles podem permanecer compactados? Você precisa de acesso aleatório ou apenas streaming? Se você precisar acessar entradas individuais, mas não quer descomprimir a coisa toda, você pode compactar os dados em blocos menores e manter um índice indicando o intervalo de entradas em cada bloco. O acesso a uma única entrada só precisa verificar o índice e descompactar o bloco de dados menor.

Se seus dados não estão apenas sendo processados, mas também serão gravados em disco, que tal  migração de dados ou adição / remoção de campos. Agora você estará lidando com os [] byte em sua forma crua, em vez de bons tipos estruturados de Go, portanto, então você irá precisar do pacote unsafe e considerar as suas opções de serialização.

Falaremos mais sobre layouts de dados posteriormente.

Computadores modernos e a hierarquia de memória 
fazem o trade-off espaço / tempo menos claro. É fácil que as tabelas de pesquisa estejam "distantes" na memória (e
portanto, tornam seu acesso custoso), tornando mais rápido apenas recalcular um valor toda vez que for necessário.


Isso também significa que o benchmarking frequentemente mostrará melhorias que não são percebidos no sistema de produção devido à contenção de cache
(por exemplo, tabelas de pesquisa estão no cache do processador durante o benchmarking, mas sempre são liberadas por "dados reais" quando usados ​​em um sistema real). O artigo do Google sobre [Jump Hash paper](https://arxiv.org/pdf/1406.2294.pdf) 
abordou isso diretamente, comparando o desempenho em um cache de processador com e sem contenção. (Veja os gráficos 4 e 5 no artigo Jump Hash)


TODO: como simular um cache contencioso, mostrar custos incrementais
TODO: sync.Map como um exemplo go-ish de endereçamento de contenção de cache

Outro aspecto a considerar é o tempo de transferência de dados. Geralmente, o acesso à rede e ao disco é muito lento e, portanto,poder carregar um conjunto de dados compactos será muito mais rápido que o tempo extra da CPU necessário para descomprimir estes dados quando carregados. Como sempre, benchmark. Um formato binário geralmente será menor e mais rápido de analisar do que um texto, mas com o custo de não ser mais legível por humanos.


Para transferência de dados, vá para um protocolo menos falador ou aumente a API para permitir consultas parciais. Por exemplo, uma consulta incremental em vez de ser
forçado a buscar o conjunto de dados inteiro a cada vez.

### Alterações algorítmicas

Se você não estiver alterando os dados, a outra opção principal é alterar o código.

A maior melhoria provavelmente virá de uma alteração algorítmica. Isso equivale a substituir um bubble sort (`O(n^2)`)  por um quicksort (`O(n log n)`) ou substituir a varredura linear de uma matriz (`O(n)`) por uma pesquisa binária (`O (log n)`) ou pesquisa em um mapa (`O(1)`).

É assim que o software se torna lento. As estruturas projetadas originalmente para um uso são reaproveitadas para algo que não foram projetadas. Isso acontece gradualmente.


É importante ter uma compreensão intuitiva dos diferentes níveis Grande-O.
Escolha a estrutura de dados certa para o seu problema. Você não precisa cortar ciclos sempre, mas isso evita problemas de desempenho bobos que podem não ser percebidos até muito mais tarde.

As classes básicas de complexidade são:

* O (1): um acesso ao campo, matriz ou pesquisa de mapa

  Conselho: não se preocupe (mas lembre-se do fator constante).

* O (log n): pesquisa binária

  Conselho: apenas um problema se estiver em loop

* O (n): loop simples

  Conselho: você está fazendo isso o tempo todo

* O (n log n): dividir e conquistar, classificação

  Conselho: ainda bastante rápido

* O(n\*m): loop aninhado / quadrático

  Conselho: tenha cuidado e restrinja os tamanhos.

* Qualquer outra coisa entre quadrático e subexponencial

  Conselho: não execute isso em um milhão de linhas

* O(b ^ n), O(n!): Exponencial e acima

  Conselho: boa sorte se você tiver mais de uma dúzia ou dois pontos de dados

Link: <http://bigocheatsheet.com>

Digamos que você precise pesquisar um conjunto de dados não classificado. "Eu devo usar uma pesquisa binária", 
você pensa, sabendo que uma pesquisa binária é O (log n) mais rápida que a varredura linear O (n). 
No entanto, uma pesquisa binária exige que os dados sejam classificados, o que significa que você precisará classificá-los primeiro, o que levará tempo O (n log n).
Se você estiver fazendo muitas pesquisas, o custo inicial da classificação será recompensado. 
Por outro lado, se você estiver pesquisando principalmente em mapas, talvez ter uma matriz seja a escolha errada e seria melhor pagar o custo de pesquisa O (1) para um mapa.

Ser capaz de analisar seu problema em termos de notação Grande-O também significa que você pode descobrir se já está no limite do que é possível para o seu problema e se precisa mudar de abordagem para acelerar as coisas. Por exemplo, encontrar o mínimo de uma lista não classificada é `O(n)`, porque você precisa examinar cada item. Não há como tornar isso mais rápido.


Se sua estrutura de dados é estática, geralmente você pode fazer muito melhor do que o caso dinâmico. 
Torna-se mais fácil criar uma estrutura de dados ideal personalizada para exatamente seus padrões de pesquisa. 
Soluções como o hash perfeito mínimo podem fazer sentido aqui, ou filtros de bloom pré-computados. 
Isso também faz sentido se sua estrutura de dados for "estática" por tempo suficiente e você puder amortizar o custo inicial de construção em muitas pesquisas.

Escolha a estrutura de dados razoável mais simples e siga em frente. Este é o CS 101 para escrever "software não lento". 
Esse deve ser o seu modo de desenvolvimento padrão. Se você sabe que precisa de acesso aleatório, não escolha uma lista vinculada.
Se você sabe que precisa de uma travessia em ordem, não use um mapa. 
Os requisitos mudam e você nem sempre pode adivinhar o futuro. Faça um palpite razoável sobre a carga de trabalho.

<http://daslab.seas.harvard.edu/rum-conjecture/>

As estruturas de dados para problemas semelhantes diferem quando executam um trabalho. Uma árvore binária é classificada um pouco
de cada vez à medida que as inserções acontecem. Uma matriz não ordenada é mais rápida de inserir, mas não é ordenada: ao final, para "finalizar", você precisa fazer a ordenação de uma só vez.

Ao escrever um pacote para ser usado por outras pessoas, evite a tentação de otimizar antecipadamente todos os casos de uso. Isso resultará em código ilegível. Por projeto, as estruturas de dados são efetivamente de propósito único. Você não pode ler mentes nem prever o futuro. Se um usuário disser "Seu pacote está muito lento para este caso de uso", uma resposta razoável pode ser "Então use este outro pacote aqui". Um pacote deve "fazer uma coisa bem".

Às vezes, estruturas de dados híbridas fornecem a melhoria de desempenho que você precisa. Por exemplo, ao reunir seus dados, você pode limitar sua pesquisa a um único intervalo. Isso ainda paga o custo teórico de O(n), mas a constante será menor. Revisitaremos esses tipos de ajustes quando chegarmos em otimizações.

Duas coisas que as pessoas esquecem quando discutem a notação Grande-O:

Primeiramente, há um fator constante envolvido. Dois algoritmos que têm a mesma complexidade algorítmica podem ter diferentes fatores constantes. Imagine repetir uma lista 100 vezes ou apenas repetir uma vez. Embora ambos sejam O (n), um tem um fator constante 100 vezes maior.

Esses fatores constantes são o motivo pelo qual, embora merge sort, quick sort e heap sort sejam O (nlogn), todo mundo usa quick sort porque é a mais rápida. Este método de ordenação tem o menor fator constante.

A segunda coisa é que Grande-O diz apenas "à medida que n cresce até o infinito". Ele fala sobre a tendência de crescimento: "À medida que os números aumentam, esse é o fator de crescimento que dominará o tempo de execução". Não diz nada sobre o desempenho real, ou como ele se comporta com um valor de n pequeno.

Freqüentemente há um ponto de corte abaixo do qual um algoritmo  ingênuo é mais rápido. Um bom exemplo do pacote sort da biblioteca padrão Go. Na maioria das vezes, ele usa o quicksort, mas ele passa por uma classificação por shell e depois por inserção quando o tamanho da partição cai abaixo de 12 elementos.

Para alguns algoritmos, o fator constante pode ser tão grande que esse ponto de corte pode ser maior que todas as entradas razoáveis. Ou seja, o algoritmo O(n^2) é mais rápido que o algoritmo O(n) para todas as entradas com as quais você provavelmente lida.


Isso também significa que você precisa conhecer os tamanhos de entrada representativos, tanto para escolher o algoritmo mais apropriado quanto para escrever boas avaliações. 10 itens? 1000 itens? 1000000 itens?

Isso também acontece de outra maneira: por exemplo, optar por usar uma estrutura de dados mais complicada para fornecer o escalonamento de O (n) em vez de O (n^2),
mesmo que com os parâmetros de referência para pequenas entradas tenham ficado mais lentos. Isso também se aplica à maioria das estruturas de dados sem contenção. 
Elas geralmente são mais lentas no caso de uma única thread, mas são mais escaláveis quando estão sendo usadas por muitas threads.

A hierarquia de memória nos computadores modernos confunde um pouco o problema aqui, isto pois os caches preferem o acesso previsível de uma varredura ao acesso aleatório que temos com a perseguição de um ponteiro. Ainda assim, é melhor começar com um bom algoritmo. Falaremos sobre isso na seção específica de hardware.

TODO: estendendo o último parágrafo, mencione a notação O () é um modelo em que cada 
operação tem custo fixo. Essa é uma suposição errada no hardware moderno.

> A luta nem sempre é vencida pelo mais forte, nem a corrida pelo mais rápido,
mas essa é a maneira de apostar.
> -- <cite>Rudyard Kipling</cite>

Às vezes, o melhor algoritmo para um problema específico não é um único algoritmo, mas uma coleção de algoritmos especializados para classes de entrada ligeiramente diferentes. Esse "polialgoritmo" detecta rapidamente com que tipo de entrada ele precisa lidar e despacha para o caminho de código apropriado. É isso que faz o pacote de classificação mencionado acima: determina o tamanho do problema e escolhe um algoritmo diferente. Além de combinar quicksort, classificação de shell,
e classificação de inserção, também rastreia a profundidade de recursão do quicksort e chama heapsort, se necessário. Os pacotes `string` e` bytes` fazem algo semelhante, detectando e se especializando para casos diferentes. Assim como na compactação de dados, quanto mais você souber sobre a aparência de sua entrada, melhor poderá ser sua solução personalizada. Mesmo que uma otimização nem sempre seja aplicável, vale a pena complicar seu código, determinando que é seguro usar e executar lógicas diferentes.

Isso também se aplica aos subproblemas que seu algoritmo precisa resolver. 
Por exemplo, ter o usar radix sort a disposição pode ter um impacto significativo no desempenho ou 
usar a seleção rápida se você precisar apenas de uma classificação parcial.

Às vezes, em vez de especialização para sua tarefa específica, a melhor abordagem é abstraí-la para um espaço de 
problemas mais geral que foi bem estudado pelos pesquisadores. 
Em seguida, você pode aplicar a solução mais geral ao seu problema específico. 
Mapear seu problema em um domínio que já possui implementações bem pesquisadas pode ser uma vitória significativa.

Da mesma forma, usar um algoritmo mais simples significa que os detalhes das trocas, análises e implementação 
têm mais probabilidade de serem mais estudados e bem compreendidos do que os mais esotéricos ou exóticos e complexos.


Algoritmos mais simples também podem ser mais rápidos. Esses dois exemplos não são casos isolados
  https://go-review.googlesource.com/c/crypto/+/169037
  https://go-review.googlesource.com/c/go/+/170322/

TODO: notas sobre seleção de algoritmo

TODO:
  melhore o comportamento do pior caso a um custo leve para o tempo de execução médio
  da correspondência de expressão regular de tempo linear


Embora a maioria dos algoritmos seja determinística, há uma classe de algoritmos que usam a aleatoriedade como uma maneira de simplificar etapas de tomada de decisão complexas.
Em vez de ter um código que faz a coisa certa, você usa a aleatoriedade para selecionar uma coisa provavelmente não *ruim*. Por exemplo, um treap é uma árvore binária probabilisticamente equilibrada. Cada nó tem uma chave, mas também recebe um valor aleatório. 
Ao inserir na árvore, o caminho de inserção normal da árvore binária é seguido, mas os nós também obedecem à propriedade heap
com base no peso atribuído aleatoriamente a cada nó. Essa abordagem mais simples substitui soluções de 
rotação de árvores complicadas (como árvores AVL e Rubro negras), mas ainda mantém uma árvore equilibrada com inserção/pesquisa de O (log n) "com alta
probabilidade". As Skip lists são outra estrutura de dados simples e semelhante que usa aleatoriedade para produzir "provavelmente" inserção e pesquisas de O (log n).

Da mesma forma, a escolha de um pivô aleatório para quicksort pode ser mais simples do que uma abordagem de média mediana mais complexa para encontrar um bom pivô, 
e a probabilidade de maus pivôs serem continuamente escolhidos (aleatoriamente) e degradar o desempenho do quicksort para O(n^2) é muito pequena.

Os algoritmos aleatórios são classificados como algoritmos "Monte Carlo" ou "Las Vegas", a partir de dois locais de jogo bem conhecidos. 
Um algoritmo de Monte Carlo joga com exatidão: pode gerar uma resposta errada (ou, no caso acima, uma árvore binária desequilibrada). Um algoritmo de Las Vegas sempre
gera uma resposta correta, mas pode levar muito tempo para terminar.

Outro exemplo bem conhecido de um algoritmo aleatório é o algoritmo de teste de primalidade de Miller-Rabin. Cada iteração produzirá "não primo" ou "talvez primo". Enquanto "não primo" é certo, o "talvez primo" está correto com probabilidade de pelo menos 1/2. Ou seja, existem não primos para os quais "talvez primo" ainda será produzido. Ao executar muitas iterações de Miller-Rabin, podemos tornar a probabilidade de falha (ou seja, gerar "talvez primo" para um número composto) tão pequena quanto gostaríamos. Se passar 200 iterações, podemos dizer que o número é composto com probabilidade no máximo 1/(2^200).


Outra área em que a aleatoriedade desempenha um papel é chamada "O poder de duas escolhas aleatórias". Embora inicialmente a pesquisa tenha sido aplicada ao balanceamento de carga, ela se mostrou amplamente aplicável a vários problemas de seleção. A idéia é que, em vez de tentar encontrar a melhor seleção dentre um grupo de itens, escolha dois aleatoriamente e selecione o melhor. Voltando ao balanceamento de carga (ou cadeias de tabelas de hash), o poder de duas opções aleatórias reduz a carga esperada (ou o comprimento da cadeia de hash) dos itens O(log n) para O(log log n)
Itens. Para obter mais informações, consulte [The Power of Two Random Choices: A Survey of Techniques and Results (https://www.eecs.harvard.edu/~michaelm/postscripts/handbook2001.pdf)

algoritmos aleatórios:
     outros algoritmos de armazenamento em cache
     aproximações estatísticas (frequentemente dependem do tamanho da amostra e não do tamanho da população)

  TODO: lote para reduzir a sobrecarga: https://lemire.me/blog/2018/04/17/iterating-in-batches-over-data-structures-can-be-much-faster/

TODO: - Algorithm Design Manual: http://algorist.com/algorist.html
       - Como resolvê-lo por computador
       - até que ponto é este livro "como escrever algoritmos"? Se você vai mudar
       o código para acelerar, por definição, você está escrevendo novos algoritmos. Então ... talvez?

### Entradas para avaliação de desempenho

As contribuições do mundo real raramente correspondem ao "pior caso" teórico. 
O benchmarking é vital para entender como o sistema se comporta na produção.

Você precisa saber qual classe de entradas seu sistema verá depois de implantado e seus benchmarks devem usar instâncias extraídas dessa mesma distribuição. 
Como vimos, algoritmos diferentes fazem sentido em diferentes tamanhos de entrada.
Se o seu intervalo de entrada esperado for <100, seus benchmarks devem refletir isso. Caso contrário, escolher um algoritmo ideal para n = 10^6 pode não ser o mais rápido.

Ser capaz de gerar dados de teste representativos. Diferentes distribuições de dados podem provocar comportamentos diferentes em seu algoritmo: 
pense no exemplo clássico de "quicksort é O (n^2) quando os dados são classificados". Da mesma forma, a pesquisa de interpolação é O (log log n) para dados aleatórios 
uniformes, mas O (n) no pior caso. Saber como são as suas entradas é a chave para os benchmarks representativos e para escolher o melhor algoritmo. 
Se os dados que você está usando para testar não são representativos de cargas de trabalho reais, você pode facilmente otimizar um determinado conjunto de dados,
"ajustando demais" seu código para funcionar melhor com um conjunto específico de entradas.

Isso também significa que seus dados de referência precisam ser representativos do mundo real.
 O uso de entradas puramente aleatórias pode distorcer o comportamento do seu algoritmo.
Os algoritmos de cache e compactação exploram distribuições distorcidas ausentes
em dados aleatórios e, portanto, terá um desempenho pior, enquanto uma árvore binária executará
melhor com valores aleatórios, pois eles tendem a manter a árvore equilibrada. (Isto é
a ideia por trás de uma treap, a propósito.)

Por outro lado, considere o caso de testar um sistema com um cache. 
Se sua entrada benchmark consiste apenas em uma única consulta, então cada solicitação atingirá o
cache, fornecendo uma visão potencialmente muito irreal de como o sistema se comportará
no mundo real com um padrão de solicitação mais variado.

Além disso, observe que alguns problemas que não são aparentes no seu laptop podem estar visíveis
depois de implantar na produção e atingir 250k reqs / segundo em um núcleo de 40
servidor. Da mesma forma, o comportamento do coletor de lixo durante o benchmarking
pode deturpar o impacto no mundo real. Existem casos (raros) em que um
O microbenchmark mostrará uma desaceleração, mas o desempenho no mundo real melhora.
Microbenchmarks podem ajudar a empurrá-lo na direção certa, mas ser capaz de
testar completamente o impacto de uma mudança em todo o sistema é melhor.

Escrever boas referências pode ser difícil.

* <https://timharris.uk/misc/five-ways.pdf>

Use média geométrica para comparar grupos de benchmarks.

* <https://www.cse.unsw.edu.au/~cs9242/current/papers/Fleming_Wallace_86.pdf>

Avaliando a precisão do benchmark:

* <http://www.brendangregg.com/blog/2018-06-30/benchmarking-checklist.html>

### Melhoria do programa.

A melhoria do programa costumava ser uma forma de arte, mas os compiladores ficaram melhores. Portanto, agora os compiladores podem otimizar o código melhor do que complicar aquele código. 
O compilador Go ainda tem um longo caminho a percorrer para corresponder ao gcc e ao clang, mas isso significa que você precisa 
ter cuidado ao ajustar e principalmente ao atualizar as versões de Go para que seu código não se torne "pior".
Definitivamente, há casos em que os ajustes para solucionar a falta de uma otimização específica do compilador
façam o código se tornar mais lento depois que o compilador foi aprimorado.


Minha implementação de cifra RC6 teve um aumento de velocidade de 10% para o loop interno apenas mudando para `encoding / binary` e` math / bits` em vez de usar uma de minhas versões feitas manualmente.

Da mesma forma, o pacote `compress / bzip2` foi acelerado ao mudar para [código mais simples que o compilador conseguiu otimizar] (https://github.com/golang/go/commit/9eb219480e8de08d380ee052b7bff293856955f8)

Se você estiver trabalhando em torno de um tempo de execução específico ou geração de código do compilador sempre documente sua alteração com um link para a edição anterior. Este link permitirá que você revisite rapidamente sua otimização depois que o bug for corrigido.

Lute contra a tentação de "dicas de desempenho" baseadas no folclore cult, ou até
generalizar demais a partir de sua própria experiência. Cada bug de desempenho precisa ser
abordado por seus próprios méritos. Mesmo se algo funcionou anteriormente, certifique-se de criar um perfil para garantir que a correção ainda seja aplicável. Seu trabalho  pode orientá-lo, mas não aplique otimizações anteriores às cegas.

A melhoria do programa é um processo iterativo. Continue revisitando seu código e vendo
que mudanças podem ser feitas. Verifique se você está progredindo em cada etapa.
Freqüentemente, uma melhoria permitirá que outras sejam feitas. (Agora que eu não estou
fazendo A, eu posso simplificar B fazendo C em vez disso). Isso significa que você precisa manter-se
olhando para a foto inteira e não fique muito obcecado com um pequeno conjunto de
linhas.

Depois de escolher o algoritmo certo, a melhoria do programa é o processo de
melhorar a implementação desse algoritmo. Na notação Grande-O, isso é
o processo de redução das constantes associadas ao seu programa.

Toda a melhoria no programa ou vai tornar mais rápido algo que era mais lento ou fazer algo que é lento uma menor quantidade de vezes.
As mudanças algorítmicas também se enquadram nessas categorias, mas veremos mudanças menores. Exatamente como você faz isso varia conforme as tecnologias mudam.

Como exemplo, fazer uma coisa lenta rapidamente pode ser substituir SHA1 ou `hash/fnv1` por uma função hash mais rápida. Fazer um processo lento menos vezes pode ser salvar o resultado do cálculo de hash de um arquivo grande para que você não precise fazer isso várias vezes.

Mantenha comentários. Se algo não precisar ser feito, explique o porquê. Freqüentemente, ao otimizar um algoritmo, você descobrirá etapas que não precisam ser executadas sob algumas circunstâncias. Documente-as. Outra pessoa pode pensar que é um bug.

> Programas vazios dão a resposta errada em pouco tempo.
>
> É fácil ser rápido se você não precisa estar correto.

A "correção" pode depender do problema. Os algoritmos heurísticos mais corretos na maioria das vezes podem ser rápidos, assim como os algoritmos que adivinham e melhoram, permitindo que você pare quando atingir um limite aceitável.


Casos comuns de cache:

Todos conhecemos o memcache, mas também existem caches em processo. O uso de um cache em processo economiza o custo da chamada de rede e o custo da serialização. Por outro lado, isso aumenta a pressão do GC, pois há mais memória para acompanhar. Você também precisa considerar estratégias de remoção, invalidação de cache e segurança de threads. 
Um cache externo geralmente lida com o despejo, mas a invalidação do cache permanece um problema. As condições de corrida também podem ser um problema com caches externos, pois se torna um estado mutável efetivamente compartilhado entre goroutines diferentes no mesmo serviço ou até diferentes instâncias de serviço se o cache externo for compartilhado.

Um cache salva as informações que você acabou de gastar em tempo de computação, na esperança de poder reutilizá-las novamente em breve e economizar tempo. Um cache não precisa ser complexo. Mesmo o armazenamento de um único item - a consulta / resposta vista mais recentemente - pode ser uma grande vitória, como visto no exemplo `time.Parse ()` abaixo.

Com  caches, é importante comparar o custo (em termos da complexidade real do relógio e do código) da sua lógica de cache para simplesmente buscar ou recalcular os dados. 

Os algoritmos mais complexos que oferecem taxas de acerto mais altas geralmente não são baratos. A remoção aleatória de cache é simples e rápida e pode ser eficaz em muitos
casos. Da mesma forma, a inserção aleatória * de cache * pode limitar seu cache apenas a itens populares com lógica mínima. Embora estes possam não ser tão eficazes quanto os
algoritmos mais complexos, a grande melhoria será adicionar um cache em primeiro lugar: escolher exatamente qual algoritmo de cache oferece apenas pequenas melhorias.

É importante avaliar sua escolha do algoritmo de remoção de cache com rastreamentos do mundo real. Se, no mundo real, as solicitações repetidas são suficientemente raras, pode
ser mais caro manter as respostas em cache do que simplesmente recalculá-las quando necessário. Eu tive serviços em que o teste com dados de produção mostrou que nem um cache ideal valeu a pena. simplesmente não tivemos solicitações repetidas suficientes para fazer sentido a complexidade adicional de um cache.


A taxa de acertos esperados do cache é importante. Você deseja exportar a proporção para sua pilha de monitoramento. A alteração das taxas mostrará uma mudança no tráfego. 
Chegou a hora de revisitar o tamanho do cache ou a política de expiração.

Um cache grande pode aumentar a pressão do GC. No extremo (pouca ou nenhuma remoção, cache de todas as solicitações para uma função cara), isso pode se transformar em [memorização] (https://en.wikipedia.org/wiki/Memoization)

Melhoria do programa:

Melhoria do programa é a arte de melhorar iterativamente um programa em pequenas etapas.
Egon Elbre apresenta seu procedimento:

* Crie uma hipótese de por que seu programa é lento.
* Crie N soluções para resolvê-lo
* Experimente todos e mantenha o mais rápido.
* Mantenha o segundo mais rápido por precaução.
* Repetir.

As afinações podem assumir várias formas.

* Se possível, mantenha a antiga implementação para teste.
* Se não for possível, gere casos de teste de ouro suficientes para comparar a saída.
* "Suficiente" significa incluir casos extremos, pois esses são os que provavelmente serão afetados pelo ajuste
com o objetivo de melhorar o desempenho no caso geral.
* Explorar uma identidade matemática:
  * Observe que implementar e otimizar cálculos numéricos é quase seu próprio campo
    * <https://github.com/golang/go/commit/ed6c6c9c11496ed8e458f6e0731103126ce60223>
    * <https://gist.github.com/dgryski/67e6a7ff94c3a1add30eb26ec0ad8b0f>
    * multiplicação com adição
    * use WolframAlpha, Maxima, sympy e similares para especializar, otimizar ou criar tabelas de pesquisa
    * (Além disso, https://users.ece.cmu.edu/~franzf/papers/gttse07.pdf)
    * movendo-se da matemática do ponto flutuante para a matemática inteira
    * ou mandelbrot removendo sqrt ou lttb removendo abs, `a <b / c` =>` a * c <b`
    * considere diferentes representações numéricas: ponto fixo, ponto flutuante, números inteiros (menores),
    * extravagante: números inteiros com acumuladores de erro (por exemplo, linha e círculo de Bresenham), números com várias bases / sistemas de números redundantes
  * "pague apenas pelo que você usa, não pelo que você poderia ter usado"
    * zero apenas parte de uma matriz, em vez da coisa toda
  * melhor feito em pequenas etapas, algumas declarações de cada vez
  * checagens baratas antes de checagens caras:
    * por exemplo, strcmp antes da regexp, (q.v., filtro de bloom antes da consulta)
    "faça coisas caras menos vezes"
  * casos comuns antes de casos raros
    ou seja, evite testes extras que sempre falham
  * desenrolar ainda eficaz: https://play.golang.org/p/6tnySwNxG6O
    * tamanho do código. sobrecarga de teste de ramificação vs
  * usar deslocamentos em vez de atribuição de fatia pode ajudar com verificações de limites, dependências de dados e geração de código (menos para copiar no loop interno).
  * remova verificações de limites e verificações nulas dos loops: https://go-review.googlesource.com/c/go/+/151158
  * outros truques para o passe de prova
  * é aqui que pedaços de prazer para um hacker caem

Muitas dicas de desempenho do folclore para ajuste dependem de compiladores de otimização insuficiente e incentivam o programador a fazer essas transformações manualmente. Os
compiladores usam deslocamentos de bits em vez de multiplicar ou dividir por uma potência de dois há 15 anos - ninguém deve fazer isso manualmente. Da mesma forma, içar
cálculos invariantes a partir de loops, desenrolamento básico do loop, eliminação de subexpressão e muitos outros são feitos automaticamente pelo gcc e clang e similares. O compilador da Go faz muitas delas e continua a melhorar. Como sempre, faça uma referência antes de se comprometer com a nova versão.

As transformações que o compilador não pode fazer dependem de você saber coisas sobre o algoritmo,
seus dados de entrada, invariantes em seu sistema e outras suposições que você pode fazer, 
além de levar em consideração esse 
conhecimento implícito para remover ou alterar as etapas na estrutura de dados.

Toda otimização codifica uma suposição sobre seus dados. 
Estes *devem* ser documentados e, melhor ainda, testados.
Essas suposições serão onde o seu programa falha, diminui a velocidade
ou começa a retornar dados incorretos à medida que o sistema evolui.

As melhorias no ajuste do programa são cumulativas. 5x 3% de melhorias são 15% de melhoria. 
Ao fazer otimizações, vale a pena pensar na melhoria de desempenho esperada.
Substituir uma função de hash por uma mais rápida é uma constante melhoria do fator.

Compreender seus requisitos e onde eles podem ser alterados pode levar a melhorias de desempenho. 
Um problema apresentado no canal \#performance Gophers Slack foi a quantidade de tempo gasto na criação de um identificador exclusivo para um mapa de pares de chave/valor de 
string. A solução original era extrair as chaves, classificá-las e passar a sequência resultante para uma função hash. A solução melhorada que encontramos foi o hash individual
das chaves/valores à medida que foram adicionados ao mapa, e então xor todos esses hashes juntos para criar o identificador.

Aqui está um exemplo de especialização.


Digamos que estamos processando um grande arquivo de log por um único dia,
e cada linha começa com um carimbo de hora.

```
Sun  4 Mar 2018 14:35:09 PST <...........................>
```

Para cada linha, nós iremos chamar `time.Parse()` para transformá-la em época. Se
o perfilamento nos mostra que `time.Parse()`  é um gargalo, nós temos algumas opções para
speed things up.

A mais fácil é manter um cache contendo uma única linha, o tempo anterior e a época associada. Enquanto o nosso arquivo de log tiver múltiplas linhas para um mesmo segundo, teremos uma vitória. Para o caso de um arquivo de log com 10 milhões de linhas, essa estratégia reduz o número de chamadas caras `time.Parse()` de 10.000.000 to 86.400 -- uma para cada segundo único.

TODO: exemplo de código para cache de item único

Podemos fazer mais? Como sabemos exatamente em que formato os carimbos de data e hora estão *e* em que todos caem em um único dia, podemos escrever uma lógica de análise de 
tempo personalizada que leva isso em consideração. Podemos calcular a época da meia-noite, extrair horas, minutos e segundos da sequência do carimbo de data/hora - todos eles
estarão em desvios fixos na sequência - e fazer algumas contas inteiras.

TODO: exemplo de código para versão de deslocamento de string

Nas minhas avaliações, isso reduziu o tempo de análise de 275ns/op para 5ns/op. (Obviamente, mesmo com 275 ns/op, é mais provável que você seja bloqueado na E/S e não na CPU para analisar o tempo.)

O algoritmo geral é lento porque precisa lidar com mais casos. Seu algoritmo pode ser mais rápido porque você sabe mais sobre o seu problema. Mas o código está mais intimamente 
ligado ao que você precisa. É muito mais difícil atualizar se o formato da hora mudar.

Otimização é especialização, e códigos especializados são mais frágeis de serem alterados que códigos de uso geral.

As implementações da biblioteca padrão precisam ser "rápidas o suficiente" para a maioria dos casos. Se você tiver necessidades de desempenho mais altas, provavelmente 
precisará de implementações especializadas.

Faça um perfil regularmente para garantir o rastreamento das características de desempenho do seu sistema e esteja preparado para otimizar novamente conforme o tráfego muda. 
Conheça os limites do seu sistema e tenha boas métricas que permitem prever quando você atingirá esses limites.

Quando o uso do aplicativo é alterado, diferentes partes podem se tornar pontos de acesso. Revise as otimizações anteriores e decida se ainda valem a pena e volte para um 
código mais legível, quando possível. Eu tinha um sistema que otimizava o tempo de inicialização do processo com um conjunto complexo de mmap, refletir e inseguro. Depois que
alteramos a maneira como o sistema foi implantado, esse código não era mais necessário e eu o substituí por operações regulares de arquivos muito mais legíveis.

### Resumo do fluxo de trabalho de otimização ( Ou melhoria de perfomance!)

Todas as otimizações devem seguir estas etapas:

1. determine seus objetivos de desempenho e confirme que não os está atingindo
1. Avalie para identificar as áreas a serem melhoradas.
    * Pode ser CPU, alocações de heap ou bloqueio de goroutine.
1. Avaliação para determinar a velocidade que sua solução fornecerá usando
    a estrutura de benchmarking integrada (<http://golang.org/pkg/testing/>)
    * Verifique se você está avaliando a coisa certa em seu alvo
      sistema operacional e arquitetura.
1. Avalie novamente depois para verificar se o problema desapareceu
1. use <https://godoc.org/golang.org/x/perf/benchstat> ou
    <https://github.com/codahale/tinystat> para verificar se um conjunto de checagens
    são 'suficientemente' diferentes para que uma otimização valha o acréscimo
    complexidade do código.
1. use <https://github.com/tsenart/vegeta> para carregar serviços http de teste
    (+ outros extravagantes: k6, fortio, fbender)
    - se possível, teste a aceleração / desaceleração além da carga em estado estacionário
1. verifique se seus números de latência fazem sentido

TODO: mencione github.com/aclements/perflock como ferramenta de redução de ruído da CPU

O primeiro passo é importante. Informa quando e onde começar a otimizar.
Mais importante, ele também informa quando parar. Praticamente todas as otimizações
adicione complexidade de código em troca de velocidade. E você pode *sempre* criar código
Mais rápido. É um ato de equilíbrio.


## Garbage Collection

Você paga pela alocação de memória mais de uma vez. O primeiro é obviamente quando você o aloca. Mas você também paga sempre que o garbage collector é executado.

> Reduce/Reuse/Recycle.
> -- <cite>@bboreham</cite>

* Alocações de pilha vs. heap 
* O que causa alocações de heap?
* Compreendendo a análise de escape (e a limitação atual)
* / debug / pprof / heap e -base
* Design da API para limitar alocações:
  * permitir a passagem de buffers para que o chamador possa reutilizar em vez de forçar uma alocação
  * você pode até modificar uma fatia no lugar com cuidado enquanto a digitaliza
  * passar uma estrutura pode permitir que o chamador empilhe a alocação
* redução de ponteiros para reduzir o tempo de verificação do gc
  * fatias sem ponteiro
  * mapas com chaves e valores sem ponteiro
* GOGC
* reutilização de buffer (sync.Pool vs ou personalizado via go-slab, etc)
* fatiamento x deslocamento: as gravações do ponteiro enquanto o GC está sendo executado precisam de uma barreira de gravação: https://github.com/golang/go/commit/b85433975aedc2be2971093b6bbb0a7dc264c8fd
  * nenhuma barreira de gravação se estiver gravando na pilha https://github.com/golang/go/commit/2140975ebde164ea1eaa70fc72775c03567f2bc9
* use variáveis de erro em vez de errors.New () / fmt.Errorf () no site de chamada (desempenho ou estilo? a interface requer ponteiro, para que ele escape para a pilha de qualquer maneira)
* use erros estruturados para reduzir a alocação (passe o valor da estrutura), crie uma string no momento da impressão de erro
* classes de tamanho
* cuidado ao fixar alocação maior com substrings ou fatias menores

## Tempo de execução e compilador

* custo de chamadas via interfaces (chamadas indiretas no nível da CPU)
* runtime.convT2E / runtime.convT2I
* asserções de tipo vs. comutadores de tipo
* defer
* implementações de mapas de casos especiais para ints, strings
   * mapa para byte / uint16 não otimizado; use uma fatia.
   * Você pode falsificar um float64 otimizado com math.Float {32,64} {from,} bits, mas cuidado com os problemas de igualdade de float
   * https://github.com/dgryski/go-gk/blob/master/exact.go diz 100x mais rápido; precisa de benchmarks
* eliminação de verificação de limites
* [] byte <-> cópias de string, otimizações de mapa
* o intervalo de dois valores copiará uma matriz, use a fatia:
   * <https://play.golang.org/p/4b181zkB1O>
   * <https://github.com/mdempsky/rangerdanger>
* use concatenação de strings em vez de fmt.Sprintf sempre que possível; tempo de execução otimizou rotinas para ele

## Unsafe

* E todos os perigos que a acompanham
* Usos comuns para inseguros
* mmap'ing arquivos de dados
   * struct padding
   * mas nem sempre suficientemente rápido para justificar o custo de complexidade / segurança
   * mas "fora da pilha", tão ignorado pelo gc (mas uma fatia sem ponteiros)
* precisa pensar no formato de serialização: como lidar com ponteiros, indexação (mph, cabeçalho do índice)
* desserialização rápida
* protocolo de conexão binária para estruturar quando você já possui o buffer
* string <-> conversão de fatia, [] byte <-> [] uint32, ...
* int para boolear hack inseguro (mas cmov) (mas! = 0 também é livre de ramificação)
* preenchimento:
   - https://dave.cheney.net/2015/10/09/padding-is-hard
   - http://www.catb.org/esr/structure-packing/#_go_and_rust
   - https://golang.org/ref/spec#Size_and_alignment_guarantees
   - https://github.com/dominikh/go-tools structlayout, structlayout-optimize
   - escreva testes para o layout da estrutura com inseguro. Offsetof para perceber quebras de inseguras ou asm

## Dicas comuns com a biblioteca padrão 

* time.After () vaza até disparar; use t: = NewTimer (); t.Stop () / t.Reset ()
* Reutilizando conexões HTTP ...; garantir que o corpo seja drenado (questão nº?)
* rand.Int () e seus amigos são 1) protegidos por mutex e 2) caros para criar
   * considere a geração alternativa de números aleatórios (go-pcgr, xorshift)
* binary.Read e binary.Write usam reflexão e são lentos; faça à mão. (https://github.com/conformal/yubikey/commit/613e3b04ae2eeb78e6a19636b8ff8e9106d2e7bc)
* use strconv em vez de fmt, se possível
* Use `strings.EqualFold (str1, str2)` em vez de `strings.ToLower (str1) == strings.ToLower (str2)` ou `strings.ToUpper (str1) == strings.ToUpper (str2)` para comparar eficientemente cordas, se possível.
* ...

## Implementações alternativas

Substituições populares para pacotes de bibliotecas padrão:

* codificação / json -> [ffjson] (https://github.com/pquerna/ffjson), [easyjson] (https://github.com/mailru/easyjson), [jingo] (https: // github. com / bet365 / jingo) (apenas codificador), etc
* net / http
  * [fasthttp] (https://github.com/valyala/fasthttp/) (mas API incompatível, não compatível com RFC de maneiras sutis)
  * [activationprouter] (https://github.com/julienschmidt/httprouter) (tem outros recursos além da velocidade; nunca vi roteamento nos meus perfis)
* regexp -> [ragel] (https://www.colm.net/open-source/ragel/) (ou outro pacote de expressões regulares)
* serialização
  * encoding / gob -> <https://github.com/alecthomas/go_serialization_benchmarks>
  * protobuf -> <https://github.com/gogo/protobuf>
  * todos os formatos de serialização têm vantagens: escolha um que corresponda ao que você precisa
    - Grave carga de trabalho pesada -> velocidade de codificação rápida
    - Carga de trabalho pesada -> velocidade de decodificação rápida
    - Outras considerações: tamanho codificado, compatibilidade de idioma / ferramentas
    - compensações de formatos binários compactos vs. formatos de texto autoexplicativos
* database / sql -> possui trocas que afetam o desempenho
  * procure drivers que não o utilizem: jackx / pgx, crawshaw sqlite, ...
* gccgo (referência!), gollvm (WIP)
* container / list: use uma fatia (quase sempre)

## cgo

> cgo não é go
> - <cite> Rob Pike </cite>

* Características de desempenho de chamadas cgo
* Truques para reduzir os custos: lote
* Regras sobre a passagem de ponteiros entre Go e C
* arquivos syso (detector de corrida, dev.boringssl)

## Técnicas avançadas

Técnicas específicas para a arquitetura executando o código

* introdução aos caches da CPU
  * falésias de desempenho
  * construir intuição em torno das linhas de cache: tamanhos, preenchimento, alinhamento
  * Ferramentas do SO para visualizar erros de cache (perf)
  * mapas vs. fatias
  * Layouts SOA vs AOS: linha principal x coluna principal; quando você tem um X, precisa de outro X ou de Y?
  * localidade temporal e espacial: use o que você tem e o que está por perto o máximo possível
  * reduzindo a perseguição do ponteiro
  pré-busca explícita de memória; freqüentemente ineficaz; falta de intrínseca significa sobrecarga de chamada de função (removida do tempo de execução)
  * faça os primeiros 64 bytes da sua estrutura contar
* previsão de ramificação
  * remova ramos dos circuitos internos:
    se um {for {}} else {for {}}
    ao invés de
    para {if a {} else {}}
    benchmark devido à previsão de ramificação
    estrutura para evitar ramificação

    se i% 2 == 0 {
        evens ++
    } outro {
        odds ++
    }

    conta [i & 1] ++
    "código sem ramificação", referência; nem sempre mais rápido, mas frequentemente mais difícil de ler
    TODO: classe ASCII conta exemplo, com benchmarks

* a classificação dos dados pode ajudar a melhorar o desempenho por meio da localização do cache e da previsão de ramificação, mesmo levando em consideração o tempo necessário para classificar
* sobrecarga de chamada de função: o inliner está melhorando
* reduz cópias de dados (inclusive para grandes listas repetidas de parâmetros de função)

* Comentário sobre os números de 2002 de Jeff Dean (mais atualizações)
  * cpus ficaram mais rápidos, mas a memória não se manteve

TODO: pouco comentário sobre otimização livre de alinhamento de código (ou não otimização)

## Concorrência

* Descubra quais peças podem ser feitas em paralelo e quais devem ser seqüenciais
* goroutines são baratas, mas não gratuitas.
* Otimizando código multiencadeado
  * compartilhamento falso -> tamanho do pad ao cache da linha
  * compartilhamento verdadeiro -> fragmentação
* Sobreposição com a seção anterior sobre caches e compartilhamento falso / verdadeiro
* Sincronização preguiçosa; é caro, portanto, duplicar o trabalho pode ser mais barato
* coisas que você pode controlar: número de trabalhadores, tamanho do lote

Você precisa de um mutex para proteger o estado mutável compartilhado. Se você tem muitos mutex
contenção, você precisa reduzir o compartilhado ou o mutável. Dois
maneiras de reduzir o compartilhado são 1) fragmentar os bloqueios ou 2) processar independentemente
e combine depois. Para reduzir mutável: bem, faça sua estrutura de dados
somente leitura. Você também pode reduzir o tempo que os dados precisam ser compartilhados, reduzindo
a seção crítica - segure a trava o mínimo necessário. Às vezes, um RWMutex
será suficiente, embora observe que eles são mais lentos, mas permitem vários
leitores em.

Se você estiver compartilhando os bloqueios, tenha cuidado com as linhas de cache compartilhadas. Você precisará preencher para evitar oscilações na linha de cache entre os processadores.


var stripe [8] struct {sync.Mutex; _ [7] uint64} // mutex é de 64 bits; preenchimento preenche o restante do cacheline

Não faça nada caro em sua seção crítica, se puder ajudá-lo. Isso inclui coisas como E / S (que são baratas, mas lentas).

TODO: como decompor o problema por simultaneidade
TODO: razões pelas quais a implementação paralela pode ser mais lenta (sobrecarga de comunicação, o melhor algoritmo é seqüencial, ...)

## Assembly

* Coisas sobre como escrever código de montagem para o Go
* compiladores melhoram; A barra está alta
* substitua o mínimo possível para causar impacto; o custo de manutenção é alto
* boas razões: instruções SIMD ou outras coisas fora do que o Go e o compilador podem fornecer
* muito importante para avaliar: as melhorias podem ser enormes (10x para a rodovia)
  zero (go-speck / rc6 / farm32) ou até mais lento (sem inlining)
* rebenchmark com novas versões para ver se você já pode excluir seu código
   * TODO: link para 1.11 patches removendo código asm
* sempre tenha a versão pure-Go (purego build tag): testing, arm, gccgo
* breve introdução à sintaxe
* como digitar o ponto do meio
* convenção de chamada: tudo está na pilha, seguido pelos valores de retorno.
  - tudo está na pilha, seguido pelos valores de retorno
  - isso pode mudar https://github.com/golang/go/issues/18597
  - https://science.raphael.poss.name/go-calling-convention-x86-64.html
* usando opcodes não suportados pelo asm (asm2plan9, mas isso está ficando mais raro)
* observações sobre por que a montagem em linha é difícil: https://github.com/golang/go/issues/26891
* todas as ferramentas para facilitar isso:
  - asmfmt: gofmt para montagem https://github.com/klauspost/asmfmt
  - c2goasm: converte o assembly de gcc / clang para goasm https://github.com/minio/c2goasm
   - go2asm: converta ir para montagem, você pode vincular https://rsc.io/tmp/go2asm
   - peachpy / avo: assembler de nível superior em python (peachpy) ou Go (avo)
   - diferenças acima
* https://github.com/golang/go/wiki/AssemblyPolicy
* Design do Go Assembler: https://talks.golang.org/2016/asm.slide

## Otimizando um serviço inteiro

Na maioria das vezes, você não recebe uma única rotina vinculada à CPU.
Esse é o caso fácil. Se você possui um serviço para otimizar, precisa procurar
em todo o sistema. Monitoramento. Métricas. Registre muitas coisas ao longo do tempo
para que você possa vê-los piorando e para ver o impacto que seu
mudanças têm em produção.

tip.golang.org/doc/diagnostics.html

* referências para o design do sistema: Livro SRE, design prático do sistema distribuído
* ferramentas extras: mais registro + análise
* As duas regras básicas: acelerar as coisas lentas ou fazê-las com menos frequência.
* rastreamento distribuído para rastrear gargalos em um nível superior
* padrões de consulta para consultar um único servidor em vez de em massa
* seus problemas de desempenho podem não ser o seu código, mas você precisará contorná-los de qualquer maneira
* https://docs.microsoft.com/en-us/azure/architecture/antipatterns/

## Ferramentas

### Introdução aoProfilingpr

Este é um guia rápido para usar as ferramentas do pprof. Existem muitos outros guias disponíveis sobre isso.
Confira https://github.com/davecheney/high-performance-go-workshop.

TODO (dgryski): vídeos?

1. Introdução ao pprof
   * vá ferramenta pprof (e <https://github.com/google/pprof>)
1. Escrever e executar (micro) benchmarks
   * pequenos, como testes de unidade
   * perfil, extrair código quente para benchmark, otimizar benchmark, perfil.
   * -cpuprofile / -memprofile / -benchmem
   * 0,5 ns / op significa que ele foi otimizado -> como evitar
   * dicas para escrever boas marcas de microbench (remova trabalhos desnecessários, mas adicione linhas de base)
1. Como ler saída pprof
1. Quais são as diferentes partes do tempo de execução que aparecem
  * malloc, trabalhadores do gc
  * tempo de execução. \ _ ExternalCode
1. Macro-benchmarks (criação de perfil na produção)
   * maiores, como testes de ponta a ponta
   * net / http / pprof, muxer de depuração
   * por amostragem, atingir 10 servidores a 100Hz é o mesmo que atingir 1 servidor a 1000Hz
1. Usando -base para observar diferenças
1. Opções de memória: -inuse_space, -inuse_objects, -alloc_space, -alloc_objects
1. Perfil na produção; localhost + tsh ssh, cabeçalhos de autenticação, usando curl.
1. Como ler gráficos de chama
### Tracer

### Veja algumas ferramentas mais interessantes / avançadas

* outras ferramentas em /x/ perf
* perf (perf2pprof)
* intel vtune / amd codexl / apple instruments
* https://godoc.org/github.com/aclements/go-perf

Apêndice: Implementando documentos de pesquisa

Dicas para implementar documentos: (Para `algoritmo`, leia também` estrutura de dados`)

* Não. Comece com a solução óbvia e estruturas de dados razoáveis.

Os algoritmos "modernos" tendem a ter complexidades teóricas mais baixas, mas alta constante
fatores e muita complexidade de implementação. Um dos exemplos clássicos de
isso é montes de Fibonacci. Eles são notoriamente difíceis de acertar e têm
um enorme fator constante. Existem vários artigos publicados comparando
implementações de heap diferentes em diferentes cargas de trabalho e, em geral, as
ou montes implícitos de 8 árias aparecem sempre no topo. E mesmo nos casos
onde a pilha de Fibonacci deve ser mais rápida (devido a O (1) "chave de diminuição"),
experimentos com o algoritmo de busca profunda da Dijkstra mostram que é mais rápido
quando eles usam a remoção e adição direta de heap.

Da mesma forma, treaps ou skiplists vs. as árvores mais complexas de vermelho-preto ou AVL.
No hardware moderno, o algoritmo "mais lento" pode ser rápido o suficiente ou até
Mais rápido.

> O algoritmo mais rápido pode frequentemente ser substituído por um que é quase tão rápido e muito mais fácil de entender.
>
> - <cite> Douglas W. Jones, Universidade de Iowa </cite>

A complexidade adicional deve ser suficiente para que o retorno valha a pena.
Outro exemplo são os algoritmos de remoção de cache. Algoritmos diferentes podem ter
complexidade muito mais alta, apenas para uma pequena melhoria na taxa de acertos. Claro,
talvez não seja possível testá-lo até que você tenha uma implementação funcional e
integrou-o ao seu programa.

Às vezes, o trabalho tem gráficos, mas é muito parecido com a tendência
publicando apenas resultados positivos, estes tenderão a ser distorcidos em favor de
mostrando o quão bom é o novo algoritmo.

* Escolha o papel certo.
* Procure o artigo que seu algoritmo pretende superar e implementar.

Freqüentemente, artigos anteriores serão mais fáceis de entender e necessariamente terão
algoritmos mais simples.

Nem todos os papéis são bons.

Veja o contexto em que o artigo foi escrito. Determine suposições sobre
hardware: espaço em disco, uso de memória etc. Alguns papéis mais antigos
tradeoffs diferentes que eram razoáveis ​​nos anos 70 ou 80, mas não
necessariamente se aplica ao seu caso de uso. Por exemplo, o que eles determinam ser
memória "razoável" versus trocas de uso de disco. Os tamanhos de memória agora são pedidos de
magnitude maior e os SSDs alteraram a penalidade de latência pelo uso do disco.
Da mesma forma, alguns algoritmos de streaming são projetados para hardware de roteador, o que
pode dificultar a tradução em software.

Verifique se as suposições que o algoritmo faz sobre seus dados são mantidas.

Isso vai demorar um pouco. Você provavelmente não deseja implementar o
primeiro artigo que você encontrar.

* Certifique-se de entender o algoritmo. Isso parece óbvio, mas será
  impossível depurar de outra maneira.

  <https://blizzard.cs.uwaterloo.ca/keshav/home/Papers/data/07/paper-reading.pdf>

  Um bom entendimento pode permitir que você extraia a ideia-chave do documento
  e, possivelmente, aplique exatamente isso ao seu problema, que pode ser mais simples do que
  reimplementando a coisa toda.

* O documento original de uma estrutura ou algoritmo de dados nem sempre é o melhor. Trabalhos posteriores podem ter melhores explicações.

* Alguns trabalhos publicam código fonte de referência com o qual você pode comparar, mas
  1) o código acadêmico é quase universalmente terrível
  2) cuidado com as restrições de licença ("apenas para fins de pesquisa")
  3) cuidado com os erros; casos extremos, verificação de erros, desempenho etc.

  Também procure outras implementações no GitHub: elas podem ter os mesmos (ou diferentes!) Bugs que o seu.

Outros recursos sobre este tópico:
* <https://www.youtube.com/watch?v=8eRx5Wo3xYA>
* <http://codecapsule.com/2012/01/18/how-to-implement-a-paper/>
