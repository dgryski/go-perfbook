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

A hierarquia de memória nos computadores modernos confunde um pouco o problema aqui, pois os caches preferem o acesso previsível da varredura de uma fatia ao acesso efetivamente aleatório da perseguição de um ponteiro. Ainda assim, é melhor começar com um bom algoritmo. Falaremos sobre isso na seção específica de hardware.

TODO: estendendo o último parágrafo, mencione a notação O () é um modelo em que cada 
operação tem custo fixo. Essa é uma suposição errada no hardware moderno.

> A luta pode nem sempre ser a mais forte, nem a corrida a mais rápida,
mas essa é a maneira de apostar.
> -- <cite>Rudyard Kipling</cite>

Às vezes, o melhor algoritmo para um problema específico não é um único algoritmo, mas uma coleção de algoritmos especializados para classes de entrada ligeiramente diferentes. Esse "polialgoritmo" detecta rapidamente com que tipo de entrada ele precisa lidar e despacha para o caminho de código apropriado. É isso que o pacote de classificação mencionado acima: determina o tamanho do problema e escolhe um algoritmo diferente. Além de combinar quicksort, classificação de shell,
e classificação de inserção, também rastreia a profundidade de recursão do quicksort e chama heapsort, se necessário. Os pacotes `string` e` bytes` fazem algo semelhante, detectando e se especializando para casos diferentes. Assim como na compactação de dados, quanto mais você souber sobre a aparência de sua entrada, melhor poderá ser sua solução personalizada. Mesmo que uma otimização nem sempre seja aplicável, vale a pena complicar seu código, determinando que é seguro usar e executar lógicas diferentes.

Isso também se aplica aos subproblemas que seu algoritmo precisa resolver. 
Por exemplo, poder usar a classificação radix pode ter um impacto significativo no desempenho ou 
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
  correspondência de Expressão regular de tempo linear


Embora a maioria dos algoritmos seja determinísticp, há uma classe de algoritmos que usam a aleatoriedade como uma maneira de simplificar etapas de tomada de decisão complexas.
Em vez de ter um código que faz a coisa certa, você usa a aleatoriedade para selecionar uma coisa provavelmente não *ruim*. Por exemplo, um treap é uma árvore binária probabilisticamente equilibrada. Cada nó tem uma chave, mas também recebe um valor aleatório. 
Ao inserir na árvore, o caminho de inserção normal da árvore binária é seguido, mas os nós também obedecem à propriedade heap
com base no peso atribuído aleatoriamente a cada nó. Essa abordagem mais simples substitui soluções de 
rotação de árvores complicadas (como árvores AVL e Rubro negras), mas ainda mantém uma árvore equilibrada com inserção/pesquisa de O (log n) "com alta
probabilidade". As Skip lists são outra estrutura de dados simples e semelhante que usa aleatoriedade para produzir "provavelmente" inserção e pesquisas de O (log n).

Da mesma forma, a escolha de um pivô aleatório para quicksort pode ser mais simples do que uma abordagem de média mediana mais complexa para encontrar um bom pivô, 
e a probabilidade de maus pivôs serem continuamente escolhidos (aleatoriamente) e degradar o desempenho do quicksort para O(n^2) é muito pequeno.

Os algoritmos aleatórios são classificados como algoritmos "Monte Carlo" ou "Las Vegas", depois de dois locais de jogo bem conhecidos. 
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
