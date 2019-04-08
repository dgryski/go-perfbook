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

# Cuando y donde optimizar

Escribo esta sección primero porque es el paso más importante. Deberías estar haciendo esto?

Todas las optimizaciones tienen un costo. Generalmente, este costo es expresado en terminos de complejidad de codigo o carga cognitiva -- codigo optimizado es rara vez más simple que una versión sin optimizar.

Pero hay otro aspecto que llamaré la economía de la optimización. Como programador, tu tiempo es valioso. Existe el costo oportuno de algun otro proyecto en el que puedas estar trabajando, que `bug` puedes arreglar, cual mejora puedeas agregar. Optimizar cosas es divertido, pero no siempre es la tarea correcta para escoger. El rendimiento de un programa es una característica, pero también los es terminarlo y cuan correcto está hecho.

Escoge lo más importanted en que debas trabajar. A veces, eso no significa una optimización de CPU, sino una de experiencia de usuario. Algo tán simple como agregar una barra de progreso, o modificar la página para que sea más veloz mediante cálculos en el fondo después de reproducir la página.

Algunas veces esto será obvio: un reporte de cada hora que demora 3 horas en completar es menos ùtil que uno que termina en menos de una hora.

Solo porque algo es fácil de optimizar no significa que valga la pena hacerlo. Ignorar lo simple es una estrategia de desarrollo válida.

Considera esto como optimizar *tu* tiempo.

Tú decides que optimizar y cuando hacerlo. Puedes mover la manilla entre "Software Veloz" y "Desarrollo Rápido".

Las personas escuchan y repiten sin pensarlo que "la optimización premature es el origen del problema", pero ellos ignoran el contexto completo del mensaje.

"Programadores desperdician una enorme cantidad de tiempo pensando sobre, o preocupandose sobre, la velocidad de partes no tán críticas de sus programas, y estos intentos de eficiencias en realidad causan un gran impacto negativo cuando mantenimiento o depuración son considerados. Deberiamos olvidarnos sobre eficiencias diminutas, digamos que 97% del tiempo: la optimización premature es el origen del problema. Sin embargo, no deberiamos dejar pasar la oportunidad de optimizar en ese 3% crítico." -- Knuth

https://www.youtube.com/watch?time_continue=429&v=RT46MpK39rQ

- No ignores las optimizaciones fáciles
- Más conocimiento de algoritmos y estructuras de data hacen que la optización sea más "fácil" o "obvio"

"Deberías optimizar tu código? Si, pero sólo si el problema es importante, el programa es genuinamente lento, y si hay alguna expectativa que se puede mejorar mientras se mantenga la exactitud, robustez, y claridad" -- The Practice of Programming, Kernighan and Pike

Optimización premature también puede afectarte al atarte a ciertas decisiones. El código final puede ser más difícil de modificar si los requerimientos cambian y más difícil de desechar(falacia de costo) si es necesario.

[La estimacioón de desempeño BitFunnel](http://bitfunnel.org/strangeloop) muestra datos que hacen esta compensación mas explícita. Imagina una pltaforma de búsqueda hipotética que necesite 30,000 servidores en varios centros de datos. Estos servidores tienen un costo aproximado de $1,000 USD por año. Si duplicaras la velocidad del software, este cambio puede ahorrarle a la compañia $15M USD por año. Incluso un solo desarrollador trabajando un año completo para mejorar el rendimiento por solo 1% valdría la pena.

En la gran mayoría de casos, el tamaño y velocidad del programa no es la preocupación. La optimización más fácil es no hacerla. The segunda alternativa más facil es simplemented comprar mejor hardware.

Cuando hayas decidido cambiar tu programa, sigue leyendo.