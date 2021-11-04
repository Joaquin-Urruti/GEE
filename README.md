# GEE
## Este es mi primer post en Github!!

Soy Ingeniero Agronomo y una parte de mi trabajo consiste en hacer ambientaciones de campos para hacer agricultura de precision.

Para hacer mapeos de forma manual el QGIS o para hacer un primer approach de un campo, siempre es una buena practica buscar varias imagenes con distinto nivel de anegamiento, para poder diferenciar las partes mas altas y mas bajas del campo.

Este script sirve para hacer justamente eso. 

Lo que hace es primero calcular el NDWI (un indice que permite diferenciar superficie cubierta de agua) en cada una de las imagenes de la coleccion Sentinel, luego calcula la superficie cubierta de agua en cada imagen (que son los todos los pixeles cuyo valor de NDWI es mayor que 0) y agrega el valor del area anegada a la metadata de cada una. Luego, ordena toda la serie segun el area anegada y selecciona la de mayor, menor, la mediana y dos situaciones intermedias (percentil 75 y 25). Es decir se obtienen 5 imagenes con niveles distintos de anegamiento del area de interes.


https://code.earthengine.google.com/132b535f0c011e683083203af9c2be0e
