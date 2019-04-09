# Escribiendo y optimizando código en Go

Este documento describe las mejores prácticas para escribir código de alto rendimiento en Go.

Si bien se discutirán maneras de optimizar servicios individuales (almacenamiento en caché, etc.), el diseño de sistemas distribuidos de alto rendimiento está fuera del alcance de este trabajo. Ya existen textos detallados sobre monitorización y diseño de sistemas distribuidos. Dicho tema abarca un conjunto completamente diferente de investigación y concesiones en el diseño.

Todo el contenido está sujeto a licencia bajo CC-BY-SA.

Este libro está dividido en diferentes secciones:

1. Consejos básicos para escribir software que no sea lento.
   * Temas básicos de Ciencias de la Computación
2. Consejos para escribir software eficiente.
   * Secciones específicas de Go sobre cómo obtener lo mejor del lenguaje
3. Consejos avanzados para escribir *software realmente* eficiente
   * Para cuando tu código optimizado no sea lo suficientemente eficiente.

Podemos resumir estas tres secciones como:

1. "Sé razonable"
2. "Sé deliberado"
3. "Sé peligroso"

# Cuándo y dónde optimizar

Escribo esta sección primero porque es el paso más importante. Deberías estar haciendo esto?

Toda optimización tiene un coste. Generalmente, este coste se expresa en términos de complejidad de código o carga cognitiva -- un código optimizado es rara vez más simple que una versión sin optimizar.

Pero hay otro aspecto que llamaré la economía de la optimización. Como programador, tu tiempo es valioso. Está el coste de oportunidad de otras cosas que podrías estar haciendo en tu proyecto, `bug` que podrías arreglar, mejoras que podrías agregar. Optimizar cosas es divertido, pero no siempre es la tarea correcta a hacer. El rendimiento de un programa es una característica, pero también lo es terminarlo y cuan correcto está hecho.

Escoge lo más importante en lo que debas trabajar. A veces esto no es una optimización del uso de CPU, sino una de experiencia de usuario. Algo tan simple como agregar una barra de progreso, o modificar una página para que sea más veloz ejecutando cálculos en segundo plano después de mostrarla.

Algunas veces esto será obvio: un informe que se genera cada hora y tarda 3 horas en completar es menos útil que uno que termina en menos de una hora.

Sólo porque algo sea fácil de optimizar no significa que valga la pena hacerlo. Ignorar lo simple es una estrategia de desarrollo válida.

Considera esto como optimizar *tu* tiempo.

Tú decides qué optimizar y cuándo optimizar. Puedes mover la manilla entre "Software Veloz" y "Desarrollo Rápido".

Las personas escuchan y repiten sin pensar que "la optimización prematura es la raíz de todo mal", pero ellos ignoran el contexto completo de la frase.

"Los Programadores gastan una enorme cantidad de tiempo pensando, o preocupandose, por la velocidad de las partes no críticas de sus programas, y estos intentos por ser más eficientes en realidad causan un gran impacto negativo cuando el mantenimiento o la depuración son considerados. Debemos olvidarnos de las pequeñas eficiencias, digamos el 97% del tiempo: la optimización prematura es la raíz de todo mal. Sin embargo, no deberíamos dejar pasar la oportunidad de optimizar en ese otro 3% crítico." -- Knuth

https://www.youtube.com/watch?time_continue=429&v=RT46MpK39rQ

- No ignores las optimizaciones fáciles
- Más conocimiento de algoritmos y estructuras de datos hacen que la optización sea más "fácil" u "obvia"

"Deberías optimizar tu código? Si, pero sólo si el problema es importante, el programa es genuinamente lento, y si hay alguna expectativa de que se puede mejorar mientras se mantenga la exactitud, robustez, y claridad" -- The Practice of Programming, Kernighan and Pike

La optimización prematura también puede afectarte al atarte a ciertas decisiones. El código final puede ser más difícil de modificar si los requerimientos cambian y más difícil de desechar (falacia de coste) si es necesario.

