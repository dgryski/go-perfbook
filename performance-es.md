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