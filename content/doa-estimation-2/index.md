---
title: "Detección de escapes modificados: implementación"
draft: false
date: 2025-01-10
description: "Deep-dive en un algoritmo de estimación de dirección de arribo e implementación."
tags:
  - doa-estimation
  - signal-processing
---


Partiendo del posteo anterior, y recapitulando el algoritmo, hoy vamos a implementar lo siguiente:

1. Aplicar la transformada de Fourier sobre ventanas solapadas de tiempo de las señales de los micrófonos
2. Buscamos las zonas de frecuencia donde hay sólo una fuente
3. Aplicamos la detección de origen de dirección de arribo en cada zona
4. Juntamos las predicciones y consolidamos las estimaciones de los orígenes 

La idea es poder tomarnos un tiempo para entender cómo funciona el algoritmo de detección de dirección de arribo, tanto de la parte de implementación como en la derivación matemática donde se pueda (para mi humilde cerebro de ingeniero y no matemático).

### Notas sobre los snippets de código

Modelé el detector como una clase para poder agrupar más fácilmente los parámetros compartidos entre las funciones. Me gusta pensar el detector como un objeto que se instancia al principio del programa y a medida que llegan observaciones se corre sobre lo nuevo:

```python
detector = Detector(parameters..)
while signals := microphone_array.receive():
    detector.detect(signals)
```

De esta manera, las funciones internas pueden sólo recibir los datos nuevos y argumentos que cambien con cada cálculo, pero los parámetros del detector (como las posiciones de los micrófonos, la velocidad del sonido, la cantidad de bins en los histogramas) no son necesarios de explicitar cada vez. A nivel conceptual, el objeto con los parámetros representa la configuración del hardware y eso es inmutable, mientras que las señales cambian en el tiempo.

El código del resto del post va a tener usos a `self` por esto mismo; si no está indentado es porque está en el código principal de detección, caso contrario serán funciones aparte de la clase. 

### Procesando las señales de los micrófonos

Al recibir las señales de los micrófonos vamos a tener una lista de señales de tamaño $M$. Cada señal en sí misma va a ser un vector de longitud $T \cdot f_s$, $T$ siendo el tiempo total de la grabación (ventana móvil en caso de detección de tiempo real) y $f_s$ la frecuencia de sampleo. Fundamentalmente, es un vector de valores reales, así que para obtener la FFT podemos usar `rfft` que es un poco más rápido que el método general y no devuelve las frecuencias negativas.

Si el arreglo de señales lo recibimos en `mic_signals`, procedemos a obtener las ventanas de tiempo superpuestas -- buscamos aplicar la FFT sobre ventanas de aprox 2048 elementos, y queremos que haya solapamiento entre una ventana y la subsiguiente -- y luego calculamos las transformadas:


```python
self.mic_time_slices = []
for mic_i, signal in enumerate(self.mic_signals):
    self.mic_time_slices.append(list())
    # La función `overlapping_slices` nos genera los intervalos [a, b], [c, d], etc
    # tal que entre dos consecutivos siempre haya un solapamiento fijo y que cada
    # intervalo tenga el tamaño pedido. p.ej. b-c = overlap_size; b-a=slice_size.
    for start, stop in overlapping_slices(
        self.parameters.slice_size, self.parameters.overlap_size, len(signal)
    ):
        self.mic_time_slices[mic_i].append(signal[start:stop])

self.mic_fft_slices = [
    [scipy.fft.rfft(_slice) for _slice in slices]
    for slices in self.mic_time_slices
]

self.freq_bins = scipy.fft.rfftfreq(
    self.parameters.slice_size, 1 / self.parameters.sampling_frequency
)
```

Ahora que tenemos la descomposición en frecuencias para cada ventana de tiempo de la información que llega a cada micrófono, vamos a analizar las zonas de frecuencias para detectar las fuentes de sonido.


### Definición de zona de una sola fuente

La idea de la estrategia es transformar un algoritmo de detección de una sola fuente en uno que pueda detectar múltiples. Para esto primero detecta en qué zonas hay una sola fuente y luego opera el algoritmo sobre ellas. 

Definimos cada zona $\Omega$ como algunas bandas de frecuencia $\omega$ contiguas, por ejemplo, seis frecuencias contiguas de los bins de la FFT (`freq_bins`). Teniendo esta definición, podemos calcular la covarianza _entre dos micrófonos_ en una zona de frecuencias de la siguiente de manera:

