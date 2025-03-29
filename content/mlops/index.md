---
title: "Todas las formas de deployar un modelo"
draft: false
date: 2025-03-28
description: "Estuve repasando patrones de despliegue para modelos y me encontré con muchos recursos dispersos o malos. Me armé un apunte con todos los patrones principales."
tags:
  - mlops
  - machine-learning
  - notes
---

En el último tiempo me puse a repasar un poco algunos patrones de MLOps para prepararme para entrevistas. Particularmente me parecía interesante investigar los patrones de despliegue de modelos. En mi experiencia las cosas caían en real-time o en batch, lo primero se hacía con una API y lo segundo directamente con las máquinas virtuales usadas como _workers_ del orquestador (o lanzar un job en SageMaker .. Databricks .. etc.), pero seguramente había más patrones para distintas arquitecturas con distintos _trade-offs_. 

Encontré que no había _un_ recurso listando todas las posibles maneras de embeber un modelo en una arquitectura de datos. Peor, me topé con cosas como el libro de _Practical MLOps_ de O'Reilly, un libro que se la pasa explicando cosas que no vienen al caso (como qué es un algoritmo greedy, qué es el TSP, cómo armar un script en Python o una guía para trabajar remoto) y sólo explica tres patrones de despliegue. En otros casos, había blog posts que estaban _muy_ optimizados para el SEO y cada párrafo intentaba mechar la mayor cantidad de palabras clave posible, lo cual era muy engorroso de leer.

Mi objetivo con este apunte es poner en un solo lugar los patrones de la manera más sucinta posible, con un diagrama y para qué sirve, y con links para leer más :)

Si me olvidé de algún patrón, o me equivoqué en alguno, mandame un mail!

## Contexto y arquitecturas

Para dar un poco de contexto del resto de los sistemas sobre los cuales se ubica el modelo predictivo, me sirve hablar sobre dos arquitecturas muy comunes, Lambda y Kappa. En internet hay otras como la `iot-a` y la `Zeta`, pero nunca vi un caso real de ellas así que no las incluyo por el momento.

