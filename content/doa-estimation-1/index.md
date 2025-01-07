---
title: "Detección de escapes modificados"
draft: false
date: 2025-01-07
description: "Diseño de una cámara acústica para evitar sonidos fuertes "
tags:
  - doa-estimation
  - signal-processing
---

Vivo al lado de una avenida grande en la capital y es imposible dejar abiertas las ventanas para dormir. Hay demasiado ruido. Las recomendaciones son de hasta ~40dB durante la noche, y [se encontró](https://www.association-of-noise-consultants.co.uk/wp-content/uploads/2017/06/C3-Tools-for-Assessing-Night-Noise-Impact-wide.pdf) una relación clara entre los picos de volumen y la probabilidad de despertarse a la noche (o de pasar a una fase de sueño más ligera). Por ejemplo, en mi casa, registro picos de hasta 70dB en la madrugada, lo que corresponde a un 10% de probabilidad de un microdespertar. Esto genera mala calidad de sueño y, aparte de las consecuencias al día siguiente, a la larga puede ser perjudicial para la salud. 

Un poco de ruido es esperable -- vivimos en una ciudad y no en el medio del campo -- pero cuando los picos de sonidos los generan autos o motos con escapes excesivamente ruidosos modificados a propósito, deja de ser algo inevitable y debería pasar a ser algo regulado. Se [aplican sanciones](https://quedigital.com.ar/politica/aprobaron-la-prohibicion-de-circular-en-motos-con-canos-de-escape-modificados/), se [secuestran los escapes](https://www.bragado.gov.ar/continuan-los-controles-y-secuestros-de-motos-con-escapes-modificados/) y hay simpáticos videos de [cómo los destruyen](https://www.youtube.com/watch?v=41dm6PK1ZH4), pero son controles específicos de tránsito que son difíciles de escalar. 

Una posible solución es tener una cámara acústica, algo así como las cámaras o radares de velocidad pero para el sonido. Un kit que sea fácil de instalar en distintos lugares, detecte los autos o motos que generan excesivo ruido, y reporten las patentes que están en infracción. Como diferencia interesante, es fácil evitar una multa en un radar porque tienen ubicaciones conocidas y basta con frenar un poco, pero si la moto ya está modificada directamente no podrías pasar por ahí hasta no resolverla. 

El objetivo de este proyecto es diseñar un dispositivo (hardware & software) que pueda detectar las distintas fuentes de sonido y medir la intensidad de cada una.

### El problema

Vamos a tener varias fuentes de sonido y cada una produce una señal producen señales $s_g(t)$
($g\in \left\\{ 1\ldots P \right\\}$). 
Tenemos también $M$ micrófonos, cada uno captura una sola señal $x_i(t)$ que va a ser la suma de todo lo que le llega más un poco de ruido $n_i(t)$. En general le llegan las señales $s_g$ atenuadas por algún factor relativo al micrófono $i$ y a la fuente $g$ y retrasadas según la distancia de la fuente al micrófono:

$$
x_i(t) = \sum_{g=1}^P a_{ig} s_g (t - d_{ig}) + n_i(t)
$$

El problema entonces es, a partir de los distintos $x_i(t)$, identificar los $s_g$ distintos y obtener los $d_{ig}$ más probables. En general, como no se puede distinguir la distancia de la señal sólo con esta información (siempre puede haber venido de más lejos pero con mayor amplitud original, generando exactamente la misma observación), lo que se hace es predecir la dirección de arribo (DoA). 

En el caso de vehículos agregamos que $d_{ig}$ no es estático en el tiempo, lo cual introduce problemas como el efecto Doppler y en muchos casos limita la cantidad de observaciones que se pueden utilizar para una estimación.

### Una solución

El paper de Pavlidi, Griffin, Puigt, Mouchtaris (2013) da una solución que me parece bastante elegante. En general en la literatura que había revisado, siempre había algunas hipótesis fuertes que hacían que no sea aplicable el método a mi caso (como conocer las señales de antemano, o que sea una sola fuente, o usar algoritmos demasiado complejos como para que se ejecuten en el hardware). En cambio, este paper diseña una solución que debería ser rápida en ejecución, permite muchas fuentes distintas y no pone restricciones fuertes sobre las señales mismas. Además, el diseño de la estrategia está modularizado y me da la sensación de que podría cambiar un componente por otro de ser necesario.


{{< img
  src="Diagrama.png"
  alt="Diagrama"
  caption="Algoritmo de detección de fuentes de sonido de Pavlidi et al. (2013)." >}}


El espíritu del algoritmo es transformar un algoritmo que detecta una sola fuente en uno que puede detectar muchas, con la hipótesis de que las fuentes no se solapan completamente en su espectro de frecuencias todo el tiempo. Para esto, arma zonas de frecuencia $\Omega$ y busca las zonas donde sólo hay una fuente. En cada una aplica el algoritmo de estimación de origen, y esto lo hace en todas las zonas, teniendo muchas estimaciones. Con esas estimaciones se arma un histograma y luego se buscan las fuentes que sean más probables, es decir, las que aparezcan muchas veces, sobre todo en ventanas de tiempo cercanas. 

Yendo un poco más en específico, se define que en una zona se sospecha que sólo hay una fuente cuando la correlación entre dos micrófonos adyacentes, $r_{ij}^\prime (\Omega)$, supera un umbral preestablecido $1-\epsilon$. El algoritmo de detección define un funcional que llama $\mathrm{CICS}$, circular integrated cross spectrum. Para cada para de micrófonos adyacentes se calcula la fase del espectro (que dependen de la señal recibida) y unos factores de rotación (que dependen del ángulo estimado), y la idea es encontrar el ángulo de dirección de arribo (DoA) que maximiza esa función. No está escrito en términos de _maximum likelihood_ pero me da la sensación de estar maximizando la esperanza dado el parámetro. El paper no ahonda en la derivación, que en realidad es de Karbasi y Sugiyama (2007); tengo que darle un poco más de atención todavía a esa parte. Pero, bueno, esos ángulos de arribo se calculan para cada zona de interés, y además en algunos intervalos de tiempo sucesivos para generar un _pool_ grande de estimaciones de direcciones, y con éste se obtienen las estimaciones más repetidas en el histograma, que vienen a ser las direcciones más probables. Hay que tener en cuenta que también hay que estimar _cuántas_ fuentes hay, así que lo que tiende a hacer es ir removiendo las fuentes estimadas del histograma y cuando ya no hay ningún pico interesante entonces se deja de contar.

### Siguientes pasos

El algoritmo lo tengo implementado pero no me está dando buenos resultados 🙃. Mi idea es escribir detalladamente cómo es la implementación para, en parte, yo tener más clara la lógica detrás. Pero será en la próxima.

### Referencias

Despoina Pavlidi, Anthony Griffin, Matthieu Puigt, Athanasios Mouchtaris. Real-Time Multiple
Sound Source Localization and Counting Using a Circular Microphone Array. IEEE Transactions on Audio, Speech and Language Processing, 2013, 21 (10), pp.2193-2206. 10.1109/TASL.2013.2272524. hal-01367320

Karbasi, Amin, and Akihiko Sugiyama. "A new DOA estimation method using a circular microphone array." 2007 15th European Signal Processing Conference. IEEE, 2007.