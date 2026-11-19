# Deflactar por grupo de variables

Experimento colaborativo, comisión lunes, DMEyF 2026.

**Hipótesis:** usar un deflactor distinto para cada grupo de variables monetarias
da más ganancia que dividir todas por el IPC, y deflactar da más que no hacerlo.

**Qué se compara:** tres modelos idénticos salvo por la transformación de las
columnas monetarias.

| Modelo | Transformación | Notebook |
|---|---|---|
| nominal (control) | ninguna | `2_baseline.ipynb` |
| IPC único | todo ÷ IPC | `3_deflactado_ipc.ipynb` |
| por grupos | cada grupo con su deflactor | `4_grupos.ipynb` |

El modelo nominal es el baseline que especificó la cátedra en
`#Experimentos - Lunes > Baseline + Entregables`, sin ninguna modificación.

---

## Los cinco notebooks

| # | Notebook | Qué hace | Cuándo se corre |
|---|---|---|---|
| 0 | `0_datos.ipynb` | Del crudo al parquet: construye `clase_ternaria` con DuckDB y exporta `datos_2019.parquet` | Una sola vez |
| 1 | `1_home.ipynb` | La receta común: constantes, lags y deltas, la BO, la ganancia, los gráficos | No se corre sola; la traen las demás con `%run` |
| 2 | `2_baseline.ipynb` | Modelo nominal | Primero |
| 3 | `3_deflactado_ipc.ipynb` | Modelo con IPC único | Segundo |
| 4 | `4_grupos.ipynb` | Modelo por grupos **y la comparación final** | Último, porque lee los CSV de los otros dos |

El orden importa solo por el 4: los tres modelos son independientes, pero la
comparación necesita que los otros ya hayan dejado su CSV.

---

## Qué hace falta antes

- VM `desktop-jr` encendida (e2-highmem-8: 8 vCPU, 64 GB).
- `competencia_02_crudo.csv.gz` en `~/datasets/`. Si no está:
  `gcloud storage cp gs://manuel_marasco_dmeyf_buckito2026/datasets/competencia_02_crudo.csv.gz ~/datasets/`
- `datos_2019.parquet` en `~/datasets/`, que lo genera el notebook 0.

---

## Cómo correr

Interactivo, para mirar los gráficos: abrir cada notebook y Run All, en orden.

Desatendido, que es lo recomendable para los notebooks 3 y 4 porque tardan horas:

```bash
cd ~/buckets/b1/dmeyf2026
nohup bash -c 'for n in 3_deflactado_ipc 4_grupos; do jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=-1 $n.ipynb; done' > ~/corrida.log 2>&1 &
tail -f ~/corrida.log
```

Corren uno después del otro, no en paralelo: los 8 núcleos ya los usa LightGBM,
en paralelo cada uno va tres veces más lento y se duplica la memoria.

**Tiempos aproximados** en esta VM:

| Paso | Tiempo |
|---|---|
| Notebook 0, una vez | ~15 min |
| Lags y deltas, por notebook | ~3 min |
| Optimización bayesiana, 50 trials | ~2 a 3 h |
| Modelo final, 5 semillas | ~17 min |

---

## Qué deja cada corrida

En `~/buckets/b1/`, que es lo único que sobrevive si se borra la VM:

- `baseline_5semillas.csv`, `ipc_5semillas.csv`, `grupos_5semillas.csv`:
  una fila por semilla con ganancia y envíos.
- `baseline_20semillas.csv`, `ipc_20semillas.csv`, `grupos_20semillas.csv`:
  lo mismo con las 15 semillas extra agregadas. Son los que usa la comparación
  final.
- `optuna_baseline.db`, `optuna_ipc.db`, `optuna_grupos.db`: los estudios
  completos de Optuna, con todos los trials.

Los estudios son **idempotentes**: si el sqlite ya tiene sus 50 trials, la
notebook no vuelve a optimizar. Se puede hacer Run All sin repetir horas de
cómputo. Y si la VM es nueva, el notebook restaura el estudio desde el bucket
antes de decidir.

---

## Decisiones metodológicas

**Se deflacta antes de armar lags y deltas.** Un delta sobre montos nominales
mezcla el cambio real con la inflación. Deflactar después sería peor todavía:
una resta entre dos meses distintos no se corrige con un solo índice.

**Cada modelo tiene su propia optimización bayesiana.** Los datos deflactados
son otro dataset y los hiperparámetros que le quedan bien son otros. Imponerle
los del nominal compararía el mejor baseline contra una versión mal configurada
del experimento. Es lo que indicaron los tutores en Zulip: "siempre OP",
"un BO por cada set de datos".

**Las mismas 5 semillas en los tres modelos**, para que la comparación sea
pareada: la misma semilla comparte el azar entre modelos y lo que queda es el
efecto de deflactar.

