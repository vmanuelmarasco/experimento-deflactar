## Diseño experimental

**Datos.** Dataset de la segunda competencia, meses 201901 a 201912: 1.563.565
filas y 154 columnas. La `clase_ternaria` se construye con el calendario
completo de cliente × mes, de modo que las bajas se detectan por ausencia en
los meses siguientes y no por un campo del banco. Los dos últimos meses quedan
sin clase por construcción: 201911 solo puede tener BAJA+1 y 201912 no puede
tener clase, por lo que el dato etiquetado llega hasta 201910.

**Partición.** Es la que fija el baseline de la cátedra. Entrenamiento en
201903, 201904, 201905 y 201906, completos y sin sampling. Validación en
201908, que ordena la búsqueda de hiperparámetros por AUC. Test en 201910, que
no se usa para nada más que la medición final. Para el test, el modelo se
reentrena incorporando 201907 y 201908.

**Target.** En entrenamiento, BAJA+1 y BAJA+2 combinadas en una clase positiva.
La ganancia, en cambio, solo reconoce BAJA+2: $1.072.500 por cada BAJA+2
estimulado y $27.500 de costo por cada estímulo enviado a quien no lo era.

**Variables.** Sobre cada una de las 152 columnas se calculan lag 1, lag 2,
delta 1 y delta 2 por cliente, lo que lleva el dataset a 760 features. En los
brazos deflactados, la deflación se aplica **antes** de calcular las
históricas, de manera que los lags y deltas se construyan sobre valores reales:
un delta sobre montos nominales mezclaría el cambio real del cliente con la
inflación del período.

**La intervención.** Tres brazos que difieren únicamente en cómo se tratan las
73 columnas monetarias:

1. **Nominal.** Sin transformación. Es el baseline de la cátedra sin modificar y
   funciona como control.
2. **IPC único.** Las 73 columnas divididas por el IPC del mes y llevadas a
   precios de enero 2019.
3. **Por grupos.** Las 73 columnas repartidas en siete grupos, cada uno con el
   deflactor que le corresponde: IPC para consumos, RIPTE para sueldos, tasa de
   plazo fijo capitalizada para saldos, valor modal del mes para comisiones,
   tipo de cambio para montos en dólares y tasa activa capitalizada para
   crédito. Los cuatro límites de tarjeta no se deflactan: se agregan en su
   lugar cuatro ratios de uso (consumo sobre límite de compra y saldo sobre
   límite de financiación, para Master y Visa).

Todos los índices provienen de fuentes oficiales por la API de series de tiempo
de datos.gob.ar, y el mes de referencia es enero 2019 en los tres casos.

**Hiperparámetros.** Cada brazo tiene su propia optimización bayesiana, con
Optuna y sampler TPE, 50 trials, sobre los cinco hiperparámetros que indica la
consigna: `num_leaves` (8 a 256), `learning_rate` (0,01 a 0,3 en escala
logarítmica), `min_data_in_leaf` (100 a 8.000 en escala logarítmica),
`feature_fraction` (0,2 a 1) y `bagging_fraction` (0,2 a 1), con `bagging_freq`
fijo en 1 y el resto de los parámetros en sus valores por defecto. El criterio
de la búsqueda es el AUC en 201908, con early stopping a 100 rondas sin mejora
y un techo de 2.000 rondas.

Se optimiza por separado en cada brazo porque los datos deflactados son otro
conjunto de datos y los hiperparámetros que le quedan bien son otros. Imponerle
al experimento la configuración ajustada para el nominal compararía el mejor
baseline posible contra una versión subóptima del experimento, y el resultado
hablaría más de los hiperparámetros que de la deflación.

**Modelo final y medición.** Con los mejores hiperparámetros y la cantidad de
rondas que la búsqueda encontró, se reentrena sobre 201903–201908 y se predice
201910. Los clientes se ordenan por probabilidad descendente y se acumula la
ganancia; se reporta el máximo de esa curva (la meseta) y la cantidad de envíos
en la que se alcanza.