[La estimación de desempeño BitFunnel](http://bitfunnel.org/strangeloop) muestra datos que hacen este equilibrio más explícito. Imagina una plataforma de búsqueda hipotética que necesite 30,000 servidores en varios centros de datos. Estos servidores tienen un coste aproximado de $1,000 USD por año. Si duplicaras la velocidad del software, este cambio puede ahorrarle a la compañia $15M USD por año. Incluso un solo desarrollador trabajando un año completo para mejorar el rendimiento por solo 1% valdría la pena.

En la gran mayoría de casos, el tamaño y velocidad del programa no es el problema. La optimización más fácil es no hacerla. La segunda alternativa más fácil es simplemente comprar mejor hardware.

Cuando hayas decidido cambiar tu programa, sigue leyendo.

### Cambios algorítmicos

Si no estas cambiando los datos, la otra opción más importante es cambiar el código.

Las mayores mejoras es muy posible que vengan de un cambio algorítmico. Esto es equivalente a sustituir bubble sort (`O(n^2)`) con quicksort (`O(n log n)`) o reemplazar un acceso linear a un array (`O(n)`) con una busqueda binaria (`O(log n)`) o una busqueda en un mapa (`O(1)`).

Así es como el software se vuelve lento. Estructuras originalmente diseñadas para un proposito se reusan para algo que no habian sido diseñadas. Esto ocurre gradualmente.

Es importante tener un entendimiento intuitivo de los diferentes niveles de big-O. Elige la estructura de datos para tu problema. Esto no siempre ahorra ciclos de CPU, pero previene problemas de rendimiento que pueden no ser detectados hasta muchos más adelante.

Las clases básicas de complejidad son:

* O(1): acceso a un campo, un array o un mapa

  Consejo: no te preocupes por ellos

* O(log n): busqueda binaria

  Consejo: sólo es un problema si se hace en un bucle

* O(n): bucle simple

  Consejo: lo haces todo el tiempo

* O(n log n): divide y vencerás, ordenación

  Consejo: sigue siendo bastante rapido

* O(n\*m): bucles anidados / cuadrático

  Consejo: ten cuidado y limita el tamaño de tu conjunto de datos

* Cualquier cosa entre cuadrático y subexponencial

  Consejo: no lo ejecutes en un millón de filas

* O(b ^ n), O(n!): exponencial y mayor

  Consejo: buena suerte si tienes más de una o dos docenas de datos

Link: <http://bigocheatsheet.com>

Let's say you need to search through of an unsorted set of data. "I should
use a binary search" you think, knowing that a binary search is O(log n) which
is faster than the O(n) linear scan. However, a binary search requires that
the data is sorted, which means you'll need to sort it first, which will take
O(n log n) time. If you're doing lots of searches, then the upfront cost of
sorting will pay off. On the other hand, if you're mostly doing lookups,
maybe having an array was the wrong choice and you'd be better off paying the
O(1) lookup cost for a map instead.

If your data structure is static, then you can generally do much better than
the dynamic case. It becomes easier to build an optimal data structure
customized for exactly your lookup patterns. Solutions like minimal perfect
hashing can make sense here, or precomputed bloom filters. This also make
sense if your data structure is "static" for long enough and you can amortize
the up-front cost of construction across many lookups.

Choose the simplest reasonable data structure and move on. This is CS 101 for
writing "not-slow software". This should be your default development
mode. If you know you need random access, don't choose a linked-list.
If you know you need in-order traversal, don't use a map.
Requirements change and you can't always guess the future. Make a reasonable
guess at the workload.

<http://daslab.seas.harvard.edu/rum-conjecture/>

Data structures for similar problems will differ in when they do a piece of
work. A binary tree sorts a little at a time as inserts happen. A unsorted
array is faster to insert but it's unsorted: at the end to "finalize" you
need to do the sorting all at once.

When writing a package to be used by others, avoid the temptation to
optimize up front for every single use case. This will result in unreadable
code. Data structures by design are effectively single-purpose. You can
neither read minds nor predict the future. If a user says "Your package is
too slow for this use case", a reasonable answer might be "Then use this
other package over here". A package should "do one thing well".

Sometimes hybrid data structures will provide the performance improvement you
need. For example, by bucketing your data you can limit your search to a
single bucket. This still pays the theoretical cost of O(n), but the constant
will be smaller. We'll revisit these kinds of tweaks when we get to program
tuning.

Two things that people forget when discussion big-O notation:

One, there's a constant factor involved. Two algorithms which have the same
algorithmic complexity can have different constant factors. Imagine looping
over a list 100 times vs just looping over it once. Even though both are O(n),
one has a constant factor that's 100 times higher.

These constant factors are why even though merge sort, quicksort, and
heapsort all O(n log n), everybody uses quicksort because it's the fastest.
It has the smallest constant factor.

The second thing is that big-O only says "as n grows to infinity". It talks
about the growth trend, "As the numbers get big, this is the growth factor
that will dominate the run time." It says nothing about the actual
performance, or how it behaves with small n.

There's frequently a cut-off point below which a dumber algorithm is faster.
A nice example from the Go standard library's `sort` package. Most of the
time it's using quicksort, but it has a shell-sort pass then insertion sort
when the partition size drops below 12 elements.

For some algorithms, the constant factor might be so large that this cut-off
point may be larger than all reasonable inputs. That is, the O(n^2) algorithm
is faster than the O(n) algorithm for all inputs that you're ever likely to
deal with.

This also means you need to know representative input sizes, both for
choosing the most appropriate algorithm and for writing good benchmarks.
10 items?  1000 items?  1000000 items?

This also goes the other way: For example, choosing to use a more complicated
data structure to give you O(n) scaling instead of O(n^2), even though the
benchmarks for small inputs got slower. This also applies to most lock-free
data structures. They're generally slower in the single-threaded case but
more scalable when many threads are using it.

The memory hierarchy in modern computers confuses the issue here a little
bit, in that caches prefer the predictable access of scanning a slice to the
effectively random access of chasing a pointer. Still, it's best to begin
with a good algorithm. We will talk about this in the hardware-specific
section.

> The fight may not always go to the strongest, nor the race to the fastest,
> but that's the way to bet.
> -- <cite>Rudyard Kipling</cite>

Sometimes the best algorithm for a particular problem is not a single
algorithm, but a collection of algorithms specialized for slightly different
input classes. This "polyalgorithm" quickly detects what kind of input it
needs to deal with and then dispatches to the appropriate code path. This is
what the sorting package mentioned above does: determine the problem size and
choose a different algorithm. In addition to combining quicksort, shell sort,
and insertion sort, it also tracks recursion depth of quicksort and calls
heapsort if necessary. The `string` and `bytes` packages do something similar,
detecting and specializing for different cases. As with data compression, the
more you know about what your input looks like, the better your custom
solution can be. Even if an optimization is not always applicable,
complicating your code by determining that it's safe to use and executing
different logic can be worth it.

This also applies to subproblems your algorithm needs to solve. For example,
being able to use radix sort can have a significant impact on performance, or
using quickselect if you only need a partial sort.

Sometimes rather than specialization for your particular task, the best
approach is to abstract it into a more general problem space that has been
well-studied by researchers.  Then you can apply the more general solution to
your specific problem.  Mapping your problem into a domain that already has
well-researched implementations can be a significant win.

Similarly, using a simpler algorithm means that tradeoffs, analysis, and
implementation deals are more likely to be more studied and well understood
than more esoteric or exotic and complex ones.

Simpler algorithms can also be faster.  These two examples are not isolated cases
  https://go-review.googlesource.com/c/crypto/+/169037
  https://go-review.googlesource.com/c/go/+/170322/

TODO: notes on algorithm selection

TODO:
  improve worst-case behaviour at slight cost to average runtime
  linear-time regexp matching
  randomized algorithms: MC vs. LV
    improve worse-case running time
    skip-list, treap, randomized marking,
    primality testing, randomized pivot for quicksort
    power of two random choices
    statistical approximations (frequently depend on sample size and not population size)

 TODO: batching to reduce overhead: https://lemire.me/blog/2018/04/17/iterating-in-batches-over-data-structures-can-be-much-faster/