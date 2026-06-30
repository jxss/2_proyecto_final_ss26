# Proyecto final: Matching bipartito 4×4 con Ecobici CDMX — QUBO → QAOA local

QMéxico Summer School 2026 — Qubit.mx

## 1. Fuente de datos

**Dataset:** Afluencia desglosada acumulada del sistema Ecobici (Ciudad de México).

**Institución responsable:** Secretaría de Movilidad (SEMOVI), Gobierno de la Ciudad de México, a través del Portal de Datos Abiertos de la CDMX (Agencia Digital de Innovación Pública).

**Portal:** https://datos.cdmx.gob.mx/dataset/afluencia-diaria-del-sistema-ecobici

**Archivo descargado:** `afluencia_desglosada_acumulada_2024_07.csv`

**Fecha de consulta:** 29 de junio de 2026.

**Licencia / condiciones de uso:** datos publicados como datos abiertos por el Gobierno de la Ciudad de México; portal de uso público, sin restricción de uso para fines educativos.

**Cobertura temporal del archivo original:** abril de 2010 a julio de 2024, con columnas `anio, mes, fecha_corte, hora, genero, rango_edad, viajes` (115,773 filas). Los valores de `viajes` ya vienen agregados por hora, género y rango de edad — **no existe ninguna fila por persona individual**, por lo que el dataset cumple de origen el criterio de no usar datos personales identificables.

## 2. Limpieza aplicada

Antes de construir la instancia 4×4 se aplicaron tres correcciones, documentadas para reproducibilidad:

1. **Columna `hora`:** el valor original incluía el sufijo de texto `" hrs"` (p. ej. `"00:00:00 hrs"`). Se eliminó el sufijo y se extrajo la hora como entero 0–23.
2. **Columna `rango_edad`:** el mismo rango de edad aparece escrito con dos formatos distintos a lo largo del histórico (`"10 - 20 años"` con espacios en años tempranos, `"10-20 años"` sin espacios en años recientes). Sin esta corrección, pandas trataba el mismo grupo de edad como categorías distintas (16 categorías en vez de las 8 reales). Se normalizó eliminando los espacios alrededor del guion.
3. **Valores faltantes:** se encontraron 3,470 filas con `genero` vacío y 1,397 con `rango_edad` vacío (algunas comparten ambos). En total, **4,744 filas (4.10% del archivo original) se excluyeron** del análisis por no tener información demográfica completa, en vez de imputar un valor no verificable.

**Resultado:** `data/ecobici_julio2024_limpio.csv` — 11,480 filas correspondientes únicamente a julio de 2024 (mes completo, 31 días, sin huecos), con 1,825,593 viajes totales registrados ese mes.

**Nota de validación adicional:** la categoría de edad `"10-20 años"` podría parecer inconsistente con la política de Ecobici, pero se verificó (sitio oficial `ecobici.cdmx.gob.mx`) que la edad mínima real para registrarse es 16 años (con autorización de tutor entre 16 y 18 años). La etiqueta de la categoría es más amplia que el uso real permitido, pero no contradice los datos.

## 3. Definición de $A$, $B$, $x_{ij}$ y $S_{ij}$

**Conjunto $A$ — 4 franjas horarias de uso**, construidas dividiendo las 24 horas del día en bloques de 6 horas:

| id | Franja |
|---|---|
| A1 | Madrugada (00–05h) |
| A2 | Mañana (06–11h) |
| A3 | Tarde (12–17h) |
| A4 | Noche (18–23h) |

**Conjunto $B$ — 4 categorías demográficas** (combinación de género y rango de edad), seleccionadas con la siguiente regla explícita: de las 23 combinaciones posibles de género×edad observadas en julio 2024, se identificó que el patrón de preferencia horaria relativa (qué franja ocupa el segundo lugar en volumen, después de "Tarde", que domina en todas las categorías) se bifurca en dos grupos  unas categorías prefieren Noche en segundo lugar, otras prefieren Mañana. Se eligieron las 2 categorías de mayor volumen de cada grupo, maximizando la divergencia real observada en los datos en lugar de tomar arbitrariamente las 4 de mayor volumen total:

| id | Categoría | Viajes totales (jul. 2024) | Preferencia 2º lugar |
|---|---|---|---|
| B1 | Hombre, 21-30 años | 490,378 | Noche |
| B2 | Hombre, 10-20 años | 38,669 | Noche |
| B3 | Hombre, 31-40 años | 455,796 | Mañana |
| B4 | Mujer, 21-30 años | 245,724 | Mañana |

**Variable de decisión:** $x_{ij}=1$ si la franja horaria $a_i$ se asigna como "franja característica" de la categoría demográfica $b_j$ en el modelo de matching (es decir, qué franja horaria representa mejor el patrón de uso de cada categoría, bajo la restricción de que cada franja se usa una sola vez y cada categoría recibe una sola franja); $x_{ij}=0$ en otro caso.

**Score $S_{ij}$:** número total de viajes reales registrados en julio de 2024 cuya hora de arribo cae en la franja $a_i$ y cuya combinación género+edad corresponde a la categoría $b_j$, contado directamente del archivo limpio mediante `pandas.groupby(['franja','categoria'])['viajes'].sum()`. Posteriormente normalizado con escalamiento min-max **por columna** (cada categoría demográfica se normaliza de forma independiente, dividiendo entre su propio rango de variación), para que ninguna categoría domine artificialmente el score total solo por tener mayor volumen absoluto de viajes.

