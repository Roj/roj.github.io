---
title: "Me topé con el algoritmo húngaro"
draft: false
date: 2025-01-31
description: "Estaba implementando una función que pensé que iba a ser casi trivial y entré en un rabbit hole hermoso."
---

Hoy estaba consolidando el código para hacer el benchmark de los algoritmos y encontré que había implementado el mismo fragmento de código de dos maneras distintas. Intentando ver cuál era mejor me di cuenta que.. ¡ambas estaban mal! Me lancé al agujero negro y encontré dos cambios de representaciones que hacen que el problema se vincule con uno ya conocido. 

El contexto es que, para poder comparar dos algoritmos que estiman la dirección de arribo de distintas fuentes de sonido, armé un banco de simulaciones que calculan el error promedio de cada técnica. En cada iteración se generan aleatoriamente las posiciones de las fuentes. Con estas posiciones y las señales en cada fuente se generan las señales en los micrófonos, con los retardos y atenuaciones apropiadas para el viaje que emprendieron esas onditas (ver ecuación [acá]({{< ref "/doa-estimation-1" >}})). El algoritmo toma esas señales, hace cálculos, y devuelve las direcciones estimadas de arribo de cada una de las fuentes. El problema concreto es que esas direcciones de arribo no tienen un orden en particular respecto a las fuentes. El algoritmo sólo ve una suma de señales, ¡y en la suma no hay orden! Para siquiera poder calcular el error del algoritmo, primero habría que entender a cuál fuente se puede referir cada dirección de arribo.

Lo que escribí originalmente fue permutar todas las opciones de asignación y quedarme con la que tuviera un error más pequeño (en grados o radianes):

```python
import itertools
def align_predictions(real_doas: np.ndarray, predicted_doas: np.ndarray) -> np.ndarray:
    min_norm = 1e3 # Seguro ya que 1000 > 2pi o 360
    best_permutation = None
    for permutation in itertools.permutations(range(real_doas.shape[0])):
        new_norm = np.linalg.norm(real_doas - predicted_doas[list(permutation)])
        if new_norm < min_norm:
            best_permutation = list(permutation)
            min_norm = new_norm
    return best_permutation
```

La segunda implementación se dio después, en algún momento, cuando me pareció que en la mayoría de los casos este alineamiento terminaba llevando los pequeños contra los pequeños y así, por lo cual un alineamiento válido sería `sorted(real_doas)` a `sorted(predicted_doas)`. Pero.. ambos están equivocados.

¡El problema es que ambos no consideran que es un círculo! Supongamos que tenemos dos fuentes en las direcciones $179^\circ$ y $359^\circ$ respecto del centro del conjunto de micrófonos. Nuestro algoritmo predijo las direcciones $0^\circ$ y $180^\circ$. Sólo tenemos dos opciones para hacer el alineamiento de los valores reales a los predichos, así que calculemos el error absoluto promedio de cada uno:

$$ \frac{(179-0) + (359-180)}{2} = 179$$ 

$$ \frac{(180-179) + (359-0)}{2} = 180$$

¡y sin embargo la segunda opción es la correcta! $359^\circ$ está a solo un grado de $0^\circ$, y el error promedio debería quedar entonces en sólo $1^\circ$, no en $179^\circ$. 

Una solución horrible que se me ocurrió brevemente fue agregar la opción de sumarle/restarle $2\pi$ o $360^\circ$ a la dirección predicha para ver si conseguía un error menor.. pero esto tendría que hacerse para cada una de los ángulos predichos.. o sumándole el período a algunos y otros no.. entonces la solución de permutaciones que ya de por sí era $O(N!)$ pasaba a tener que considerar si cambiarle el período a cada una de las predicciones, lo cual genera que sea $O(N! \times 2^N)$. Fatal.

Fui a regar las plantas y no se me ocurrió una forma sencilla de que estas diferencias se tuvieran en cuenta en el círculo, que es algo muy fácil de ver:


{{< svg
  src="circle.svg"
  alt="Diagrama"
  caption="El problema de los ángulos visto sólo con los números es difícil, pero al ubicarlos en el círculo es trivial." >}}