$$R_{i,j}^\prime(\Omega) \overset{\underset{\mathrm{def} }{}}{=} \sum_{\omega\in\Omega} \left| X_i(\omega)\cdot X_j(\omega)\right|$$

Como va a haber zonas de mayor intensidad y de menos, necesitamos normalizar para poder tener un criterio uniforme de detección. Para esto armamos un coeficiente de correlación, o sea una covarianza divididas las varianzas[^1]:

[^1]: Con la salvaguarda de que son covarianzas y correlaciones usando L1 y no L2.

$$r^\prime_{i,j} (\Omega) \overset{\underset{\mathrm{def} }{}}{=}
\frac{R\_{i,j}^\prime(\Omega)}{\sqrt{R\_{i,i}^\prime(\Omega)\cdot R\_{j,j}^\prime(\Omega)}}$$

Ahora el criterio de detección es que el promedio a lo largo de todo el array de micrófonos de la correlación entre las observaciones de un micrófono y el adyacente sea mayor a cierto límite:

$$\bar{r^\prime} (\Omega)  \geq 1 - \epsilon$$

En definitiva, implementando las ecuaciones y teniendo en cuenta que el análisis siempre es para la ventana de tiempo actual, nos queda un código como el siguiente:

```python
def covariance(
    self, 
    mic1: int, 
    mic2: int, 
    timestep: int, 
    f_from: int, 
    f_to: int, 
    mic_fft_slices: list,
):
    return np.linalg.norm(
        mic_fft_slices[mic1][timestep][f_from:f_to]
        * mic_fft_slices[mic2][timestep][f_from:f_to],
        ord=1,
    )
def correlation_coefficient(
    self,
    mic1: int,
    mic2: int,
    timestep: int,
    f_from: int,
    f_to: int,
    mic_fft_slices: list,
):
    # Covarianza entre dos micrófonos
    cov = self.covariance(mic1, mic2, timestep, f_from, f_to, mic_fft_slices)

    # Normalizando con la varianza de cada micrófono
    coeff = cov / np.sqrt(
        self.covariance(
            mic1,
            mic1,
            timestep=timestep,
            f_from=f_from,
            f_to=f_to,
            mic_fft_slices=mic_fft_slices,
        )
        * self.covariance(
            mic2,
            mic2,
            timestep=timestep,
            f_from=f_from,
            f_to=f_to,
            mic_fft_slices=mic_fft_slices,
        )
    )
    return coeff
```

Y el criterio nos queda:

```python
def is_single_source_zone(
    self, zone_index: int, timestep: int, mic_fft_slices: list
) -> bool:
    avg = 0
    omega_index = self.parameters.adjacent_zone * zone_index
    for i in range(self.parameters.num_mics):
        next_mic = (i + 1) % self.parameters.num_mics
        avg += (
            (1 / self.parameters.num_mics)
            * self.correlation_coefficient(
                i,
                next_mic,
                timestep,
                omega_index,
                omega_index + self.parameters.adjacent_zone - 1,
                mic_fft_slices,
            )
        )
    return avg >= self.parameters.single_source_threshold
```

### Detección de dirección en la zona

Ahora sí, debemos definir cómo se calcula la dirección de arribo de señal sabiendo que para cierto intervalo de tiempo y en ciertas frecuencias sólo tenemos una fuente. Para esto tenemos que hacer un poco más de matemática 🫠. Partamos del objetivo a maximizar para tener la idea general y vayamos incluyendo las definiciones del paper:

$$\hat{\theta}_\omega = \underset{0\leq \theta \lt 2\pi}{\mathrm{argmax}} \left | \mathrm{CICS}^{(\omega)} (\phi) \right |$$

Enchufamos la definición de CICS:

$$\hat{\theta}\_\omega = 
\underset{0\leq \theta \lt 2\pi}{\mathrm{argmax}} 
\left | \
\sum_{i=1}^M G_{m_i\to m_1}^{(\omega)} (\phi) G_{m_im_{i+1}}(\omega)
\right |$$

El primer componente son los _Phase Rotation Factors_ que sólo dependen de la geometría del conjunto de micrófonos y de la frecuencia a analizar:

$$
G_{m_i\to m_1}^{(\omega)} (\phi) 
\overset{\underset{\mathrm{def} }{}}{=}
\exp\{-j\omega \tau_{m_i\to m_1}(\phi)\}
$$
Ese $\tau$ es la diferencia entre el delay de las señales observado en $\{m_1m_2\}$ y en $\{m_im_{i+1}\}$:

