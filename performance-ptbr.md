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