**Repeticiones.** Cada brazo se corre con las mismas veinte semillas. Las
primeras cinco (100129, 100183, 100151, 500041 y 900007) son las de la primera
vuelta; las otras quince (100193, 100207, 100213, 100237, 100267, 100271,
100279, 100291, 100297, 100313, 500069, 500083, 500107, 500111 y 900017) se
agregaron en una segunda vuelta, reutilizando los hiperparámetros ya
optimizados, de modo que ampliar la muestra no implicó rehacer ninguna búsqueda.
La comparación es pareada: la misma semilla en dos brazos comparte el azar del
bagging, de modo que la diferencia que queda es atribuible a la deflación. Sobre
esas diferencias se aplica un test de Wilcoxon pareado, que con veinte pares sí
es interpretable: con cinco, el p mínimo alcanzable de dos colas es 0,0625, así
que ningún resultado podía resultar significativo por fuerte que fuera el efecto.

**Criterio de decisión, fijado antes de mirar los resultados.** La métrica
primaria es la ganancia en 201910; el AUC en validación es secundario y sirve
para interpretar, no para decidir. Las cinco corridas del brazo nominal dieron
una media de $289,9 millones con un desvío de $4,6 millones, y esa es la banda
de ruido de referencia. Se considerará que un brazo supera a otro si la
diferencia de medias excede ese desvío y además gana en al menos cuatro de las
cinco semillas. Una diferencia menor se declarará empate, aunque el signo
favorezca a la hipótesis.

Al ampliar a veinte semillas el criterio se mantuvo con la misma forma, solo
recalculado sobre la muestra nueva: la banda de ruido pasó a ser el desvío del
nominal con veinte corridas ($3,41 millones) y la mayoría exigida pasó de cuatro
de cinco a la mayoría de veinte. No se cambió la regla después de ver los
números; se cambió el tamaño de la muestra sobre la que se aplica, y se agregó
el p del Wilcoxon, que con cinco semillas no se podía calcular.

---

## Resultados

Los tres brazos corrieron completos con las mismas veinte semillas. La métrica
es el máximo de la curva de ganancia acumulada en 201910.

| Brazo | Ganancia media | Desvío | Rango de envíos | Diferencia vs nominal |
|---|---|---|---|---|
| Nominal (control) | $290,57M | $3,41M | 3.075 – 5.625 | — |
| IPC único | $287,32M | $2,09M | 3.831 – 5.834 | −$3,25M (−1,12%) |
| Por grupos | $284,86M | $1,89M | 2.896 – 6.285 | −$5,71M (−1,97%) |

Las veinte ganancias de cada brazo están en `baseline_20semillas.csv`,
`ipc_20semillas.csv` y `grupos_20semillas.csv`.

Comparaciones pareadas:

| Comparación | Diferencia de medias | Semillas a favor | Wilcoxon |
|---|---|---|---|
| Nominal vs IPC único | $3,25M (1,12%) | 19 de 20 | p = 0,0001 |
| Nominal vs por grupos | $5,71M (1,97%) | 19 de 20 | p < 0,0001 |
| IPC único vs por grupos | $2,46M (0,86%) | 15 de 20 | p = 0,0012 |

**La hipótesis queda refutada.** No solo no se confirma: el resultado es
significativo en la dirección contraria. Deflactar empeora la ganancia, y
deflactar con un índice por grupo empeora más que hacerlo con un IPC único. El
orden quedó exactamente al revés del esperado en las dos comparaciones que la
hipótesis predecía.

Aplicando el criterio fijado antes de mirar los resultados:

- **Nominal contra por grupos.** $5,71M de diferencia, por encima de la banda de
  ruido, y el nominal gana en diecinueve de veinte semillas. El nominal es mejor.
- **Nominal contra IPC único.** $3,25M de diferencia, también por encima de la
  banda, y gana en diecinueve de veinte. El nominal es mejor. Con cinco semillas
  esta comparación había quedado en empate técnico; al ampliar la muestra la
  banda de ruido se angostó de $4,65M a $3,41M y la diferencia la superó.
- **IPC único contra por grupos.** $2,46M de diferencia y quince de veinte
  semillas a favor del IPC único. Es la comparación más ajustada de las tres,
  pero el p la respalda.

Vale registrar que la primera vuelta, con cinco semillas, habría dejado dos de
estas tres comparaciones en empate. La conclusión no cambió de signo al
cuadruplicar la muestra —el nominal ganaba en las tres desde el principio—, pero
recién con veinte semillas hay con qué afirmarlo.