$$\tau_{m_i\to m_1}(\phi)
\overset{\underset{\mathrm{def} }{}}{=}
\tau_{m_1m_2}(\phi) - \tau_{m_im_{i+1}}(\phi)
$$

Que, usando la definición del delay entre dos micrófonos adyacentes:

$$\tau_{m_im_{i+1}} (\theta)
\overset{\underset{\mathrm{def} }{}}{=}
l\sin (A + \frac{\pi}{2} - \theta + (i-1) \alpha)/c$$


Ese $/c$ está tal cual del paper y un poco me pone nervioso que no haya sido acompañante del $l$ 💢 pero en el análisis dimensional se entiende que no podría ser de otra manera para que la cuenta tenga unidad de tiempo. En fin, digresión aparte, usamos esa definición para los componentes:

$$\tau_{m_i\to m_1}(\phi) =
\frac{l}{c} \left(
    \sin (A + \frac{\pi}{2} - \phi + (1-1) \alpha)
    - \sin (A + \frac{\pi}{2} - \phi + (i-1) \alpha)
\right)
$$

$$\tau_{m_i\to m_1}(\phi) =
\frac{l}{c} \left(
    \sin (A + \frac{\pi}{2} - \phi)
    - \sin (A + \frac{\pi}{2} - \phi + (i-1) \alpha)
\right)
$$
y sólo por gusto si definimos $A^\prime \overset{\underset{\mathrm{def} }{}}{=} A + \frac{\pi}{2}$:

$$\tau_{m_i\to m_1}(\phi) =
\frac{l}{c} \left(
    \sin (A^\prime - \phi)
    - \sin (A^\prime - \phi + (i-1) \alpha)
\right)
$$

Bien! Primera parte lista:

$$G_{m_i\to m_1}^{(\omega)} (\phi) =
\exp\left(-j\omega \frac{l}{c} \left(
    \sin (A^\prime - \phi)
    - \sin (A^\prime - \phi + (i-1) \alpha)
\right)\right)$$

Vamos ahora con la otra definición, que es más directa:

$$G_{m_im_{i+1}}(\omega) = \frac{X_i(\omega) \cdot X_{i+1}^\*(\omega)}{\left|X_i(\omega) \cdot X_{i+1}^*(\omega)\right|}$$

(Esta parte es la que más me hace ruido porque en el paper original de los arrays circulares de micrófonos se usa otra definición. Además, hacen distinción entre $\phi$ y $\theta$, cosa que en este paper parece inconsistente. En el funcional original de CICS están _ambos_ como parámetros y recién luego usan $\phi = \theta$ en la maximización, pero este componente queda distinto. En fin, por ahora usamos como está en este paper y después veremos si hay que cambiarlo -- puede haber habido algún error al redactar la publicación.) 

Entonces nos queda:

$$\hat{\theta}\_\omega = 
\underset{0\leq \theta \lt 2\pi}{\mathrm{argmax}} 
\left | \
\sum_{i=1}^M 
\exp\left(-j\omega \frac{l}{c} \left(
    \sin (A^\prime - \phi)
    - \sin (A^\prime - \phi + (i-1) \alpha)
\right)\right)
\frac{X_i(\omega) \cdot X_{i+1}^\*(\omega)}{\left|X_i(\omega) \cdot X_{i+1}^*(\omega)\right|}
\right |$$

Por último para implementar el bicho, como generalmente los métodos de optimización son de minimización y nosotros buscamos maximizar, metemos un $-1$ por ahí para que dé bien el sentido. Ahí vamos:

```python
def negative_cics(
  self,
  phi: float,
  omega_index: int,
  t: int,
  mic_fft_slices: list,
  freq_bins: typing.Sequence,
):
  """Negative Circular Integrated Cross Spectrum"""
  value = 0
  omega = freq_bins[omega_index]
  for i in range(self.parameters.num_mics):
      phase_rotation_factor = np.exp(
          -1j
          * omega
          * (self.parameters.distance_to_next_mic / self.parameters.speed_of_sound)
          * (
              np.sin(self.parameters.A_prime - phi)
              - np.sin(
                  self.parameters.A_prime
                  - phi
                  # Agregamos un +1 porque el código indexa comenzando en 0
                  + (i + 1 - 1) * self.parameters.angle_rotation
              )
          )
      )
      # Pegamos la vuelta en el último micrófono
      cross_power = mic_fft_slices[i][t][omega_index] * np.conj(
          mic_fft_slices[((i + 1) % self.parameters.num_mics)][t][omega_index]
      )

      phase_cross_spectrum = cross_power / np.abs(cross_power)
      value += phase_cross_spectrum * phase_rotation_factor
  return -np.abs(value)
```

