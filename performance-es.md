# Escribir y optimizar código en Go

Este documento describe las mejores prácticas para escribir código en Go de alto rendimiento.

Si bien se discutirán maneras de optimizar servicios individuales (almacenamiento en caché, etc.), el diseño de sistemas distribuidos de alto rendimiento está fuera del alcance de este trabajo. Ya hay buenos textos sobre monitorización y diseño de sistemas distribuidos. Este tema abarca un conjunto completamente diferente de investigación y concesiones en el diseño.

Todo el contenido está sujeto a licencia bajo CC-BY-SA.

Este libro está dividido en diferentes secciones:

1. Consejos básicos para escribir software que no sea lento.
   * Temas básicos de Ciencias de la Computación
2. Consejos para escribir software rápido.
   * Secciones específicas de Go sobre cómo obtener lo mejor del lenguaje
3. Consejos avanzados para escribir * software realmente * rápido
   * Para cuando tu código optimizado no es lo suficientemente rápido.

Podemos resumir estas tres secciones como:

1. "Sé razonable"
2. "Sé deliberado"
3. "Sé peligroso"
