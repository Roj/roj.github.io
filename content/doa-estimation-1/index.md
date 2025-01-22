---
title: "Detección de escapes modificados"
draft: false
date: 2025-01-07
description: "Diseño de una cámara acústica para detectar orígenes de sonidos fuertes en las ciudades a la noche."
tags:
  - doa-estimation
  - signal-processing
---

Vivo al lado de una avenida grande en la capital y es imposible dejar abiertas las ventanas para dormir. Hay demasiado ruido. Una noche particularmente molesta medí con un sonómetro un promedio de 55dB, con picos de hasta 70dB, todo durante la madrugada. Como referencia, las recomendaciones son de hasta ~40dB durante la noche, y [se encontró](https://www.association-of-noise-consultants.co.uk/wp-content/uploads/2017/06/C3-Tools-for-Assessing-Night-Noise-Impact-wide.pdf) una relación clara entre los picos de volumen y la probabilidad de despertarse a la noche (o de pasar a una fase de sueño más ligera). Con esa regla, abrir la ventana corresponde a un 10% de probabilidad de un microdespertar! Esto genera mala calidad de sueño y, aparte de las consecuencias al día siguiente, a la larga puede ser perjudicial para la salud. 

Un poco de ruido es esperable -- vivimos en una ciudad y no en el medio del campo -- pero cuando los picos de sonidos los generan autos o motos con escapes excesivamente ruidosos (modificados a propósito), deja de ser algo inevitable y debería ser algo regulado. Se [aplican sanciones](https://quedigital.com.ar/politica/aprobaron-la-prohibicion-de-circular-en-motos-con-canos-de-escape-modificados/), se [secuestran los escapes](https://www.bragado.gov.ar/continuan-los-controles-y-secuestros-de-motos-con-escapes-modificados/) y hay simpáticos videos de [cómo los destruyen](https://www.youtube.com/watch?v=41dm6PK1ZH4), pero son controles específicos de tránsito que son difíciles de escalar (y fáciles de evitar). 

Una posible solución es tener una cámara acústica: algo así como las cámaras o radares de velocidad, pero para el sonido. Se necesitaría un kit que sea fácil de instalar en distintos lugares, como arriba de un semáforo, detecte los autos o motos que generan excesivo ruido, y reporten las patentes que están en infracción. De hecho, esto es un poco mejor que las cámaras de velocidad [^1] ya que es fácil evitar la multa por exceso de velocidad frenando antes de entrar a la zona y acelerando después de pasarla --  pero si un escape está modificado, no se puede pasar nunca por la cámara acústica ya que siempre va a excederse de ruido. Como idea adicional, se podrían tener políticas de ruido variables en el tiempo, de manera que a la noche el límite de ruido sea menor al del día, etcétera.

[^1]: En algunos lugares del mundo ahora se instalan [radares de velocidad promedio](https://en.wikipedia.org/wiki/SPECS_(speed_camera)), que miden el tiempo que le llevó a un vehículo trasladarse una distancia fija. 

El objetivo de este proyecto es diseñar un dispositivo (hardware & software) que pueda detectar las distintas fuentes de sonido y medir la intensidad de cada una. Mi idea es armar una serie de posts describiendo cómo es la idea, la implementación y en general pensarlo como bitácora de un proyecto. Disfruto de leer cómo se arman las cosas, no sólo del resultado final sino de la experiencia de desarrollo, y además, escribir ayuda a organizar las ideas y permite ordenar el progreso. En este primer texto, la idea es introducir el problema y motivar una estrategia de resolución.

### El problema

Vamos a tener $P$ fuentes de sonido, cada una produce una señal $s_g(t)$
($g\in \left\\{ 1\ldots P \right\\}$). 
Tenemos también $M$ micrófonos, cada uno captura una sola señal $x_i(t)$ que va a ser la suma de todo lo que le llega más un poco de ruido $n_i(t)$. En general le llegan las señales $s_g$ atenuadas por algún factor relativo al micrófono $i$ y a la fuente $g$ y desfasadas/retrasadas según la distancia de la fuente al micrófono:

$$
x_i(t) = \sum_{g=1}^P a_{ig} s_g \left (t - \frac{d_{ig}}{c}\right ) + n_i(t)
$$

El problema entonces es, a partir de los distintos $x_i(t)$, identificar los $s_g(t)$ distintos y obtener los $d_{ig}$ más probables. En realidad, como no se puede distinguir la distancia de la señal sólo con esta información (siempre puede haber venido de más lejos pero con mayor amplitud original y un desfase distinto, generando exactamente la misma observación), lo que se hace es predecir la dirección de arribo (DoA). Para estimar la distancia se necesitarían hipótesis adicionales que nos restringen la aplicación, y no son importantes para nuestro caso (por ejemplo, si se conoce la señal de antemano, el desfase ya es una referencia de cuán lejos está).

En el caso de vehículos agregamos que $d_{ig}$ no es estático en el tiempo, lo cual introduce problemas como el efecto Doppler y en muchos casos limita la cantidad de observaciones que se pueden utilizar para una estimación. De momento, si hacemos el análisis de dirección de arribo en una ventana de tiempo corta, como cien milisegundos, el efecto doppler no afecta tanto las observaciones. Esto es también un criterio de elección de algoritmo, ya que si uno necesita más ~7 segundos para tener una estimación razonable, es tiempo suficiente para que la fuente haga media cuadra hasta llegar a la cámara y media cuadra para alejarse, generando los efectos de expansión y compresión de la señal.

### Una solución

El paper de Pavlidi, Griffin, Puigt, Mouchtaris (2013) da una solución que me parece bastante elegante. En general en la literatura que había revisado, siempre había algunas hipótesis fuertes que hacían que no sea aplicable el método a este caso (como conocer las señales de antemano, o que sea una sola fuente, o capacidad de cómputo relativamente grande). En cambio, este paper diseña una solución que debería ser rápida en ejecución, permite muchas fuentes distintas y no pone restricciones fuertes sobre las señales mismas. Además, el diseño de la estrategia está modularizado y me da la sensación de que podría cambiar un componente por otro de ser necesario.


{{< svg
  src="Diagrama.svg"
  alt="Diagrama"
  caption="Algoritmo de detección de fuentes de sonido de Pavlidi et al. (2013)." >}}

El espíritu del algoritmo es transformar un algoritmo que detecta una sola fuente en uno que puede detectar muchas, con la hipótesis de que las fuentes no se solapen completamente en su espectro de frecuencias todo el tiempo. Para esto, arma zonas de frecuencia $\Omega$ y busca las zonas donde sólo hay una fuente. En cada una aplica el algoritmo de estimación de origen, y esto lo hace en todas las zonas, teniendo muchas estimaciones. Con esas estimaciones se arma un histograma y luego se buscan las fuentes que sean más probables, es decir, las que aparezcan muchas veces, sobre todo en ventanas de tiempo cercanas. 

Yendo un poco más en específico:

* Se define que en una zona se sospecha que sólo hay una fuente cuando la correlación entre dos micrófonos adyacentes, $r_{ij}^\prime (\Omega)$, supera un umbral preestablecido $1-\epsilon$. 
* El algoritmo de detección define un funcional que llama $\mathrm{CICS}$ (circular integrated cross spectrum). Para cada para de micrófonos adyacentes se calcula la fase del espectro (que dependen de la señal recibida) y unos factores de rotación (que dependen del ángulo estimado), y la idea es encontrar el ángulo de dirección de arribo (DoA) que maximiza esa función. No está escrito en términos de _maximum likelihood_ pero me da la sensación de estar maximizando la esperanza dado el parámetro. El paper no ahonda en la derivación, que en realidad es de Karbasi y Sugiyama (2007); tengo que darle un poco más de atención todavía a esa parte. 
* Los posibles ángulos de arribo se calculan para cada zona de interés, y además en algunos intervalos de tiempo sucesivos para generar un _pool_ grande de estimaciones de direcciones, y con éste se obtienen las estimaciones más repetidas en el histograma, que vienen a ser las direcciones más probables. 
* Hay que tener en cuenta que también hay que estimar _cuántas_ fuentes hay, así que lo que tiende a hacer es ir removiendo las fuentes estimadas del histograma y cuando ya no hay ningún pico interesante entonces se deja de contar.
* En general todos estos sistemas tienen una resolución de detección tal que no pueden detectar dos fuentes muy cercanas. Esto en el paper está especificado como un hiperparámetro (👎) y tendremos que ver cómo es el balance entre cantidad de datos (ensanchando la ventana de tiempo o agregando más micrófonos), la resolución y el error de detección.

Si todo esto se siente como leer arameo, a no desesperar, ~~para mí también~~ la idea es ir paso por paso en la siguiente publicación.

### Siguientes pasos

El algoritmo lo tengo implementado pero no me está dando buenos resultados 🙃. Mi idea es escribir detalladamente cómo es la implementación para, en parte, yo tener más clara la lógica detrás. Pero será en la próxima.

### Referencias

Despoina Pavlidi, Anthony Griffin, Matthieu Puigt, Athanasios Mouchtaris. Real-Time Multiple
Sound Source Localization and Counting Using a Circular Microphone Array. IEEE Transactions on Audio, Speech and Language Processing, 2013, 21 (10), pp.2193-2206. 10.1109/TASL.2013.2272524. hal-01367320

Karbasi, Amin, and Akihiko Sugiyama. "A new DOA estimation method using a circular microphone array." 2007 15th European Signal Processing Conference. IEEE, 2007.