La arquitectura Lambda es, en mi experiencia, la más común para organizaciones que operan con grandes volúmenes de datos. Si tenemos datos de eventos (nativos o generados por [CDC](https://www.striim.com/blog/change-data-capture-cdc-what-it-is-and-how-it-works/)) y tenemos tanto que poder contestar preguntas históricas (ej., promedio de últimas semanas) como preguntas actuales (valor de cierto campo), podemos separar los flujos para considerar una parte batch y una parte de streams. Por ejemplo, muchas veces los modelos predictivos se entrenan sobre data histórica (parte batch) pero en tiempo real se sirven usando features actuales (con las vistas de tiempo real). También se puede ver como una forma de separar el problema en una parte mutable --la de tiempo real-- y otra inmutable --la batch, porque nunca se modifica el pasado, sólo se incrementa el conjunto-- lo que simplifica mucho el diseño y ataca con correctitud eventual [algunos de los problemas de consistencia, disponibilidad y tolerancia a particiones](http://nathanmarz.com/blog/how-to-beat-the-cap-theorem.html) [^1].

[^1]: Hablando del teorema CAP, me encontré [esta respuesta genérica](https://ferd.ca/beating-the-cap-theorem-checklist.html) para posibles soluciones que me hizo reír.

Insisto acá con que esto es aparte de la arquitectura común de backend, frontend, cliente. El backend puede tener un Postgres y hasta un Redis para datos que cachee o lo que necesite, pero toda esta arquitectura está del lado analítico y no del transaccional.

{{< svg
  src="diagrams/lambda_architecture.svg"
  height="500"
  caption="Ejemplo de arquitectura Lambda estándar. En la parte batch directamente incluímos el patrón de arquitectura Medallion. Generalmente se puede también armar una capa de serving que pueda resolver consultas yendo a cada capa. Las capas pueden estar conectadas; parte de la información batch puede ser replicada en las tablas de tiempo real para facilitar algunas consultas. Las queries que marcamos a la derecha no son de usuarios finales sino de otros servicios de la empresa, para armar reportes, ejecutar modelos predictivos y más." >}}

El objetivo de la parte batch es tener un dataset que se aumenta de manera incremental. Una forma común para resolver esto es la [arquitectura Medallion](https://www.databricks.com/glossary/medallion-architecture) que divide el proceso de incorporar los datos y re-generar las tablas finales en etapas.


Un problema que tiene la arquitectura es que, en muchos casos, es necesario generar los mismos valores en la capa de tiempo real y en la capa batch. ¡Esto en el tiempo genera duplicación de esfuerzo! Consideramos que el centro de verdad de los datos está en la parte batch, entonces muchas veces los campos generados por la _speed layer_ también se recrean en la otra capa para que queden persistidos [^2]. Por otra parte, si tenemos una arquitectura basada en eventos y buscamos que el centro de verdad esté en la parte de _real time_, seguramente nos convenga más una [arquitectura Kappa](https://www.oreilly.com/radar/questioning-the-lambda-architecture/). Prácticamente no hay parte batch y todo se hace con aplicaciones de streams y flujos de procesamiento de tiempo real (Apache Flink, Kafka Streams, Spark Microbatching, etc). Se pueden ver varios recursos para esa arquitectura [acá](https://milinda.pathirage.org/kappa-architecture.com/).

[^2]: Además, es mucho más difícil reprocesar en streams que en batch. La idea de la arquitectura Medallion es que sea fácil hacer cambios en las transformaciones de los datos porque siempre se tienen los datos originales crudos, y la separación en etapas hace que sólo sea necesario recalcular desde el punto necesario.

Habiendo introducido las arquitecturas principales, pensemos ahora cómo embeber un modelo en ellas.

## Patrones

### Interfaz HTTP

El patrón de una interfaz de HTTP es el más común para situaciones de inferencia en tiempo real. Tiene sentido porque es fácil de levantar, es fácil de escalar y se apoya en un montón de bibliotecas existentes para el desarrollo web. La interfaz puede ser un microservicio con un solo _endpoint_ o puede estar embebida dentro de una interfaz más grande que sea RPC o REST.



{{< svg
  src="diagrams/api.svg"
  height="500"
  caption="Despliegue en formato de API HTTP." >}}


Además de la facilidad, en definitiva la llamada al modelo coincide perfectamente con el patrón _Request-Reply_ de HTTP. Los problemas con este patrón aparecen cuando la predicción involucra servicios de terceros o procesos grandes que exceden un tiempo razonable para una petición. En esos casos se puede implementar una especie de [_long polling_ ](https://www.pubnub.com/guides/long-polling/) o se puede implementar un patrón asíncrono distinto. En caso de aplicaciones interactivas donde las predicciones se pueden hacer proactivamente (ej., el servidor envía periódicamente actualizaciones de contenido y con recomendaciones), o donde se requiera mantener un estado, seguramente convennga una conexión de WebSockets.

Quiero hacer un mini párrafo para hablar sobre REST. Me pasa mucho que digo "API REST" en vez de decir "API" como si fueran sinónimos o como si fuera el único patrón. En realidad, una interfaz para la predicción de un modelo se puede implementar como RPC o como REST. REST no es exclusivo para manejo de operaciones como CRUD, mientras se cumplan las restricciones del estándar. Por otra parte, RPC no tiene un estándar fijo, pero sí hay estándares como gRPC (Google, 2016). Hay _flavors_ de RPC que no usan HTTP, gRPC sí usa HTTP/2. Sobre la diferencia con RPC, [Fielding escribió lo siguiente](https://ics.uci.edu/~fielding/pubs/dissertation/evaluation.htm):

> What distinguishes HTTP from RPC isn't the syntax. It isn't even the different characteristics gained from using a stream as a parameter, though that helps to explain why existing RPC mechanisms were not usable for the Web. What makes HTTP significantly different from RPC is that the requests are directed to resources using a generic interface with standard semantics that can be interpreted by intermediaries almost as well as by the machines that originate services. 

En el párrafo hay que interpretar a HTTP como el protocolo más los estándares que proponía en la tesis, porque luego evidentemente se puede implementar un RPC sobre HTTP. Ahora, a fines más prácticos, usando (g)RPC uno espera tener un contrato mucho más estable sobre lo que el servidor espera y envía al cliente. La firma de la función puede estar fijada en protobuf, por ejemplo. Por otra parte, está bueno porque definir el contrato evita problemas luego por no tener una interfaz acordada entre cliente y servidor. [Acá hay una tabla comparativa](https://learn.microsoft.com/en-us/aspnet/core/grpc/comparison?view=aspnetcore-9.0) entre APIs estilo gRPC y estilo REST-JSON. 

Por último, si armamos una API en HTTP con un endpoint que sea `/predict` que acepta y devuelve JSON, no es RPC, ¡[pero tampoco es REST](https://en.wikipedia.org/wiki/Richardson_Maturity_Model)! 

### Batch

La predicción en formato batch es más sencilla. A través de un orquestador (CRON, Airflow, Dagster, Prefect, etc.) se toman los datos nuevos, se hace la predicción y se publican en algún lado. El _trigger_ para la ejecución puede ser un intervalo de tiempo o una condición (que hayan llegado datos nuevos, que se hayan acumulado tantos datos, etc.).

{{< svg
  src="diagrams/batch.svg"
  height="500"
  caption="Despliegue en formato batch." >}}

El orquestador no necesariamente ejecuta el modelo en la máquina propia. Puede levantar un contenedor de docker o usar un _worker_ específico para la inferencia. Puede lanzar también la tarea en otro servicio como Databricks, haciendo una ejecución remota. Acá no importa tanto cómo se ejecuta el modelo sino que hay un orquestador indicando cuándo se hace la inferencia.

Este patrón sirve para todos los casos donde la predicción puede esperar a un proceso asincrónico de _lotes_. Por ejemplo, cuando llegan los recibos de marzo a principio de mes, se sabe que para el 5 de abril se tiene un reporte con las proyecciones de ganancias para el siguiente trimestre, y así. Los lotes pueden ser más pequeños incluso, de hasta 30 o 15 minutos. Necesitar intervalos más pequeños suelen ser indicador de que otro patrón puede ser más útil.


### Serverless

Aah, serverless. Donde sí hay servidores y los recibos de la nube pueden explotar. El beneficio del patrón es que no pagamos por los momentos donde la computadora no está haciendo nada, así que si tenemos una arquitectura basada en eventos y el servicio tiene variabilidad en la tasa de peticiones que permite aprovechar el costo, puede ser una opción interesante. El costo del servicio generalmente es por cantidad de memoria utilizada, cantidad de tiempo de cada ejecución y cantidad de peticiones totales.

{{< svg
  src="diagrams/serverless.svg"
  height="500"
  caption="Despliegue en patrón Lambda. Se ve como una API HTTP por afuera, pero internamente se utiliza una Lambda que sólo consume recursos (y costo) mientras es llamada." >}}

El escalado es muy sencillo y se ocupa el proveedor de cloud de todo el proceso. Es generalmente fácil de configurar el ambiente y las dependencias de la función. [Acá](https://aws.amazon.com/blogs/compute/deploying-machine-learning-models-with-serverless-templates/) hay un ejemplo de AWS de cómo configurar una para hacer la inferencia de un modelo.

Como se cobra por milisegundo, no está bueno usar la Lambda para orquestar algo, es decir, llamar a varios servicios y esperar la respuesta de cada uno. En esos casos conviene usar algo como AWS StepFunctions -- ver ejemplos [acá](https://aws.amazon.com/step-functions/use-cases/#Machine_Learning_Operations).

### Streaming 1: Kafka con API 

Si tenemos una arquitectura basada con eventos y tenemos flujos en Kafka, seguramente en algún momento busquemos enlazar esos mismos flujos con el modelo predictivo. Si tenemos una arquitectura Kappa donde todo se representa con colas de mensajes esto es muy probable.

Una forma sencilla es en la aplicación de Kafka hacer la llamada al servicio de HTTP para hacer la predicción:

{{< svg
  src="diagrams/kafka_with_api.svg"
  height="500"
  caption="Unión del patrón de API HTTP con una aplicación basada en streams." >}}

La aplicación que necesita la predicción toma como siempre sus entradas de una cola de mensajes, procesa y hace una llamada a la API, espera la respuesta, y al terminar su procesamiento guarda los resultados en otra cola de mensajes.

Esto permite combinar una solución estándar para el despliegue del modelo con el resto de la arquitectura. Puede escalar independientemente y si usamos una herramienta que tiene varias amenities como monitoreo automático, logging y demás, todo sigue funcionando bien y tenemos lo mejor de ambos mundos. Las desventajas obvias de esta solución son la latencia agregada por el viaje de red, la saturación de conexiones y necesitar agregar más lógica para la serialización de los mensajes al servicio (y el manejo de errores). 

Otra desventaja importante, y por suerte poco frecuente, es el problema de los _side-effects_ al reprocesar. Si Kafka reprocesa un mensaje y la llamada al servicio se había hecho, y si ese servicio hace algún registro de la acción, esa acción se puede hacer más de una vez y eso rompe la semántica _exactly-once_. 

Para estos tres enfoques de _streaming_ aproveché el post de [Kai Waehner](https://www.kai-waehner.de/blog/2020/10/27/streaming-machine-learning-kafka-native-model-server-deployment-rpc-embedded-streams/). Para este caso puntual también hay un [repo](https://github.com/kaiwaehner/tensorflow-serving-java-grpc-kafka-streams) con un ejemplo.

### Streaming 2: Kafka con modelo embebido

Una versión más directa es tener el modelo embebido directamente en la aplicación que procesa los mensajes de Kafka. Esto negocia mejor latencia y mayor control sobre los _side-effects_ a cambio de tener que hacer a mano el logging del modelo, actualización, trazas de llamadas y demás.

{{< svg
  src="diagrams/kafka_embedded.svg"
  height="500"
  caption="Despliegue del modelo predictivo directamente en la aplicación de Kafka." >}}

La solución se ve más sencilla a nivel arquitectural y pasa la complejidad a la implementación. En general yo prefiero evitar esto como un primer acercamiento, optando por desacoplar el despliegue del modelo a una herramienta común, a menos que sepamos que tenemos un requerimiento fuerte de latencia. 

### Streaming 3: completamente asincrónico

Un nivel más en la asincronización del proceso es tener otra cola para el modelo. Así, las observaciones a predecir van en una cola, y el que necesite consumirlas puede escuchar la cola de salida. Es similar al caso de Streaming 2, pero la lógica que corre en la aplicación que procesa los streams es sólo la del modelo. Si queremos seguir haciendo otras operaciones, es posible que haya que hacer un join de streams[^3]. 

[^3]: Para leer sobre join de streams, el capítulo 11 del libro _Designing data-intensive applications_ de Kleppman es muy buena referencia.


La idea de esto no es armar nuestra aplicación que tenga el modelo mismo (eso sería el caso de Streaming 2 anterior), sino aprovechar alguna herramienta de serving de modelos que opere nativamente con colas de Kafka. De esta manera mantenemos los beneficios de que una herramienta OOTB[^4] maneje el monitoreo, gestión de modelos, logging, AB testing y otras amenities.

[^4]: [Out of the box](https://www.wearedevelopers.com/dictionary/ootb).

Algunas herramientas que permiten configurar a Kafka como entrada para predicción son [MLServer](https://mlserver.readthedocs.io/en/stable/examples/kafka/README.html), [Seldon](https://docs.seldon.io/projects/seldon-core/en/latest/streaming/kafka.html) y [KServe](https://kserve.github.io/website/master/modelserving/kafka/kafka/). 



### Microbatching y casi-tiempo-real

Una opción a mitad de camino entre procesar los pedidos uno-a-uno y procesar lotes periódicamente es usar _microbatching_. La idea es armar un lote de observaciones hasta que se acumulen cierta cantidad o hasta que se cumpla cierto tiempo límite, lo que pase primero.

{{< svg
  src="diagrams/microbatching.svg"
  height="500"
  caption="Despliegue con patrón de microbatch. Las observaciones se acumulan en pequeños grupos y se pasa a un job de inferencia sobre lotes, igual al caso de inferencia batch. Se busca una latencia de menos de un segundo." >}}

Lo seductor de este patrón es que pagando el costo de un poco más de latencia comparado con una solución nativa de _streaming_, la implementación se vuelve mucho más sencilla y prácticamente igual que programando por lotes.

Últimamente _microbatching_ es sinónimo de usar Spark para esto. La [documentación](https://spark.apache.org/docs/latest/structured-streaming-programming-guide.html#overview) es bastante buena y tiene esto para decir sobre las garantías que ofrece:

> Internally, by default, Structured Streaming queries are processed using a micro-batch processing engine, which processes data streams as a series of small batch jobs thereby achieving end-to-end latencies as low as 100 milliseconds and exactly-once fault-tolerance guarantees.   
> However, since Spark 2.3, we have introduced a new low-latency processing mode called Continuous Processing, which can achieve end-to-end latencies as low as 1 millisecond with at-least-once guarantees.   
> Without changing the Dataset/DataFrame operations in your queries, you will be able to choose the mode based on your application requirements.


### Edge 

La idea de deployar el modelo en el cliente mismo que necesita la predicción la mencionamos como _Edge computing_. Es una gran navaja en términos de escalado pero puede ser un dolor de cabeza en casos cuando los dispositivos finales son heterogéneos en capacidades de cómputo, ya que la predicción puede tardar mucho o saturar el dispositivo. 

El entrenamiento se sigue haciendo en la nube, pero la inferencia se puede hacer localmente. Nos ahorra el llamado a la red (que puede ser muy lento si estamos en casos de IOT o en regiones con mala conectividad) y puede dar mayores garantías de privacidad. En algunos casos es necesario enviar aún información de la observación eventualmente al servidor para aumentar el tamaño del conjunto de entrenamiento.

En [este post](https://blog.marvik.ai/2023/12/28/edge-computing-deploying-ai-models-into-multiple-edge-devices/) ahondan un poco en el tema y mencionan algunos servicios de AWS que pueden servir para implementarlo (como AWS IOT Greengrass). 

## Reglas de escalado

Para escalar los servicios en general las siguientes reglas me sirven:

{{< svg
  src="diagrams/scaling_laws.svg"
  height="500"
  caption="En APIs HTTP, es cuestión de agregar un load balancer con alguna lógica (round-robin, hash, etc.) y desplegar varios workers detrás. En aplicaciones de streams, es necesario implementar particiones y replicar los consumidores en un mismo grupo. En serverless es cuestión de poner la plata." >}}

## Otros links

- [Redis for Machine Learning](https://redis.io/resources/redis-machine-learning/): cómo deployar modelos _directamente en_ Redis.
- [FeatureStore.org](https://www.featurestore.org), una página con un montón de recursos sobre el patrón.
- Para gestión de modelos: [AIM](https://aimstack.io) y [MLFLow](https://mlflow.org).
- [DVC](https://dvc.org) para versionado de datos.
- Herramientas comunes para desplegar modelos: [BentoML](https://www.bentoml.com), [KServe](https://kserve.github.io/website/latest/), [TF Serving](https://github.com/tensorflow/serving), [MLServer](https://mlserver.readthedocs.io/en/stable/).