**Hallazgo lateral: deflactar reduce la varianza.** El desvío entre semillas cae
de $3,41M en el nominal a $2,09M con IPC único y $1,89M por grupos. Los brazos
deflactados ganan menos pero son notablemente más estables, y además corren el
óptimo de envíos hacia arriba. El patrón se sostuvo al pasar de cinco a veinte
semillas, que es lo que le da algo de peso: no era un artefacto de la muestra
chica. Sigue sin ser lo que el experimento se propuso medir, y queda como la
candidata más clara a una segunda vuelta.

**Interpretación.** La explicación más plausible de por qué deflactar pierde es
que el nivel nominal le sirve al modelo como marcador implícito del mes: dentro
de un mes todos los montos están inflados por el mismo factor, de modo que el
nivel absoluto informa en qué parte del año cae la fila. Deflactar borra esa
señal y no la reemplaza por una mejor. Es coherente con que los dos brazos
deflactados pierdan y con que el que más uniformiza sea también el que más
achica la varianza: se está quitando información, no agregándola.

**Un brazo adicional para testear esa interpretación.** Se repitió el brazo de
IPC único agregando `foto_mes` como feature explícita, para devolverle al modelo
la referencia temporal que la deflación le había quitado. Los hiperparámetros se
volvieron a optimizar, porque es otro conjunto de features. Si la conjetura
fuera correcta, la ganancia debería acercarse a la del nominal.

| Semilla | IPC único | IPC + mes |
|---|---|---|
| 100129 | $286,2M | $287,1M |
| 100183 | $284,7M | $286,7M |
| 100151 | $287,8M | $289,8M |
| 500041 | $286,9M | $287,8M |
| 900007 | $286,4M | $286,6M |
| **Media** | **$286,4M** | **$287,6M** |

Agregar el mes mejora en las cinco semillas, $1,18M (+0,41%). La dirección es la
que predice la conjetura, sin excepciones. Pero la magnitud recupera solo un
tercio de los $3,45M que el brazo había perdido contra el nominal, y queda
dentro de la banda de ruido. **Respaldo parcial:** la referencia temporal
explica parte del efecto, no todo. El desvío, además, se mantiene en $1,33M, de
modo que la estabilidad que trae deflactar es independiente de la cuestión
temporal.

**El mes de referencia no altera los resultados, y se demuestra sin correr nada.**
En la devolución del baseline se sugirió repetir el pipeline con distintos meses
base, por si enero de 2019 fuera un mes atípico. No hace falta: la fórmula es
`monto ÷ índice[mes] × índice[base]`, y `índice[base]` es una constante igual
para todas las filas de todos los meses. Cambiar la base multiplica el dataset
entero por un escalar, y los árboles, que parten sobre umbrales, son invariantes
a un reescalado uniforme: mismos cortes, misma ganancia.

La verificación numérica lo confirma. El cociente entre la versión con base
201901 y la de base 201906, sobre las 73 columnas monetarias y los doce meses,
va de 1,1894777757092052 a 1,1894777757092057, contra el 1,1894777757092054
esperado. La dispersión aparece en el decimosexto dígito: es ruido de punto
flotante, no una diferencia. Elegir la base es elegir la unidad en que se miran
los montos, como pasar de metros a centímetros.

**Objetivo específico 1: el índice del mes anterior.** En producción, el índice
del mes que se está scoreando todavía no está publicado. Se repitió el brazo de
IPC único deflactando cada mes con el índice del mes previo (y diciembre de
2018, 184,2552, para enero de 2019), reusando sin cambios los hiperparámetros
del brazo 3: si se volviera a optimizar, no se sabría si la diferencia viene
del índice o de la búsqueda.

| Semilla | Índice del mes | Índice del mes anterior |
|---|---|---|
| 100129 | $286,2M | $287,3M |
| 100183 | $284,7M | $286,5M |
| 100151 | $287,8M | $286,6M |
| 500041 | $286,9M | $287,9M |
| 900007 | $286,4M | $288,6M |
| **Media** | **$286,4M** | **$287,4M** |

La diferencia es de $973.500 (+0,34%), muy por debajo de la banda de ruido
($4,65M). El rezago gana en cuatro de las cinco semillas, pero por una magnitud
despreciable: son equivalentes. **El objetivo queda cumplido:** el resultado no
depende de información que no estaría disponible al momento de scorear, de modo
que el pipeline es aplicable en producción tal como está.

---

## Limitaciones de lo que se probó

**Un solo mes de test.** Toda la comparación descansa en 201910. La meseta de la
curva de ganancia es inestable: entre las cinco semillas del baseline, el
óptimo de envíos fue de 3.075 a 4.945, una variación del 60% para ganancias que
difieren en menos del 4%. Medir en un único mes hace que parte de lo que
observemos sea la particularidad de ese mes.