¡Fiuf!

### Consolidación de estimaciones

Con la sección anterior ya tenemos un funcional que podemos minimizar para poder encontrar _una_ estimación de dirección de arribo. Pero, de base, tenemos varias zonas de frecuencia y ventanas de tiempo. Esta sección trabaja en cómo se generan esas estimaciones y cómo se juntan hasta tener un solo conjunto de predicciones.

De base, uno de los trucos es que en realidad se hace la estimación anterior varias veces por cada zona. Se toman las $d$ frecuencias más importantes:

```python
def d_highest_peaks(self, freq_from, freq_to, timestep, d, mic_fft_slices):
  magnitudes = {}
  for freq in range(freq_from, freq_to):
      val = 0
      for i in range(self.parameters.num_mics):
          next_mic = (i + 1) % self.parameters.num_mics
          val += np.abs(
              mic_fft_slices[i][timestep][freq]
              * np.conj(mic_fft_slices[next_mic][timestep][freq])
          )
      magnitudes[freq] = val
  return sorted(magnitudes, key=magnitudes.__getitem__)[-d:]
```

Y ahora podemos escribir el loop principal de la detección. Se toman varias ventanas de tiempo para acumular estimaciones (por ejemplo, analizando 1 segundo en total), y se analiza en cada una las zonas de frecuencia definidas. Entonces:

1. Se consideran sólamente las zonas donde hay una sola fuente.
2. Se seleccionan las $d$ frecuencias más importantes de la zona en la ventana de tiempo.
3. Para cada frecuencia importante, se hace la maximización del módulo de CICS.
4. Se guardan las estimaciones en una lista para luego analizar.

Además, agregué un chequeo de que la frecuencia importante siga teniendo un valor importante en la FFT. En el paper no se incluía, pero me daba que se estaban incluyendo muchas frecuencias de muy baja magnitud en vez de considerar las frecuencias importantes. (Historia distinta habría sido si las $d$ frecuencias importantes se hicieran a lo largo de todas las zonas, pero no parece ser el caso.) El código quedaría:

```python 
for t in range(t_from, t_to):
  for zone in range(50):
      if not self.is_single_source_zone(zone, t, self.mic_fft_slices):
          continue

      top_frequency_indices = self.d_highest_peaks(
          zone * self.parameters.adjacent_zone,
          (zone + 1) * self.parameters.adjacent_zone - 1,
          timestep=t,
          d=self.parameters.estimations_per_zone,
          mic_fft_slices=self.mic_fft_slices,
      )

      for frequency_index in top_frequency_indices:
          # Additionally - to avoid spurious DoA estimations
          if np.abs(self.mic_fft_slices[1][t][frequency_index]) < 100:
              continue
          logging.info(
              f"Using frequency {self.freq_bins[frequency_index]} in zone #{zone}"
          )
          self.frequencies_of_interest.append(self.freq_bins[frequency_index])
          logging.debug(
              f"Value: {np.abs(self.mic_fft_slices[1][t][frequency_index])}"
          )
          # for single source zone, detect DoA
          result = scipy.optimize.minimize_scalar(
              self.negative_cics,
              method="bounded",
              bounds=(0, 2 * np.pi),
              args=(frequency_index, t, self.mic_fft_slices, self.freq_bins),
          )
          logging.info(
              f"Frequency {self.freq_bins[frequency_index]} in zone #{zone}"
              f" is voting for angle {np.rad2deg(result.x)}deg"
          )
          self.doa_zone_estimations.append(result.x)
```


### Predicción

La idea de hacer muchas estimaciones rápidas es poder tener en el caso más repetido estimaciones correctas. Para eso se analiza el histograma, con la idea de analizar los picos:

```python
self.bins, self.x = np.histogram(
    np.rad2deg(np.array(self.doa_zone_estimations)),
    bins=np.linspace(0, 360, self.parameters.histogram_bins + 1),
)
```

