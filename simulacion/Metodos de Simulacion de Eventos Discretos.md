```
---
title: Simulacion de eventos discretos
author: Marcelo Olivares
date: today
---
```

# Metodos de Simulacion de Eventos Discretos

**Autor: Prof. Marcelo Olivares**

*Ingenieria Industrial, U de Chile*

Los modelos analiticos son una herramienta muy útil para evaluar estrategias y apoyar la toma de decisiones en temas relacionados a planificacion de capacidad, dotación de personal y manejo de inventario en una organización. Estos modelos permiten entender los mecanismos y los diferentes *trade-off* que se deben considerar en la toma decisiones, que permiten cuantificar los efectos de primer orden para optimizar la asignación de recursos.

Sin embargo, la implementación práctica de estas decisiones muchas veces requiere incorporar mayor nivel de detalle del funcionamiento del sistema, que no siempre es posible de capturar a través de un modelo analítico. Para esto, los modelos de simulación son muy efectivos, ya que permiten incorporar detalles idiosincráticos de un sistema que pueden ser importantes. Dentro de la familia de métodos de simulación, identificamos tres grandes categorías:

| Tipo                               | Description                                                  | Ejemplos                                                     |
| ---------------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 1. Simulación fisica               | Replica un proceso real en un laboratorio                    | - Operacion de proceso de votacion bajo medidas sanitarias.[Paper plebiscito 2020](https://papers.ssrn.com/sol3/papers.cfm?abstract_id=3734699) |
| 2. Simulación de Montecarlo        | Permite medir algun resultado especifico de interés de un proceso estocástico. | - Utilidad esperada del modelo Newsvendor. [capsula de video](https://youtu.be/E19swFOZTdE) |
| 3. Simulación de eventos discretos | Modela el flujo de eventos detallado de un proceso estocastico. | - Flujo de clientes en los cajeros de un supermercado (ver ejemplo adjunto). |

En este documento, nos focalizamos en la tercera categoría, identificando sus principales elementos. Tambien discutimos aspectos mas generales de simulación, incluyendo el análisis estadístico de los resultados que estas simulaciones entregan.

## Cuando usar simulación de eventos discretos vs. modelos analiticos?

Una herramienta frecuentemente utilizada para analizar distintas métricas de desempeño de un proceso son los modelos de colas y otros modelos analíticos. Estos modelos tienen la ventaja de entregarnos resultados de forma rápida y parametrizable, que permiten evaluar de forma eficiente distintas configuraciones del sistema. Sin embargo, para modelos mas complejos, resulta difícil (o a veces infactible) capturar detalles importantes de un sistema mediante este enfoque.

Tomemos como ejemplo un sistema de cajas en un supermercado. Un sistema de este tipo podria modelarse con los [modelos de teoría de colas](https://youtu.be/YhNDadsScnA). Una alternativa es el modelo Erlang (conocido también como M/M/s), que se basa en los siguientes supuestos sobre el proceso:

- Las llegadas al sistema sigue un proceso de Poisson homogéneo (tasa constante).
- Los tiempos de servicio siguen una distribucion exponencial.
- Las llegadas son atendidas en orden de llegada.
- Todos las llegadas son procesadas: nadie abandona el sistema.

Bajo estos supuestos, este modelo permite derivar analiticamente la distribución de probabilidad del número de unidades en el sistema (esperando para iniciar el servicio y en servicio), cuando el sistema lleva operando un tiempo suficiente y se encuentra en estado estacionario.

Relajar estos supuestos vuelve más complejo el análisis. Pero quizas la mayor limitación de los modelos de colas es que se orientan a analizar el sistema en estado estacionario y no son adecuados para estudiar efectos transcientes.

Consideremos por ejemplo una caja de un supermercado.  Para efectos de la simulacion, consideramos una tiempo promedio de 40 s para procesar un cliente. Típicamente, las llegadas a un supermercado en días de semana siguen un patrón sinusoidal, con peaks a la hora de almuerzo y en lo horarios despues del trabajo. Consideramos 4 bloques de 4 horas cada uno:

| Bloque (segs) | Tiempo promedio entre llegadas ($a$) |
| ------------- | ------------------------------------ |
| 0-14.4K       | 100 segs                             |
| 14.4K - 28.8K | 50 segs                              |
| 28.8K-43.2K   | 75 segs                              |
| 43.2K-57.6K   | 42 segs                              |

La siguiente figura muestra la evolución del numero de personas en fila de la caja para un dia simulado. Se puede apreciar que los largos de fila son sustancialmente mayores en los horarios peak, como es esperable. 

![MM1_example](/Users/molivares/Dropbox (Personal)/teaching/simulation/MM1_example.png)

Nos interesa estudiar los tiempos de espera de los clientes. En un primer enfoque es utilizar modelos de colas, para lo cual es necesario separar el modelo en al menos cuatro periodos, para capturar las diferencias en las tasas de llegada.  Para cada bloque, se considera el tiempo promedio entre llegadas ($a$), el tiempo promedio de servicio ($p=40$ segundos) y se calcula la utlizacion $u=p/a$ para cada bloque. Luego se utiliza el modelo de colas para estimar el tiempo promedio de espera basado en la ecuación dada por el modelo Erlang:

$$ T_q = p\left(\frac{u}{1-u}\right)$$

El segundo enfoque -- mas preciso -- es generar  simulaciones como la que se describe en la figura y promediar los tiempos de espera de clientes que llegaron en cada bloque. La siguiente tabla compara las estimaciones de tiempos de espera para cada bloque usando estas metodologías.

| Bloque | a    | p    | Util | Tq (Erlang) | Tq (Simulation) |
| ------ | ---- | ---- | ---- | ----------- | --------------- |
| 0      | 100  | 40   | 0.40 | 26.7        | 16.8            |
| 1      | 50   | 40   | 0.80 | 160.0       | 115.4           |
| 2      | 75   | 40   | 0.53 | 45.7        | 60.7            |
| 3      | 42   | 40   | 0.95 | 800.0       | 173.6           |

Los modelos de colas no capturan correctamente el tiempo de espera.  Vemos, por ejemplo, que en el ultimo bloque el modelo sobreestima dramaticamente el tiempo de espera. Esto sucede porque el modelo Erlang predice lo que sucede cuando el sistema está estado estacionario, algo que no ocurre en este caso dado los cambios rápidos en el patrón de llegadas: el sistema "no alcanza" a llegar a estado estacionario. En el bloque 2, en cambio, el modelo Erlang subestima el tiempo de espera, esto debido a que este bloque arrastra clientes del peak alcanzado en el bloque anterior. En resumen, sistemas que no tienen una tasa homogenea de llegada hace que el analisis en estado estacionario sea menos util para predecir lo que sucede en la práctica.

## Construyendo un simulador de eventos discretos

La implementación de un modelo de simulacion de eventos discretos considera lo siguientes elementos:

- Variables de estado de system (System State, SS): Conjunto de variables que describen el sistema en un punto dado de tiempo.
- Reloj de simulación (t): Variable que lleva cuenta del tiempo interno en que ha avanzando la simulación.
- Lista de eventos (Event List, EL): Lista de eventos que esta por ocurrir, con el tiempo agendado para su ocurrencia.
- Estadísticos: Variables que llevan cuenta de algunos eventos acumulados, como por ejemplo, el número de llegadas que han ocurrido.
- Salida (output) de la simulación: Informacion relevante que se va guardando para realizar análisis posterior. En el ejemplo anterior, esto puede incluir el tiempo de llegada, salida y tiempo de espera de cada cliente que llego al sistema de cajas antes descrito, que se utilizó para calcular los tiempos de espera promedio en cada bloque.

A modo de ejemplo, consideremos una cola con un unico servidor, donde los tiempos entre llegadas y el tiempo de servicio tienen alguna distribución conocida (no necesariamente exponencial). Este modelo es conocido como G/G/1 (el termino G indica una distribución general para llegada y tiempo de proceso). Definimos las siguiente variables:

- Reloj de simulacion : $t$
- Estadisticos : $N_a$ = numero de llegadas ; $N_d$= numero de salidas.
- Variable de estado (SS) : $n$ =numero de clientes en el sistema.
- Lista de eventos: se consideran dos tipos de eventos. (1) el tiempo de la proxima llegada ($t_a$) y (2) el tiempo de la proximo termino de proceso, esto es, una salida del sistema ($t_d$).

El siguiente algoritmo describe la simulación:

**Inicialización**

- $t=Na=Nd=0$ , no hay llegadas ni salidas al comienzo
- SS = 0 (no hay clientes en el sistema)
- Generamos una variable aleatoria $T_0$ que representa el tiempo de la primera llegada, la cual se agenda como $t_a=T_0$. Como no hay unidades en el sistema, $t_d=\infty$ .

En lo que resta, definimos $Y$ como una variable aleatoria que describe el tiempo de servicio, i.i.d. para todas las llegadas. Definimos la v.a. $T_t$ para describir el tiempo entre llegadas, cuya distribución podria variar durante el tiempo. Finalmente, definimos  $E$ como el tiempo de término de la simulacion (no hay llegadas despues de $E$). El sistema va iterando entre los siguientes casos hasta que se alcanza el fin de la simulación.

**Caso 1:** Proximo evento es una llegada al sistema. $t_a \leq t_d$, $t_a<E$ . 

- Avanzar el reloj al siguiente evento: $t=t_a$
- Actualizar el estadistico de llegadas: $N_a = N_a +1$
- Actualizar estado del sistema SS: $n = n + 1$
- Generar próxima llegada: $t_a = t + T_t$
- Si $n=1$: el sistema estaba vacío antes de esta llegada, por lo que se debe agendar la proxima salida del recien llegado: $t_d=t + Y$ .

**Caso 2:** Proximo evento es una salida. $t_d<t_a$, $t_d<E$ .

- Avanzar el reloj: $t=t_d$
- Actualizar el estadistico de salidas: $N_d = N_d + 1$ .
- Reducir estado del sistema SS: $n=n-1$.
- Si $n>0$, agendar la proxima salida del servicio que acaba de comenzar. $t_d = t + Y$ . Si $n=0$, no hay clientes ni proxima salida, $t_d=\infty$ .

**Caso 3** : Se alcanzó el fin de la simulación, $\min (t_a,t_d)>E$,

- Aun quedan unidades por procesar, $n>0$.
  - Avanzar reloj: $t=t_d$.
  - Reducir estado SS: $n =n-1$
  - Actualizar salidas: $N_d= N_d +1$
  - Si $n>0$, agendar proxima salida $t_d = t +Y$.
- Sistema se vació: $n=0$. FIN.

### Implementando un modelo de simulación

Los algoritmos antes descritos pueden ser implementados directamente en un lenguaje de programación, pero existen otras alternativas para acelerar la implementacion de estos modelos. Por ejemplo, hay software que combinan simulación con un despliegue gráfico que facilita la implementacion en base a modulos y permite "visualizar" los estados y procesos dentro de la simulacion. Algunas de las herramientas graficas son [ProModel](https://www.promodel.com/) y [Arena](https://www.arenasimulation.com/) .

Otra alternativa es usar una API para distintos lenguajes de programación. Algunas recomendadas son [Simpy](https://simpy.readthedocs.io/en/latest/) que funciona en [Python](https://www.python.org/) (y cubrimos en este curso) y [Simulilnk](https://www.mathworks.com/products/simulink.html) para Matlab, pero existen muchos otros.



## Analisis estadistico de los resultados de una simulación

La simulación nos entrega una muestra de valores de algun indicador del sistema que nos interesa evaluar. Dado que el sistema esta sujeto a variabilidad, este indicador varía de una simulación a otra. En general, nos interesa analizar la distribución de este indicador o alguna propiedad, como por ejemplo su valor esperado.

Consideremos el ejemplo de la cola con un servidor único, similar al que vimos anteriormente. Nos interesa estudiar el tiempo en que sale el n-esimo cliente del sistema. Esto se puede calcular de la siguiente manera:

- Generar un vector de tiempos de llegadas, $\mathbf t_a= (t_a^1,\ldots,t_a^n)$
- Generar un vector de tiempos de servicio, $\mathbf Y =(Y_1,\ldots Y_n)$
- Correr la simulación y obtener el tiempo de salida del ultimo cliente. Esto es esencialmente una función $H(\mathbf t_a,\mathbf Y)$, donde la función misma esta construida mediante el simulador 
- Guardar el valor de esta primera corrida $h_1=H(\mathbf t_a,\mathbf Y)$.
- Repetir para corridas $r=1\ldots R$ guardando $h_r$  en cada corrida

El vector $(h_1\ldots h_R)$ representa una muestra de los tiempos de salida del n-esimo cliente. Usando el teorema central del limite, tenemos que:

$$ \bar h - E(h_r) \sim N(0,S/\sqrt{R}), $$

donde $\bar h$ es el promedio de los $h_r$ y $S$ su desviación estándar. Con esto, podemos construir un intervalo de confianza dado por $$ \bar h \pm z_\alpha S/\sqrt{R}$$ , donde $z_\alpha$ es el percentil de la distribucion normal estandar correspondiente al nivel de confianza del intervalo que se busca lograr. Por ejemplo, para un intervalo de confianza 95% de confianza usamos $z_\alpha = 1.96$.

Este resultado se puede utilizar para determinar cuantas corridas simular. Para esto, calculamos el número de simulaciones necesarias para obtener una precisión tal que el intervalo de confianza tenga un ancho $2d$, lo cual implica:

$$ R \geq (z_\alpha S/d)^2$$

Para implementar esta regla:

- Se corren un número limitado de corridas inicial  (e.g. 100) para obtener una estimación de la desviación estandar $S$.
- Se calcula el minimo numero de simulaciones con la ecuacion antes descrita ($R$) y se corren esas simulaciones. 
- Se vuelve a calcular $S$ para verificar que se logró la precision deseada del intervalo de confianza. Sino se logra, se vuelva a calcular $R$ mínimo y se repite el proceso.

## Conclusiones

La simulacion de eventos discretos es una herramienta util para analizar procesos complejos que resultan dificil de modelar analiticamente. Estas permiten analizar sistemas con mayor detalle que lo que se puede lograr con los modelos de colas, pero tienen la desventaja que requieren mas trabajo para desarrollarlos y puede requerir mucho tiempo computacional correr simulaciones suficientes cuando se necesita evaluar muchos escenarios.