**Veinte semillas en los brazos principales, cinco en los auxiliares.** Los
tres brazos de la comparación central se corrieron con veinte semillas y el
Wilcoxon es interpretable. Los brazos 5 y 6 —`foto_mes` como feature e índice
rezagado— se quedaron en cinco, así que sobre ellos solo puede decirse la
dirección y el tamaño aproximado, no la significancia. Las conclusiones que
dependen de esos dos brazos son más débiles que las del cuerpo principal.

**Un solo año, y de inflación muy alta.** Todo el experimento vive dentro de
2019, un año que cerró con el IPC multiplicándose por 1,5 en doce meses. No se
probaron otros años: no sabemos si deflactar sigue perdiendo en un período de
inflación baja, donde la corrección sería casi nula y por lo tanto también lo
sería el daño, ni en uno de inflación mucho más alta, donde podría volverse
necesaria. La conclusión vale para este año y este nivel de inflación. (Esto no
es una limitación sobre la cantidad de índices usados: dentro de 2019 se usó el
valor del IPC mes a mes, no un único valor anual.)

**Índices nacionales aplicados a clientes individuales.** El IPC mide una
canasta promedio de hogares del país; el RIPTE, el salario promedio del sistema
previsional. Ningún cliente concreto enfrenta exactamente esa inflación: hay
diferencias regionales, de canasta y de sector. Estamos corrigiendo con un
promedio algo que en realidad es heterogéneo.

**El índice de comisiones es propio.** No existe una serie oficial de comisiones
bancarias, así que se construye desde los datos tomando el valor más frecuente
de `mcomisiones_mantenimiento` en cada mes. El método tiene respaldo —es lo que
el manual de IPC del FMI trata como scanner data— pero depende de una sola
columna y del supuesto de que la moda representa el precio de lista.

**La clasificación en grupos es un juicio, no un dato.** Se asignó cada columna
según su nombre y el diccionario de datos. Varias son discutibles:
`mrentabilidad` y los márgenes son resultado contable del banco más que precios
y quedaron con las comisiones; `mforex_buy` y `mforex_sell` se trataron como
dólares aunque el monto podría estar expresado en pesos. Un criterio distinto
daría otro brazo 3.

**El brazo por grupos no está perfectamente controlado.** Al reemplazar los
límites por ratios de uso, ese brazo tiene cuatro columnas más que los otros
dos, es decir veinte features adicionales una vez calculados lags y deltas. Si
resulta ganador, parte del mérito podría venir de los ratios y no de los
deflactores. Separar ambos efectos requeriría un cuarto brazo con los ratios
pero sin deflactar.

**El efecto solo puede aparecer entre meses, no dentro de un mes.** Dividir
todas las filas de un mes por una constante no cambia el orden de los clientes
en ese mes, y los árboles parten sobre ese orden. La deflación puede incidir
únicamente porque el entrenamiento mezcla varios meses y porque los lags y
deltas comparan meses distintos. Esto acota de entrada el tamaño del efecto que
cabe esperar.

**El chequeo del índice rezagado se hizo sobre un solo brazo.** La equivalencia
entre usar el índice del mes y el del mes anterior se verificó en el brazo de
IPC único y con los hiperparámetros de ese brazo. Es razonable esperar que valga
también para el brazo por grupos, donde conviven siete series, pero no se midió.

**La tasa de plazo fijo como deflactor de saldos es la decisión más débil.** Un
saldo en cuenta corriente no devenga esa tasa; un plazo fijo sí. Agruparlos bajo
el mismo índice es una simplificación, y convendría reportar también la versión
con IPC para esas ocho columnas.

**La interpretación quedó respaldada solo a medias.** El brazo con `foto_mes`
confirma la dirección en las cinco semillas, pero recupera apenas un tercio de
lo que la deflación había costado. Los otros dos tercios no tienen explicación
medida. Faltan dos cosas para cerrar el argumento: el brazo simétrico, con
montos nominales y sin ninguna referencia al mes, y una versión con `foto_mes`
tratada como categórica. Esto último importa porque el modelo final entrena en
201903–201908 y predice 201910, de modo que el mes de test queda fuera del rango
visto y el árbol termina tratando a octubre como si fuera agosto.