Si bien los ángulos van de 0 a 359, en realidad las _diferencias absolutas_ de ángulos van de 0 a 180. Si vemos una diferencia de 181 grados.. en realidad nos conviene ir por el otro lado del círculo y ahí veríamos que la diferencia es de 179 grados. Esto encontré que muchas veces se llama _wrap angle_ y permite decirte si esa diferencia es yendo por un camino (ej. $+90^\circ$) o por el otro camino ($-90^\circ$). Pero, fundamentalmente, ningún camino da más de media vuelta a la circunferencia. Para poder calcular esto lo que hacemos es que los ángulos de más de 180 pasen a ser negativos arrancando en -180 y los ángulos de menos de 180 se mantengan igual:

$$ \mathrm{WrapAngle}(\theta) = (\theta + 180\ \mathrm{mod}\ 360) - 180$$


Finalmente, si sólo nos interesa el camino recorrido, tomamos el absoluto del $\mathrm{WrapAngle}$. Esta transformación al final es sencilla pero me permitió entender mucho más por que tantas bibliotecas tienen por defecto el manejo de ángulos de $-\pi$ a $\pi$ en vez de $0$ a $2\pi$, que es lo que siempre usé en la universidad; deja más fácil la interpretación de los ángulos. Comparando con lo anterior, nos queda que $|\mathrm{WrapAngle}(359-0)| = |-1| = 1$ que es lo que esperábamos :)

Aún tenemos el problema de la alineación de las predicciones. Podemos armar una matriz $D$ donde las filas representen las direcciones reales de las fuentes y las columnas sean las direcciones predichas de las fuentes (en algún orden cualquiera). Entonces los $d_{ij}$ son la distancia de la fuente real $i$ a la predicción $j$. Queremos elegir para cada fila sólo un elemento, sin que se repitan columnas entre filas, de manera de minimizar la suma total de los elegidos. Elegir estos elementos es equivalente a hacer la asignación de la predicción a la fuente. Por ejemplo, para 2 fuentes de sonido:

$$D = \begin{pmatrix}
d_{11} & d_{12} \\\\
d_{21} &  d_{22} \\\\
\end{pmatrix}$$

Una solución súper sencilla sería tomar el valor más pequeño entre los cuatro, hacer la asignación que sugiere, y seguir avanzando con el resto de la matriz cancelando esa fila y columna. Por ejemplo, si $d_{11}$ es el más pequeño, entonces asignamos la primera predicción a la primera fuente, y como ya asignamos la primera fila y la primera columna, para asignar a la segunda fila sólo nos queda la segunda columna. El error total es $d_{11} + d_{22}$. Esto es un algoritmo greedy y, tristemente, no funciona bien en este caso. Basta con que $d_{22}$ sea enorme para que $d_{12} + d_{21}$ sea una solución con menor costo (por ejemplo, que los tres valores sean 1 y el último sea tres).

Buscando en internet un poco sobre el problema encontré el algoritmo Húngaro de 1955. El ejemplo clásico es de asignar trabajos a distintas personas con un costo distinto de cada persona para cada trabajo, y queremos minimizar el costo. El objetivo es encontrar una matriz de permutación[^1] $P$ tal que al rotar las filas de los costos $C$, se disminuye la traza:

[^1]: El algoritmo está formulado tanto en términos de matrices como de grafos. Yo originalmente lo había pensado en forma de grafo (es un grafo bipartito completo y queremos el matching perfecto que sume menor peso de aristas) pero dado que el problema es de juntar puntos por distancia geométrica me pareció más natural volver a la forma matricial. 

$$\underset{P}{\mathrm{min}}\  \mathrm{Tr}(PC)$$