**Matriz $S$ final (normalizada):**

| | B1 | B2 | B3 | B4 |
|---|---|---|---|---|
| A1 (Madrugada) | 0.0000 | 0.0000 | 0.0000 | 0.0000 |
| A2 (Mañana) | 0.8193 | 0.5743 | 0.8798 | 0.8882 |
| A3 (Tarde) | 1.0000 | 1.0000 | 1.0000 | 1.0000 |
| A4 (Noche) | 0.9032 | 0.6808 | 0.8334 | 0.8446 |

## 4. Restricciones

$$\sum_{j=1}^4 x_{ij}=1\ \ \forall i,\qquad \sum_{i=1}^4 x_{ij}=1\ \ \forall j.$$

Cada franja horaria se asigna como "característica" de exactamente una categoría demográfica, y cada categoría demográfica recibe exactamente una franja como representativa — un matching perfecto uno-a-uno, sin repetición ni vacantes.

## 5. Por qué es un problema de matching bipartito

$A$ (franjas horarias) y $B$ (categorías demográficas) son conjuntos disjuntos de naturaleza distinta: ningún elemento de $A$ pertenece a $B$ y viceversa. Toda relación posible ($x_{ij}$) conecta exclusivamente un elemento de $A$ con uno de $B$, nunca dentro del mismo conjunto. La restricción de asignación exactamente uno-a-uno por fila y por columna corresponde a la definición formal de matching perfecto en un grafo bipartito completo $K_{4,4}$.

## 6. Por qué puede formularse como QUBO

Las variables de decisión $x_{ij}\in\{0,1\}$ son binarias. La función objetivo (maximizar $\sum S_{ij}x_{ij}$) y las restricciones de fila/columna son, a lo más, cuadráticas en esas variables. Las restricciones se incorporan como términos de penalización cuadrática dentro de una sola función de energía sin restricciones externas:

$$E(x)=-\sum_{i,j}S_{ij}x_{ij}+\lambda_A\sum_i\Big(\sum_jx_{ij}-1\Big)^2+\lambda_B\sum_j\Big(\sum_ix_{ij}-1\Big)^2.$$

## 7. Resultados: clásico exacto vs. QAOA local

**Solución clásica exacta** (fuerza bruta sobre las 24 permutaciones posibles, verificada de forma independiente con `scipy.optimize.linear_sum_assignment`): score óptimo $2.7914$. La franja A1 (Madrugada) tiene score normalizado 0 en las cuatro categorías por construcción (es el mínimo de cada columna), por lo que **existen múltiples asignaciones óptimas empatadas** que solo difieren en a qué categoría se asigna A1 — este es un hallazgo honesto del dataset, no un error del modelo.

**QAOA local ($p=1$, simulación vectorial con NumPy, 65,536 amplitudes, optimización de $(\gamma,\beta)$ con COBYLA y múltiples puntos de inicio):**

| Penalización $\lambda$ | Valor esperado de energía | Prob. de factibilidad (2000 disparos) | Prob. de acertar el óptimo exacto |
|---|---|---|---|
| 3.0 | 12.73 | 0.65% | 0.00% |
| 1.5 | 2.98 | 2.30% | 0.05% |

**Interpretación:** con una sola capa QAOA ($p=1$), el algoritmo está lejos de concentrar probabilidad en la solución óptima de forma consistente, resultado esperado y documentado en la literatura de QAOA (la expresividad de una sola capa es limitada frente a un problema de 16 variables binarias con restricciones duras). Se observa que penalizaciones $\lambda$ más moderadas, aunque siguen siendo mayores que el rango máximo de $S$ (como exige la teoría para garantizar que el mínimo del QUBO sea factible), producen un paisaje de energía más navegable para el optimizador clásico, mejorando tanto la factibilidad como la probabilidad de acierto exacto.

## 8. Limitaciones y sesgos

- El dataset de Ecobici refleja únicamente a la población que tiene acceso a este sistema de bicicletas compartidas (registro con identificación, tarjeta bancaria o pago en efectivo en módulos físicos), por lo que no representa la movilidad general de la Ciudad de México, sino específicamente a usuarios de Ecobici.
- La categoría "Otro" (género) y los rangos de edad extremos (10-20, 71-80, 81-90 años) tienen volúmenes muy bajos en julio 2024 y fueron excluidos de las 4 categorías finales por esa razón, no por ningún criterio de exclusión deliberada.
- El periodo analizado (julio 2024) es un solo mes; el patrón observado podría variar en otros meses del año (estacionalidad, clima, vacaciones).
- El resultado de QAOA local con $p=1$ no debe interpretarse como una limitación del problema en sí, sino como una limitación conocida de esa profundidad de circuito específica.

## 9. Advertencia ética

Este proyecto es educativo. El modelo no debe usarse para tomar decisiones reales sobre operación del sistema Ecobici, asignación de recursos de movilidad, ni perfilamiento de usuarios por género o edad. Los datos utilizados son agregados estadísticos públicos, sin ningún identificador personal. El resultado de QAOA es una demostración del comportamiento de un algoritmo cuántico variacional sobre una instancia pequeña, no una recomendación operativa para SEMOVI ni para el sistema Ecobici.