**Los límites no se deflactan.** Un límite de compra queda congelado meses:
dividirlo por el IPC le inventa una caída que no existió. Se agregan en cambio
cuatro ratios de uso (consumo sobre límite y saldo sobre financiación, para
Master y Visa). Eso hace que el modelo por grupos tenga 20 features más que los
otros dos: es deliberado y hay que declararlo al comparar.

---

## Ruido de semilla

Los tres brazos se corrieron con **20 semillas**, no con 5. Las primeras cinco
(100129, 100183, 100151, 500041, 900007) son las de la primera vuelta; las otras
quince se agregaron después, usando los mismos hiperparámetros ya optimizados,
así que sumarlas no costó ninguna optimización nueva.

Veinte pares alcanzan para el test de Wilcoxon pareado, que con cinco no se
podía calcular: con n=5 el p mínimo alcanzable de dos colas es 0,0625, de modo
que ningún resultado podía dar significativo por más fuerte que fuera.

Con 20 semillas la banda de ruido del baseline es un desvío de $3,41M alrededor
de una media de $290,6M. Con 5 daba $4,65M alrededor de $289,9M: la media casi
no se movió, pero la incertidumbre sobre ella se redujo a la mitad.

---

## Resultados

Los tres brazos corrieron completos con las mismas 20 semillas.

| Brazo | Ganancia media | Desvío | Rango de envíos | vs nominal |
|---|---|---|---|---|
| nominal (control) | $290,57M | $3,41M | 3.075 – 5.625 | — |
| IPC único | $287,32M | $2,09M | 3.831 – 5.834 | −$3,25M (−1,12%) |
| por grupos | $284,86M | $1,89M | 2.896 – 6.285 | −$5,71M (−1,97%) |

Wilcoxon pareado sobre las 20 semillas:

| Comparación | Diferencia | Gana | p |
|---|---|---|---|
| nominal vs IPC único | $3,25M a favor del nominal | 19 de 20 | 0,0001 |
| nominal vs por grupos | $5,71M a favor del nominal | 19 de 20 | <0,0001 |
| IPC único vs por grupos | $2,46M a favor del IPC | 15 de 20 | 0,0012 |

**La hipótesis queda refutada.** No es que deflactar "no mejore": deflactar
empeora, y hacerlo por grupos empeora todavía más. El orden quedó exactamente
al revés del esperado y las tres diferencias son estadísticamente significativas.

Con cinco semillas dos de esas tres comparaciones eran empate técnico. Con
veinte, el criterio fijado de antemano —superar la banda de ruido y ganar en la
mayoría de las semillas— se cumple en las tres, y además el p lo respalda.

**Lo inesperado: deflactar estabiliza.** El desvío entre semillas cae de $3,41M
a $2,09M y $1,89M. Los brazos deflactados ganan menos pero son más consistentes,
y mandan más clientes. No era lo que se buscaba medir, pero es la observación
más sugerente y se sostuvo al cuadruplicar las semillas.

**El índice rezagado no cambia nada.** Repitiendo el brazo de IPC único con el
índice del mes anterior —el único publicado al momento de scorear— y los mismos
hiperparámetros, la media pasa de $286,4M a $287,4M: $973.500 de diferencia,
muy adentro del ruido. El pipeline es aplicable en producción tal como está.

**Devolverle el mes recupera un tercio.** Agregando `foto_mes` como feature al
brazo de IPC único, la media sube de $286,4M a $287,6M y mejora en las cinco
semillas. La dirección confirma que parte de lo que el modelo pierde al
deflactar es la referencia temporal, pero solo un tercio: los otros dos tercios
quedan sin explicación.

Los brazos 5 y 6 se corrieron con 5 semillas, no con 20: son chequeos de apoyo,
no parte de la comparación principal.

---

## Los cinco brazos

| Brazo | Notebook | Media | Desvío | Semillas |
|---|---|---|---|---|
| nominal | `2_baseline` | $290,57M | $3,41M | 20 |
| IPC único | `3_deflactado_ipc` | $287,32M | $2,09M | 20 |
| por grupos | `4_grupos` | $284,86M | $1,89M | 20 |
| IPC + mes | `5_deflactado_ipc_con_fotomes` | $287,6M | $1,33M | 5 |
| rezagado | `6_indice_rezagado` | $287,4M | $0,88M | 5 |

Los tres primeros notebooks (2, 3 y 4) son los que responden la hipótesis. El 5
y el 6 son chequeos de apoyo.

**El mes de referencia es irrelevante.** Cambiar la base del deflactor multiplica
todo el dataset por una constante, y los árboles son invariantes a eso. Verificado:
el cociente entre la base 201901 y la 201906 es constante hasta el decimoquinto
dígito. No hace falta correr el pipeline con varias bases.