Imagínese, lector, que el algoritmo es tan bueno que tiene una página web dedicada: [hungarianalgorithm.com](https://www.hungarianalgorithm.com). En tu cara, Dijkstra. 


La forma de obtener cuáles tareas asignar a cada trabajador con la matriz me parece bastante elegante y gráfica. En general el algoritmo trabaja con ceros en la matriz como candidatos de la asignación, y si no hay, los va agregando transformando los valores de manera de no cambiar la configuración óptima. Los primeros ceros los genera restando a cada elemento el mínimo de su fila. Luego, si hay alguna columna sin un cero, se le resta a todos los elementos el mínimo de esa columna. De esta manera tenemos al menos $N$ ceros y siempre hay un cero para cada tarea y para cada trabajador. En un caso ideal, estos ceros ya sugieren una asignación óptima, pero no siempre es posible. El mecanismo por el cual detecta si sugiere una asignación completa y en caso negativo introduce más ceros es lo maravilloso del método.

El método _pinta_ filas y columnas de manera de cubrir todos los ceros con la mínima cantidad de líneas. Si se necesitaron menos de N líneas, significa que todavía no se pueden asignar todas las tareas, como en el siguiente caso tomado de [la página](https://www.hungarianalgorithm.com/examplehungarianalgorithm.php):



{{< svg
  src="table.svg"
  alt="Tabla con los costos de las asignaciones. Hay algunos ceros ya asignados por el proceso de eliminación. Hay columnas y filas pintadas por el proceso de cobertura."
  caption="En las filas tenemos los trabajadores y en las columnas las tareas. Hay 5 ceros ya introducidos, pero podemos pintarlos con 3 líneas. Como se necesitan 4 líneas en la condición de corte, todavía no se encontró la solución. Se ve que tanto el W1 como el W3 sólo pueden hacer la tercer tarea, y esta colisión prohibe la solución." >}}

El algoritmo para pintar las filas y columnas no está explicitado en la página, en la [página de Wikipedia](https://en.wikipedia.org/wiki/Hungarian_algorithm#Matrix_interpretation) sí hay un método posible. De todas maneras cualquier algoritmo que las pinte de manera mínima sirve.

Como en el caso de la imagen, a veces es necesario seguir introduciendo ceros. El nuevo cero será el mínimo de los elementos no pintados. Esto es otro paso clave, porque tomar algún elemento pintado simplemente introduciría un cero donde ya se pintó y no generaría una línea nueva. Este mínimo se resta a todos los elementos no pintados, y se les _suma_ a los elementos _pintados dos veces_ (tanto por línea-columna como por línea-fila). En el caso de la imagen, el mínimo es el 6 y los pintados dos veces son los 12 y 90. Con el nuevo cero introducido y los valores ajustados, se vuelven a buscar las líneas y así sucesivamente hasta la condición de corte de haber encontrado N líneas. Esta interpretación visual de las soluciones no satisfactorias me parece lindísima. 

En Python está directamente implementado en scipy:

```python
scipy.optimize.linear_sum_assignment(D)
```

pero no usa el algoritmo Húngaro :( parece que de 1955 hasta acá alguien encontró algo más rápido. Aún así, ya el algoritmo húngaro se puede implementar en $O(N^3)$, mucho mejor que el $O(N!)$ que veníamos manejando al principio.

Me copó esto porque una función que en teoría iba a ser sencilla -- calcular el error de la predicción de la estimación de dirección de arribo de las fuentes de sonido -- me llevó a un mini agujero negro de entender cómo calcular las distancias, cómo modelar el problema y finalmente vincularlo con un algoritmo conocido para la asignación. 

La función para la alineación, finalmente, nos queda[^2]:


```python
import numpy as np
import scipy.optimize

def wrap_angle(angle):
  return (angle+np.pi)%(2*np.pi) - np.pi

def align_predictions(real_doas: np.ndarray, predicted_doas: np.ndarray) -> np.ndarray:
    # Asumimos que vienen en radianes
    D = np.array([
      [wrap_angle(real_i - predicted_j) for predicted_j in predicted_doas]
      for real_i in real_doas
    ])
    row_ind, col_ind = scipy.optimize.linear_sum_assignment(D)
    return predicted_doas[col_ind]
```

Y validamos que funciona con el caso de principio:

```python
real_doas_d = np.array([179, 359])
predicted_doas_d = np.array([0, 180])
real_doas, predicted_doas = np.deg2rad(real_doas_d), np.deg2rad(predicted_doas_d)
np.rad2deg(align_predictions(real_doas, predicted_doas))
# array([180.,   0.])
```

¡Éxito!

[^2]: Fun fact: quise usar `np.tau` en vez de `2*np.pi` y parece que es un [tema caliente](https://github.com/numpy/numpy/pull/9696).

