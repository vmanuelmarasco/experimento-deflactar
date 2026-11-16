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

**Repeticiones.** Cada brazo se corre con las mismas cinco semillas (100129,
100183, 100151, 500041 y 900007). La comparación es pareada: la misma semilla en
dos brazos comparte el azar del bagging, de modo que la diferencia que queda es
atribuible a la deflación. Sobre esas diferencias se aplica un test de Wilcoxon
pareado.

**Criterio de decisión, fijado antes de mirar los resultados.** La métrica
primaria es la ganancia en 201910; el AUC en validación es secundario y sirve
para interpretar, no para decidir. Las cinco corridas del brazo nominal dieron
una media de $289,9 millones con un desvío de $4,6 millones, y esa es la banda
de ruido de referencia. Se considerará que un brazo supera a otro si la
diferencia de medias excede ese desvío y además gana en al menos cuatro de las
cinco semillas. Una diferencia menor se declarará empate, aunque el signo
favorezca a la hipótesis.

---

## Resultados

Los tres brazos corrieron completos con las mismas cinco semillas. La métrica
es el máximo de la curva de ganancia acumulada en 201910.

| Brazo | Ganancia media | Desvío | Rango de envíos | Diferencia vs nominal |
|---|---|---|---|---|
| Nominal (control) | $289,9M | $4,65M | 3.075 – 4.945 | — |
| IPC único | $286,4M | $1,14M | 3.831 – 5.569 | −$3,45M (−1,19%) |
| Por grupos | $284,9M | $2,14M | 3.982 – 5.626 | −$4,98M (−1,72%) |

Semilla por semilla:

| Semilla | Nominal | IPC único | Por grupos |
|---|---|---|---|
| 100129 | $296,3M | $286,2M | $282,9M |
| 100183 | $285,0M | $284,7M | $287,9M |
| 100151 | $288,0M | $287,8M | $283,1M |
| 500041 | $287,0M | $286,9M | $286,3M |
| 900007 | $293,0M | $286,4M | $284,3M |

**La hipótesis no se confirma.** Deflactar no mejoró la ganancia, y deflactar
con un índice por grupo tampoco mejoró sobre el IPC único. El orden quedó
exactamente al revés del esperado.

Aplicando el criterio fijado antes de mirar los resultados:

- **Nominal contra por grupos.** La diferencia de medias ($4,98M) supera la
  banda de ruido ($4,65M) y el nominal gana en cuatro de las cinco semillas.
  Se declara que el nominal es mejor.
- **Nominal contra IPC único.** El nominal gana en las cinco semillas, pero la
  diferencia ($3,45M) queda por debajo de la banda. Por el criterio
  preestablecido es un empate, aunque el signo sea sistemático.
- **IPC único contra por grupos.** $1,53M de diferencia, bien dentro del ruido.
  Empate.

**Hallazgo lateral: deflactar reduce la varianza.** El desvío entre semillas cae
de $4,65M en el nominal a $2,14M en el brazo por grupos y $1,14M con IPC único.
Los brazos deflactados ganan menos pero son notablemente más estables, y además
corren el óptimo de envíos hacia arriba (el nominal corta entre 3.075 y 4.945
clientes; los deflactados, entre 3.831 y 5.626). No era lo que el experimento
se propuso medir y con cinco semillas no alcanza para afirmarlo, pero es la
observación más sugerente de la corrida y la candidata más clara a una segunda
vuelta.

**Interpretación.** La explicación más plausible de por qué deflactar pierde es
que el nivel nominal le sirve al modelo como marcador implícito del mes: dentro
de un mes todos los montos están inflados por el mismo factor, de modo que el
nivel absoluto informa en qué parte del año cae la fila. Deflactar borra esa
señal y no la reemplaza por una mejor. Es coherente con que los dos brazos
deflactados pierdan y con que el que más uniformiza sea también el que más
achica la varianza: se está quitando información, no agregándola.

---

## Limitaciones de lo que se probó

**Un solo mes de test.** Toda la comparación descansa en 201910. La meseta de la
curva de ganancia es inestable: entre las cinco semillas del baseline, el
óptimo de envíos fue de 3.075 a 4.945, una variación del 60% para ganancias que
difieren en menos del 4%. Medir en un único mes hace que parte de lo que
observemos sea la particularidad de ese mes.

**Cinco semillas.** El test de Wilcoxon necesita al menos seis pares para dar un
p interpretable, así que con cinco solo podemos mirar medias, desvíos y en
cuántas semillas gana cada brazo. Para afirmar significancia habría que repetir
con veinte o más, que es lo que recomendaron los tutores.

**Un solo régimen inflacionario.** 2019 cerró con el IPC multiplicándose por 1,5
en doce meses. No sabemos si el efecto de deflactar se mantiene en un año de
inflación baja, donde la corrección sería pequeña, ni en uno de inflación mucho
más alta. La conclusión vale para este período.

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

**El uso del índice del mes corriente.** Para medir en 201910 se usó el IPC de
201910, que en una aplicación real todavía no estaría publicado al momento de
scorear. La verificación de que usar el índice del mes anterior no altera los
resultados quedó pendiente y es el paso inmediato antes de llevar esto al
pipeline productivo.

**La tasa de plazo fijo como deflactor de saldos es la decisión más débil.** Un
saldo en cuenta corriente no devenga esa tasa; un plazo fijo sí. Agruparlos bajo
el mismo índice es una simplificación, y convendría reportar también la versión
con IPC para esas ocho columnas.

**La interpretación del resultado negativo no se testeó.** Que el nivel nominal
funcione como marcador del mes es la explicación más razonable de por qué
deflactar pierde, pero es una conjetura. Contrastarla pide un brazo adicional:
deflactar y agregar al mismo tiempo una variable explícita de tiempo, o al
revés, dejar los montos nominales y quitarle al modelo toda referencia al mes.
Mientras eso no se corra, lo que está medido es el efecto, no su causa. Por eso es testeable con un brazo más: deflactado + foto_mes como feature. Si al devolverle la referencia temporal la ganancia vuelve al nivel del nominal, la explicación queda confirmada. Si no vuelve, era otra cosa y tenés que buscar.