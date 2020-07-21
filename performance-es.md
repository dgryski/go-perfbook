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

## Cuándo y dónde optimizar

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

https://www.youtube.com/watch?time_continue=429&v=3WBaY61c9sE

- No ignores las optimizaciones fáciles
- Más conocimiento de algoritmos y estructuras de datos hacen que la optización sea más "fácil" u "obvia"

"Deberías optimizar tu código? Si, pero sólo si el problema es importante, el programa es genuinamente lento, y si hay alguna expectativa de que se puede mejorar mientras se mantenga la exactitud, robustez, y claridad" -- The Practice of Programming, Kernighan and Pike

La optimización prematura también puede afectarte al atarte a ciertas decisiones. El código final puede ser más difícil de modificar si los requerimientos cambian y más difícil de desechar (falacia de coste) si es necesario.

[La estimación de desempeño BitFunnel](http://bitfunnel.org/strangeloop) muestra datos que hacen este equilibrio más explícito. Imagina una plataforma de búsqueda hipotética que necesite 30.000 servidores en varios centros de datos. Estos servidores tienen un coste aproximado de $1.000 USD por año. Si duplicaras la velocidad del software, este cambio puede ahorrarle a la compañia $15M USD por año. Incluso un solo desarrollador trabajando un año completo para mejorar el rendimiento por solo 1% valdría la pena.

En la gran mayoría de casos, el tamaño y velocidad del programa no es el problema. La optimización más fácil es no hacerla. La segunda alternativa más fácil es simplemente comprar mejor hardware.

Cuando hayas decidido cambiar tu programa, sigue leyendo.

## Cómo optimizar

### Flujo de trabajo de optimización

Antes de entrar en detalle, hablemos del proceso general de optimización.

La optimización es una forma de refactoring. Pero cada paso, en vez de mejorar algún aspecto del código (código duplicado, claridad, etc...), mejora algún aspecto relacionado con el rendimiento: menor uso de CPU, de memoria, latencia, etc... Estas mejoras generalmente se hacen a costa de la legibilidad. Esto supone que además de un completo conjunto de pruebas unitarias (para garantizar que tus cambios no han roto nada), necesitarás un buen conjunto de benchmarks para garantizar que tus cambios tienen el efecto deseado sobre el rendimiento. Tienes que ser capaz de verificar que tu cambio realmente *está* reduciendo el uso de CPU. A veces, un cambio que pensabas iba a mejorar el rendimiento realmente no tiene impacto o tiene un impacto negativo. Asegúrate de deshacer tus cambios en estos casos.

<cite>[What is the best comment in source code you have ever encountered? - Stack Overflow](https://stackoverflow.com/questions/184618/what-is-the-best-comment-in-source-code-you-have-ever-encountered)</cite>:
<pre>
//
// Dear maintainer:
//
// Once you are done trying to 'optimize' this routine,
// and have realized what a terrible mistake that was,
// please increment the following counter as a warning
// to the next guy:
//
// total_hours_wasted_here = 42
//
</pre>

Los benchmarks que estés usando deben ser correctos y generar medidas reproducibles con cargas de trabajo representativas. Si las ejecuciones individuales tienen variaciones muy altas, hará muy difícil identificar pequeñas mejoras. Necesitarás usar [benchstat](https://golang.org/x/perf/benchstat) o similares tests estadísticos y no podrás valorar los cambios a ojo.

(Ten en cuenta que el uso de tests estadísticos es una buena idea en cualquier caso). Los pasos para ejecutar los benchmarks deben estar documentandos, y cualquier script y herramienta personalizada debe ser añadida al repositorio con instrucciones para ejecutarlos. Se cuidadoso con los grandes conjuntos de benchmarks que tardan mucho en ejecutarse: harán el proceso de desarollo más lento.

Ten también en cuenta que cualquier cosa que puede ser medida puede ser optimizada. Asegúrate de estar midiendo lo correcto.

El siguiente paso es decidir qué vas a optimimizar. Si el objetivo es mejorar el uso de CPU, ¿cuál es una velocidad aceptable? ¿Quieres mejorar el rendimiento actual por 2? ¿Por 10? ¿Puedes describirlo como "un problema de tamaño N en un tiempo menor a T"? ¿Estás intentando reducir el uso de memoria? ¿Por cuánto? ¿Cuánto más lento es aceptable a cambio de reducir el uso de memoria? ¿Qué estás dispuesto a perder a cambio de reducir los requerimientos de espacio?

Optimizar la latencia de un servicio es más complejo. Se han escrito libros enteros explicando como probar el rendimiento de servidores web. El principal problema es que para una única función, el rendimiento es bastante consistente para un problema de un tamaño dado. Para servicios web no tienes un único número. Un conjunto de benchmarks adecuado para un servicio web proporcionará una distribución de latencia para un determinado nivel de peticiones/segundo. Esta charla da una buena visión general de algunos de estos temas:
["How NOT to Measure Latency" by Gil Tene](https://youtu.be/lJ8ydIuPFeU)

TODO: See the later section on optimizing web services

Los objetivos de rendimientos deben ser específicos. (Casi) siempre serás capaz de hacer que algo sea más rapido. Optimizar es frecuentemente un juego de rendimientos decrecientes. Debes saber cuando parar. ¿Cuánto esfuerzo vas a dedicar para conseguir ese pequeño paso final? ¿Cuánto estás dispuesto a ensuciar y a dificultar el mantenimiento del código?

La charla de Dan Luu que se menciona más arriba sobre [estimación del rendimiento de BitFunnel](http://bitfunnel.org/strangeloop) muestra un ejemplo del uso de cálculos aproximados para determinar si tus objetivos de rendimiento son razonables.

TODO: Programming Pearls has "Fermi Problems".  Knowing Jeff Dean's slide helps.

Cuando se empieza el desarrollo, no debes dejar el benchmarking y los objetivos de rendimiento para el final. Es fácil decir "arreglaremos esto más adelante", pero si el rendimiento es realmente importante, será una consideración de diseño desde el principio. Cualquier cambio significativo en la arquitectura requerido para arreglar problemas de rendimiento será demasiado arriesgado cerca de la fecha límite. Ten en cuenta que *durante* el desarrollo, el foco debe estar en diseños, algoritmos y estructura de datos razonables. Optimizar los niveles más bajos del stack debe esperar hasta más adelante en el ciclo de desarrollo cuando se tenga una visión más completa del rendimiento del sistema. Cualquier perfilado global del sistema que se haga cuando el sistema esté incompleto dará una vista sesgada de donde estarán los cuellos de botella una vez que el sistema se finalize.

TODO: How to avoid/detect "Death by 1000 cuts" from poorly written software.
Solution: "Premature pessimization is the root of all evil".  This matches with
my Rule 1: Be deliberate.  You don't need to write every line of code 
to be fast, but neither should by default do wasteful things.

"Pesimismo prematuro es cuando escribes código que es más lento de lo que debería ser, generalmente porque se realiza trabajo adicional innecesario, cuando un código con un nivel de complejidad equivalente será más rápido y fluye de namera natural de tus dedos." -- Herb Sutter

Incluir benchmarking como una parte de CI es difícil debido a los vecinos ruidosos e incluso en el caso de no tener vecinos a diferencias en los entornos de CI. Es difícil controlar las métricas de rendimiento. Un buen término medio es tener benchmarks que son ejecutadas por el desarrollador (en hardware adecuado) e incluidas en el mensaje de commit para commits que tratan especificamente con temas de rendimiento. Para aquellos commits más generales, intenta detectar problemas de rendimiento "a ojo" en la revisión del código.

TODO: how to track performance over time?

Escribe código para el que puedas aplicar benchmark. El perfilado se puede hacer en sistemas de mayor tamaño. Mediante benchmarking quieres probar partes aisladas. Debes poder extraer y preparar un contexto suficente como para que los benchmarks prueben lo necesario y sean representativos.

La diferencia entre tu redimiento objetivo y tu rendimiento actual te dará una idea de por donde empezar. Si sólo necesitas una mejora de un 10-20%, probablemente podrás conseguirla con algunos ajustes en la implementación y pequeños arreglos. Si necesitas mejorar por un factor de 10x o más, entonces no lo vas a conseguir reemplazando una multiplicación con un left-shift. En este caso probablemente tengas que hacer cambios de arriba a abajo en tu stack, posiblemente rediseñando grandes partes del sistema teniendo los objetivos de rendimiento en mente.

Un buen trabajo que afecte al rendimiento requiere conocimiento en muchos niveles diferentes, desde el diseño de sistemas, redes, hardware (CPU, caches, almacenamiento), algoritmos, tuning y debugging. Con tiempo y recursos limitados, considera en que nivel tendrás mayores mejoras: no siempre será un cambio de algoritmo o tuning del programa.

En general, las optimizaciones deben hacerse de arriba a abajo. Optimizaciones a nivel de sistema tendrán más impacto que optimizaciones a nivel de expresión. Asegúrate de estar resolviendo el problema en el nivel apropiado.

Este libro va a hablar sobre todo de como reducir el uso de CPU, el uso de memoria y la latencia. Es importante indicar que raramente podrás conseguir las tres cosas. Quizás el tiempo de CPU sea más rapido, pero ahora tu programa usa más memoria. Quizás necesites reducir el espacio en memoria, pero ahora el programa tarda más.

[La ley de Amdahl](https://en.wikipedia.org/wiki/Amdahl%27s_law) dice que pongamos el foco en los cuellos de botella. Si doblas la velocidad de una rutina que solo ocupa el 5% del tiempo de ejecución, solo resulta en una mejora total del 2.5%. Por otro lado, acelerar una rutima que ocupa el 80% del tiempo en tan solo un 10%, mejorará el tiempo de ejecución en casi un 8%. El perfilado te ayudará a identificar donde realmente se ocupa el tiempo.

Al optimizar, quieres reducir la cantidad de trabajo que la CPU tiene que hacer. Quick sort es más rapido que Bubble sort porque resuelve el mismo problema (ordenar) en menos pasos. Es un algoritmo más eficiente. Has reducido el trabajo que la CPU tiene que hacer para conseguir el mismo resultado.

El tuning de un programa, como las optimizaciones del compilador, generalmente solo resultará en una pequeña reducción del tiempo de ejecución total. Las grandes ganancias vendrán casi siempre de un cambio de algoritmo o de un cambio en la estructura de datos, un cambio fundamental en cómo está organizado tu programa. [La ley de Proebsting](http://proebsting.cs.arizona.edu/law.html) dice que los compiladores doblan su rendimientos cada 18 *años*, un contraste evidente con la (ligeramente malinterpretada) Ley de Moore que dice que el rendimiento del procesador se dobla cada 18 *meses*. Las mejoras en los algoritmos funcionan en magnitudes mayores. Los algoritmos de mixed integer programming, [mejoraron en un factor de 30.000 entre 1991 y 2008](https://agtb.wordpress.com/2010/12/23/progress-in-algorithms-beats-moore%E2%80%99s-law). Para un ejemplo más concreto, considera [este análisis](https://medium.com/@buckhx/unwinding-uber-s-most-efficient-service-406413c5871d) sobre reemplazar un algoritmo geoespacial de fuerza bruta descrito en un blog post de Uber con una especialización más adecuada para la tarea presentada. No hay un cambio en el compilador que resulte en un incremento equivalente del rendimiento.

TODO: Optimizing floating point FFT and MMM algorithm differences in gttse07.pdf

Un profiler quizás te muestre que se pasa mucho tiempo en una rutina en particular. Puede ser que sea una rutina pesada o puede ser una rutina ligera que se llama muchas, muchas veces. En lugar de inmediatamente intentar acelerar esta rutina, mira si puedes reducir el número de veces que se llama o eliminarla completamente. Discutiremos estrategias de optimización más concretas en la siguiente sección.

Las Tres Preguntas de Optimización:

* ¿Hace falta hacer esto? El código más rapido es aquel que nunca se ejecuta.
* Si sí, ¿es este el mejor algoritmo?.
* Si sí, ¿es esta la mejor *implementación* de este algoritmo?.

## Consejos concretos para optimizar

La obra de 1982 de Jon Bentley "Writing Efficient Programs", abordó la optimización de programas como un problema de ingeniería: Mide. Analiza. Mejora. Verifica. Itera. Unos cuantos de sus consejos son aplicados automaticamente por los compiladores. El trabajo de un programador es aplicar las transformaciones que los compiladores *no pueden* hacer.

Hay resumenes del libro:

* <http://www.crowl.org/lawrence/programming/Bentley82.html>
* <http://www.geoffprewett.com/BookReviews/WritingEfficientPrograms.html>

y de las reglas de tuning de programas:

* <https://web.archive.org/web/20080513070949/http://www.cs.bell-labs.com/cm/cs/pearls/apprules.html>

Cuando pienses en los cambios que puedes hacer en tu programa, hay dos opciones básicas: puedes modificar los datos o puedes modificar tu código.

### Modificar los datos

Modificar los datos significa agregar o alterar la representación de los datos
que estas procesando. Desde el punto de vista del rendimiento, algunos de
estos acabaran cambiando la complejidad O() asociada a diferentes aspectos de
la estructura de datos.

Ideas para mejorar tu estructura de datos:

* Campos adicionales

  El clásico ejemplo de esto es almacenar la longitud de una lista enlazada en un
  campo en el nodo raíz. Mantenerla actualizada conlleva un poco más de trabajo,
  pero consultar la longitud se convierte en un simple acceso a un campo en vez de una
  búsqueda transversal con complejidad O(n). Tu estructura de datos puede presentar una
  mejora similar: un poco de mantenimiento en algunas operaciones a cambio de
  mejorar el rendimiento en un caso de uso común.

  De manera similar, almacenar punteros a nodos frecuentemente utilizados en vez
  de realizar búsquedas adicionales. Esto cubre cosas como el link "hacia atrás"
  en una lista doblemente enlazada para hacer que la eliminación de nodos tenga una complejidad O(1).
  Algunas listas de salto guardan un "puntero de búsqueda", donde almacenas un
  puntero a donde recientemente estuviste en tu estructura de datos, bajo la
  suposición de que es un buen punto de partida para la siguiente operación.

* Índices de búsqueda adicionales

  La mayoría de las estructuras de datos están diseñadas para un único tipo de
  consulta. Si necesitas dos tipos de consultas, disponer de una "vista"
  adicional a tus datos puede ser una gran mejora. Por ejemplo, un set de
  structs puede tener un ID primario (integer) que usas para buscar en un
  slice, pero a veces necesitas buscar por un ID secundario (string). En lugar
  de iterar sobre el slice, puedes mejorar tu estructura de datos con un mapa de
  string a ID o directamente al struct en cuestión.

* Información adicional sobre los elementos

  Por ejemplo, mantener un filtro de Bloom de todos los elementos que has
  insertado puede permitirte retornar rápidamente "sin coincidencias" a las
  búsquedas. Estos necesitan ser pequeños y rápidos para no abrumar el resto de
  la estructura de datos. (Si una búsqueda en tu estructura de datos principal
  es barata, el costo del filtro de Bloom superará cualquier ahorro.)

* Si las búsquedas son costosas, agrega una cache

  A mayor escala, una cache interna o externa (como memcache) puede ayudar.
  Puede ser excesivo para una única estructura de datos. Hablaremos más sobre
  caches más adelante.

Este tipo de cambios son útiles cuando los datos que necesitan son baratos de
almacenar y fáciles de mantener actualizados.

Estos son todos ejemplos claros de "realizar menos trabajo" a nivel de
la estructura de datos. Todos cuestan espacio. La mayoría de las veces, si estás
optimizando para CPU, tu programa usará más memoria. Se trata del clásico [space-time trade-off](https://en.wikipedia.org/wiki/Space%E2%80%93time_tradeoff).

Si tu programa utiliza demasiada memoria, también es posible ir por el otro
camino. Reduce el uso de espacio a cambio de una mayor carga computacional. En
lugar de almacenar cosas, calcúlalas cada vez. También puedes comprimir los datos
en memoria y descomprimirlos cuando los necesites.

[Small Memory Software](http://smallmemory.com/book.html) es un libro disponible
online que cubre técnicas para reducir el espacio utilizado por tus programas.
Aunque fue originalmente escrito dirigído a desarrolladores de sistemas
embebidos, sus ideas son aplicables para programas en hardware moderno que
manejen gran cantidad de datos.

* Reorganiza tus datos

  Elimina el padding. Remueve campos extra. Utiliza tipos de datos más pequeños.

* Cambia a una estructura de datos más lenta

  Estructuras de datos más simples frecuentemente presentan menores
  requerimientos de memoria. Por ejemplo, cambiar una estructura tipo arbol con
  uso extensivo de punteros a un slice y búsqueda lineal.

* Compresión a medida para tus datos

  Los algoritmos de compresión dependen fuertemente de qué esté siendo comprimido. Lo mejor es
  elegir uno que se ajuste a tus datos. Si tienes un []byte, entonces algo como snappy, gzip, lz4,
  funciona bien. Para datos de punto flotante existe go-tsz para series temporales y fpc para datos
  científicos. Se ha realizado mucha investigación sobre la compresión de integers, generalmente
  para la obtención de datos en motores de búsqueda. Algunos ejemplos son delta encoding y variantes
  o esquemas más complejos que involucran diferencias xor codificadas con el algoritmo de Huffman.
  También puedes usar tu propio algoritmo optimizado exactamente para tus datos.

  ¿Necesitas inspeccionar los datos o pueden permanecer comprimidos? ¿Necesitas acceso aleatorio o
  sólo streaming? Si necesitas acceso a entradas individuales pero no quieres descomprimirlo todo,
  puedes comprimir los datos en pequeños bloques y mantener un índice que indique qué rangos de
  entradas hay en cada bloque. El acceso a una única entrada solo requiere consultar el
  índice y descomprimir ese bloque pequeño.

  Si tus datos no estan sólo en memoria, sino que vas a escribirlos a disco, ¿qué sucede con la
  migración o agregár o eliminár campos?. Estarás ahora lidiando simplemente con []byte en vez de
  los convenientes tipos estructurados de Go, por lo que necesitaras hacer uso del paquete unsafe
  y considerar opciones de serialización.

Hablaremos más sobre la disposición de datos más adelante.

Las computadoras modernas y la jerarquía de memoria hacen que el compromiso
entre memoria/espacio sea menos claro. Es muy fácil para las tablas de búsqueda estar
"lejos" en memoria (y por lo tanto ser costosas de acceder) haciendo que sea más rápido
simplemente recalcular un valor cada vez que se necesita.

Esto también significa que el benchmarking frecuentemente mostrará mejoras que no
aparecen en producción debido a la contención de cache (e.g., tablas de
búsqueda que estan en la cache del procesador durante el benchmarking pero
que siempre son purgadas cuando son utilizadas por un sistema real.)
El [Jump Hash paper](https://arxiv.org/pdf/1406.2294.pdf) de Google de hecho
abordó esto directamente, comparando la performance de caché con y sin problemas
de contención. (Ver gráficos 4 y 5 del artículo)

TODO: how to simulate a contended cache, show incremental cost

Otro aspecto a considerar es el tiempo de transmisión de datos. Generalmente el
acceso a disco y de red es muy lento, y por lo tanto ser capaz de cargar un
bloque comprimido va a resultar en un proceso mucho más rápido incluso teniendo
en cuenta el tiempo que lleva descomprimirlo. Como siempre, realiza benchmarks.
Un formato binario generalmente va a ser más liviano y rápido de parsear que uno
de texto, pero el coste es que no será legible para un humano.

Para la transferencia de datos, cambia a un protocol menos verboso, o
mejora el API para aceptar consultas parciales. Por ejemplo, usa una consulta
incremental en lugar de forzar a traer siempre el set de datos completo.

### Modificar los algoritmos

Si no estás modificando los datos, la alternativa más importante es modificar el código.

Es muy posible que las mejoras más importantes vengan de un cambio de algoritmo. Esto es equivalente a sustituir bubble sort (`O(n^2)`) con quicksort (`O(n log n)`) o reemplazar un acceso linear a un array (`O(n)`) con una búsqueda binaria (`O(log n)`) o una búsqueda en un mapa (`O(1)`).

Así es como el software se vuelve lento. Estructuras originalmente diseñadas para un propósito se reusan para algo que no habían sido diseñadas. Esto ocurre gradualmente.

Es importante tener un entendimiento intuitivo de los diferentes niveles de big-O. Elige la estructura de datos para tu problema. Esto no siempre ahorra ciclos de CPU, pero previene problemas de rendimiento que pueden no ser detectados hasta mucho más adelante.

Las clases básicas de complejidad son:

* O(1): acceso a un campo, un array o un mapa

  Consejo: no te preocupes por ellos

* O(log n): búsqueda binaria

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

  Consejo: vas a necesitar suerte si tienes más de una o dos docenas de datos

Link: <http://bigocheatsheet.com>

Supongamos que tienes que buscar en un conjunto desordenado de datos. "Debería usar búsqueda binaria" piensas, sabiendo que una búsqueda binaria es O(log n) que es más rapido que el O(n) de una búsqueda linear. Sin embargo, una búsqueda binaria requiere que los datos estén ordenados, lo que significa que tendrás que ordenarlos antes, que tarda O(n log n). Si haces muchas búsquedas, el coste inicial de la ordenación merecerá la pena. Pero, si sobre todo estas haciendo **lookups**, quizás usar un array fue una decisión equivocada y sería mejor usar un mapa con coste O(1).

Si tu estructura de datos es estática, entonces generalmente podrás hacerlo mucho mejor que en el caso de que fuera dinámica. Resultará más facil construir una estructura de datos óptima para tus patrones de búsqueda. Soluciones como minimal perfect hashing pueden tener más sentido aquí, o filtros de Bloom precalculados. Esto también tiene sentido si tu estructura de datos es "estática" durante un periodo largo de manera que puedas amortizar el coste inicial de su construcción en muchas búsquedas.

Escoje la estructura de datos más simple que sea razonable y continúa. Esto es elemental para escribir "software no lento". Este debe ser tu modo de desarrollar por defecto. Si sabes que necesitas acceso aleatorio, no escojas una lista enlazada. Si sabes
que necesitas recorrer los datos en orden, no uses un mapa. Los requerimientos cambian
y no siempre puedes averiguar el futuro. Haz una suposición razonable de la carga de trabajo.

<http://daslab.seas.harvard.edu/rum-conjecture/>

Estructuras de datos para problemas similares diferirán cuando hagan una parte de su trabajo. Un árbol binario se ordena a medida que se insertan elementos. Un array no-ordenado es más rapido al insertar pero no está ordenado: al acabar, para "finalizar", tienes que hacer la ordenación.

Cuando escribas un paquete para ser usado por otros, evita la tentación de optimizar por adelantado para cada caso de uso individual. Esto resultará en código ilegible. Las estructura de datos tienen por diseño un solo proposito. No puedes ni leer mentes ni predecir el futuro. Si un usuario dice "Tu paquete es demasiado lento para este caso de uso", una respuesta razonable puede ser "Entonces usa este otro paquete". Un paquete debe "hacer una cosa bien".

A veces, estructuras de datos hibridas proveerán las mejoras de rendimiento que necesitas. Por ejemplo, agrupando tus datos puedes limitar tu búsqueda a una sola agrupación. Esto todavía tiene un coste teórico de O(n), pero la constante será más pequeña. Volveremos a visitar estos tipos de ajustes cuando lleguemos a la parte de afinar programas.

Dos cosas que la gente olvida cuando se discuten notaciones big-O:

Primero, hay un factor constante. Dos algoritmos que tienen la misma complejidad algorítmica pueden tener diferentes factores constantes. Imagina que iteras una lista 100 veces frente a iterar una sola vez. Aunque ambas son O(n), una de ellas tiene un factor constante que es 100 veces mayor.

Estos factores constantes explican que aunque merge sort, quicksort y heapsort son todos O(n log n), todo el mundo use quicksort porque es el más rapido. Tiene el factor constante más pequeño.

La segunda cosa es que big-O solo dice "a medida que n crece a infinito". Habla de la tendencia de crecimiento, "A medida que los números crezcan, este es el factor de crecimiento que dominará el tiempo de ejecución". No dice nada sobre el rendimiento real o sobre como se comporta cuando n es pequeño.

Con frecuencia hay un punto de corte por debajo del cual un algoritmo más tonto es más rápido. Un buen ejemplo del paquete `sort` de la librería estandar de Go. La mayoría del tiempo usa quicksort, pero hace una pasada con shell sort y luego con insertion sort cuando el tamaño de la partición está por debajo de 12 elementos.

Para algunos algoritmos, el factor constante puede ser tan grande que este punto de corte puede ser mayor que cualquier input razonable. Esto es, el algoritmo O(n^2) es más rapido que el algoritmo O(n) para cualquier input con el que te vayas a encontrar.

Esto también significa que necesitas tener muestras representativas del tamaño de tu input tanto para escoger el algoritmo más apropiado como para escribir buenos benchmarks. ¿10 elementos? ¿1000 elementos? ¿1000000 elementos?

Esto también funciona en sentido contrario: por ejemplo, escoger una estructura de datos más compleja para obtener un crecimiento O(n) en lugar de O(n^2), aunque los benchmarks para inputs más pequeños sean más lentos. Esto también aplica para la mayoría de estructuras de datos que son lock-free. Son generalmente más lentas cuando se usan en un sólo hilo pero más escalables cuando hay muchos hilos usándolas.

La jerarquía de memoria en los ordenadores modernos confunde un poco el tema, en el sentido de que las caches prefieren el predecible acceso linear al recorrer un slice que el acceso aleatorio de seguir un puntero. Aún así, es mejor empezar con un buen algoritmo. Hablaremos más de esto en la sección sobre hardware.

> La pelea no siempre la ganará el más fuerte, ni la carrera el más rapido, pero esa es es la manera de apostar. -- <cite>Rudyard Kipling</cite>

A veces el mejor algoritmo para un problema específico no es un único algoritmo, sino un conjunto de algoritmos especializados en tipos de input ligeramente diferentes. Este "polialgoritmo" primero detecta el tipo de input que tiene que tratar y luego sigue el code path apropiado. De esta manera funciona el paquete `sort` mencionado anteriormente: determina el tamaño del problema y elige un algoritmo distinto. Además de combinar quicksort, shell sort e insertion sort, también controla el nivel de recursividad de quicksort y usa heapsort si es necesario. Los paquetes `string` y `bytes` hacen algo similar, detectando y especializando para diferentes casos. Como con la compresión de datos, cuanto más sepas sobre las características de tu input, mejor será tu solución especifica. Incluso si una optimización no siempre se puede aplicar, complicar tu código determinando que es seguro de usar y ejecutando una lógica diferente puede valer la pena.

Esto también aplica a los subproblemas que tu algoritmo tiene que solucionar. Por ejemplo, poder usar radix sort puede tener un impacto significativo en el rendimiento, o usar quicksort si sólo necesitas una ordenación parcial.

A veces, en vez de una especialización para tu tarea, el mejor enfoque es abstraer la tarea a un categoría de problemas más general que ya haya sido estudiada. Así podrás aplicar la solución más general a tu caso concreto. Mapear tu problema a un dominio con implementaciones bien estudiadas puede resultar en una ganancia significativa.

De manera similar, usar un algoritmo más simple significa que es más probable que las concesiones, analisis y detalles de la implementación hayan sido más estudiados y sean mejor entendidos que en otros algoritmos más esótericos, exóticos y complejos.

Los algoritmos más simples pueden ser más rápidos. Estos dos ejemplos no son casos aislados:
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
