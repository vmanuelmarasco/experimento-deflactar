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

    cd ~/buckets/b1/dmeyf2026
    nohup bash -c 'for n in 3_deflactado_ipc 4_grupos; do jupyter nbconvert --to notebook --execute --inplace --ExecutePreprocessor.timeout=-1 $n.ipynb; done' > ~/corrida.log 2>&1 &
    tail -f ~/corrida.log

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

Las 5 corridas del baseline dieron una media de $289,9M con un desvío de $4,6M
(rango 285,0 a 296,3). Esa es la banda del azar: una diferencia menor a ~5M
entre modelos no se distingue del ruido.

Para el test de Wilcoxon pareado hacen falta al menos 6 pares, así que con 5
semillas la función avisa y no lo calcula. Conviene repetir con 20 o más antes
de afirmar que un modelo le gana a otro.

---

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

Las 5 corridas del baseline dieron una media de $289,9M con un desvío de $4,6M
(rango 285,0 a 296,3). Esa es la banda del azar: una diferencia menor a ~5M
entre modelos no se distingue del ruido.

Para el test de Wilcoxon pareado hacen falta al menos 6 pares, así que con 5
semillas la función avisa y no lo calcula. Conviene repetir con 20 o más antes
de afirmar que un modelo le gana a otro.

---

## Resultados

Los tres brazos corrieron completos con las mismas cinco semillas.

| Brazo | Ganancia media | Desvío | Rango de envíos | vs nominal |
|---|---|---|---|---|
| nominal (control) | $289,9M | $4,65M | 3.075 – 4.945 | — |
| IPC único | $286,4M | $1,14M | 3.831 – 5.569 | −$3,45M (−1,19%) |
| por grupos | $284,9M | $2,14M | 3.982 – 5.626 | −$4,98M (−1,72%) |

**La hipótesis no se confirma.** Deflactar no mejoró la ganancia, y hacerlo por
grupos tampoco mejoró sobre el IPC único: el orden quedó al revés del esperado.
Por el criterio fijado de antemano, el nominal le gana al brazo por grupos
(diferencia por encima de la banda de ruido y 4 de 5 semillas), mientras que
contra el IPC único es empate técnico: gana en las 5 pero por menos que el ruido.

**Lo inesperado: deflactar estabiliza.** El desvío entre semillas cae de $4,65M
a $2,14M y $1,14M. Los brazos deflactados ganan menos pero son mucho más
consistentes, y mandan más clientes. No era lo que se buscaba medir y con cinco
semillas no alcanza para afirmarlo, pero es la observación más sugerente.