Ahora, no es sólo cuestión de sacar los picos más altos porque es posible que una fuente que viene del ángulo de $140^\circ$ se haya estimado con valores $130^\circ$, $145^\circ$, etc, para cada una de las frecuencias que componen la señal. Por eso, es necesario eliminar la contribución de esa fuente al histograma. La estrategia acá es similar a la de _Matching Pursuit_; si pensamos al histograma como una suma contribuciones de fuentes más un poco de ruido, lo que podemos hacer es ir detectando esas fuentes y restar su contribución hasta quedarnos sólo con el ruido.  Por ejemplo:

{{< img
  src="atoms.png"
  alt="Histograma con las estimaciones de arribo y una fuente detectada"
  width="100"
  caption="El primer histograma muestra la consolidación de todas las estimaciones, para todas las frecuencias de interés, para todas las zonas de una sola fuente, sobre varias ventanas de tiempo consecutivas. Se ve que hay actividad sobre todos los ángulos, pero con montañas concentradas en distintos picos. En la segunda figura se detecta no sólo el pico sino también la contribución de ese pico al histograma. El siguiente paso es eliminar esa contribución al histograma y seguir buscando fuentes de sonido. Figura tomada de Pavlidi et al. (2013)." >}}

Primero definimos una contribución base con forma de una ventana de Blackman (el tamaño es un parámetro 🙁). Calculamos todas las contribuciones posibles, es decir, tener una ventana en cada posición del histograma.

```python
window = scipy.signal.windows.blackman(self.parameters.Q)
u = np.zeros(self.parameters.histogram_bins)
u[: self.parameters.Q] = window
c = np.roll(u, -self.parameters.Q0)
C = np.array([np.roll(c, k) for k in range(self.parameters.histogram_bins)])
```

La idea ahora es en cada paso calcular cuál de esas contribuciones posibles correlaciona más con el histograma y restar esa ventana hasta que el tamaño de la correlación sea por debajo de un umbral -- es decir, que lo que quede sea básicamente ruido. La correlación, como siempre, queda como un producto interno. En este caso, como tenemos la matriz con todas las transposiciones posibles de la ventana de contribución, tenemos el producto $C \mathbf{y}$ que nos da un vector de correlaciones. Cada correlación nos dice la intensidad con la que el histograma $\mathbf{y}$ calza con la transposición de la ventana de Blackman. Como el algoritmo es iterativo, el histograma se va a ir actualizando a medida que restamos las contribuciones ya encontradas, de mayor a menor.




```python
current_histogram = self.bins
self.atoms = []
self.atom_energies = []
self.window_energy = np.dot(c, c)
self.energy_threshold = self.bins.mean()
self.atom_contributions = []
for j in range(self.parameters.max_sources):
    # El paper original incluye una condición de que las fuentes de sonido deben 
    # estar a cierta distancia entre sí de mínima, pero acá lo obviamos porque la
    # eliminación de la contribución ya debería encargarse de eso.
    corr_window = C.dot(current_histogram)
    position = np.argmax(corr_window)
    # Condición de corte: el nuevo pico es de baja intensidad
    atom_energy = corr_window.max() / self.window_energy
    if atom_energy < self.energy_threshold:
        break
    atom_contribution = C[position, :] * atom_energy
    # Eliminamos la contribución de la fuente actual al histograma y seguimos iterando.
    current_histogram = current_histogram - atom_contribution
    self.atoms.append(position)
    self.atom_energies.append(atom_energy)
    self.atom_contributions.append(atom_contribution)
```

¡De esta manera obtenemos tanto la cantidad de las fuentes como las posiciones de cada una!

### Conclusiones


El objetivo de este post era hacer una revisión en profundidad de cómo puede funcionar un algoritmo de detección de dirección de arribo, tanto viendo la matemática como una implementación. Reconozco que la parte más misteriosa sigue siendo el desarrollo de la ecuación de $\mathrm{CICS}$ -- la definición surge de otro paper, de otro equipo, de hace varios años y toman un par de diferencias -- pero a grandes rasgos la idea de tomar un método que sólo funciona para _una_ sola fuente y generalizarlo me parece que queda mucho más clara ahora. De hecho, queda tan modularizado que es posible reemplazar ese método puntual por otro, siempre que mantenga hipótesis semejantes al actual.

El siguiente texto tiene como objetivo mostrar los resultados de este método y compararlo con otros disponibles en bibliotecas para poder llegar a unas conclusiones :-) 