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

**Conjunto $B$ — 4 categorías demográficas** (combinación de género y rango de edad), seleccionadas con la siguiente regla explícita: de las 23 combinaciones posibles de género×edad observadas en julio 2024, se identificó que el patrón de preferencia horaria relativa (qué franja ocupa el segundo lugar en volumen, después de "Tarde", que domina en todas las categorías) se bifurca en dos grupos — unas categorías prefieren Noche en segundo lugar, otras prefieren Mañana. Se eligieron las 2 categorías de mayor volumen de cada grupo, maximizando la divergencia real observada en los datos en lugar de tomar arbitrariamente las 4 de mayor volumen total:

| id | Categoría | Viajes totales (jul. 2024) | Preferencia 2º lugar |
|---|---|---|---|
| B1 | Hombre, 21-30 años | 490,378 | Noche |
| B2 | Hombre, 10-20 años | 38,669 | Noche |
| B3 | Hombre, 31-40 años | 455,796 | Mañana |
| B4 | Mujer, 21-30 años | 245,724 | Mañana |

**Nota sobre la estructura del dataset:** $A$ y $B$ son los dos lados (conjuntos de vértices) del grafo bipartito $K_{4,4}$, no dos datasets independientes. Toda la información cuantitativa que alimenta la optimización vive en una sola matriz $S \in \mathbb{R}^{4\times 4}$ derivada del único archivo fuente `afluencia_desglosada_acumulada_2024_07.csv`. `A_df` y `B_df` en el notebook son tablas índice con los identificadores cortos de cada vértice; el dataset entregado en `data/dataset_real_4x4.csv` contiene la matriz $S$ ya construida en formato largo (`a_id, b_id, score`).

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

## 7. Resultados: clásico exacto vs. QAOA local vs. pipeline híbrido

**Solución clásica exacta** (fuerza bruta sobre las 24 permutaciones factibles, verificada de forma independiente con `scipy.optimize.linear_sum_assignment`): score óptimo $2.7914$, energía QUBO $-2.7914$. Existen dos asignaciones óptimas empatadas con el mismo score que difieren únicamente en el par {A1, A3} ↔ {B2, B3} (ver sección 8 para la explicación de esta degeneración).

**QAOA local ($p=2$, simulación vectorial con NumPy, 65,536 amplitudes, optimización de $(\gamma_1,\beta_1,\gamma_2,\beta_2)$ con COBYLA, semilla 2026):**

| Método | Energía mejor observada | Score mejor observado | Prob. factibilidad | Prob. óptimo clásico |
|---|---|---|---|---|
| Clásico exacto | −2.7914 | 2.7914 | — | — |
| QAOA local crudo (2000 shots) | −2.7914 | 2.7914 | 1.6% (32/2000) | 0.15% (3/2000) |
| QAOA local con reparación clásica | −2.7914 | 2.7914 | 100% por construcción | 15.6% |

**Probabilidades ideales bajo la distribución teórica del circuito QAOA (sin muestreo):** factibilidad 2.10%, óptimo clásico exacto 0.088%.

**Interpretación:** con la configuración usada ($p=2$, 1 reinicio COBYLA, 25 iteraciones), QAOA local crudo encuentra el óptimo en el muestreo pero con baja concentración de probabilidad — resultado esperado para una profundidad QAOA modesta sobre un problema de 16 variables binarias con restricciones duras y un mixer estándar $X$ que no preserva la factibilidad [Farhi, Goldstone & Gutmann, 2014]. El pipeline híbrido (QAOA + reparación clásica posterior) eleva la factibilidad al 100% por construcción y la probabilidad de observar el óptimo clásico exacto a 15.6%; este aumento corresponde íntegramente al postprocesamiento clásico determinista, no al circuito cuántico, y se reporta como tal.

## 8. Limitaciones y sesgos

El dataset de Ecobici representa únicamente a la población con acceso al sistema (registro con identificación, tarjeta bancaria o pago en módulos físicos), por lo que las distribuciones observadas no son extrapolables a la movilidad general de la Ciudad de México. La categoría "Otro" en género y los rangos de edad de bajo volumen (10-20, 71-80, 81-90 años) quedan fuera de las cuatro categorías finales por su frecuencia en julio 2024, no por criterios de exclusión deliberada — esto introduce un sesgo de cobertura demográfica que conviene tener presente al interpretar la asignación óptima. El periodo analizado es un solo mes (julio 2024), lo que impide capturar estacionalidad anual, efectos climáticos, periodos vacacionales o cambios estructurales de operación.

Tras la normalización min-max por columna, las filas A1 (Madrugada) y A3 (Tarde) presentan valores constantes (0 y 1 respectivamente) en las cuatro categorías demográficas, porque el orden jerárquico A3 > A4 > A2 > A1 se mantiene monótonamente entre todas las categorías. Esto reduce la dificultad combinatoria efectiva del problema: la optimización discrimina solo sobre el par de filas (A2, A4), y como consecuencia existen dos asignaciones óptimas empatadas con score 2.7914 que difieren únicamente en la asignación del par {A1, A3} ↔ {B2, B3}. Es un comportamiento real de los datos (la jerarquía de franjas de Ecobici es consistente entre demografías), no un defecto del modelo, pero implica que esta instancia particular no representa un caso de máxima dificultad combinatoria para QAOA [Lucas, 2014].

A nivel algorítmico, el desempeño de QAOA local con profundidad modesta no debe interpretarse como una limitación del problema sino de la expresividad de pocas capas variacionales: la teoría establece que la probabilidad de concentración en el óptimo crece con $p$ [Farhi, Goldstone & Gutmann, 2014], y el mixer estándar $\exp(-i\beta\sum_k X_k)$ no preserva la factibilidad del matching, lo cual exige reparación clásica posterior o mixers que preserven restricciones (XY-mixer) en versiones más sofisticadas del pipeline [Hadfield et al., 2019].

Finalmente, la Parte D del notebook (ejecución en hardware real de IBM Quantum, activable con `USAR_IBM_QUANTUM = True`) no se ejecutó en esta entrega por restricciones de acceso al QPU y tiempos de cola del plan abierto de IBM Quantum Platform. La comparación cuádruple proyectada por el enunciado del proyecto — clásico exacto, QAOA local, hardware real y pipeline híbrido — queda restringida en este informe a tres ramas: clásico exacto, QAOA local crudo y QAOA local con reparación clásica. La infraestructura para activar la comparación con hardware está implementada en las celdas 30-37 del notebook y queda explícitamente como extensión inmediata para una versión posterior del trabajo.

## 9. Interpretación mínima del resultado

**1. ¿Cuál fue la mejor asignación encontrada?** A1→B3, A2→B4, A3→B2, A4→B1; es decir, Madrugada↔Hombre 31-40, Mañana↔Mujer 21-30, Tarde↔Hombre 10-20, Noche↔Hombre 21-30. Existe una segunda asignación empatada con idéntico score (A1↔B2, A2↔B4, A3↔B3, A4↔B1), diferenciada solo por el par {A1, A3} ↔ {B2, B3}, consecuencia de la degeneración descrita en la sección 8.

**2. ¿Cuál fue su score en el dominio?** 2.7914 (suma de los cuatro valores normalizados min-max por columna), idéntico al óptimo clásico exacto.

**3. ¿La asignación cumple todas las restricciones?** Sí. Es una permutación uno-a-uno válida sobre las 4 franjas y las 4 categorías, sin repetición ni vacantes. Verificado por `is_feasible(best_x_qaoa) == True` en el notebook.

**4. ¿QAOA local observó el óptimo clásico?** Sí. En las 2000 muestras locales se observó el bitstring óptimo exacto 3 veces (probabilidad empírica 0.15%); la probabilidad ideal del óptimo bajo la distribución teórica del circuito es 0.088%.

**5. ¿Qué tan frecuente fue observar soluciones factibles?** Probabilidad ideal: 2.10%. Probabilidad empírica en 2000 muestras: 1.6% (32/2000). La baja factibilidad es consecuencia directa del mixer estándar $X$, que no preserva la estructura de matching uno-a-uno; el espacio factible representa $4!/2^{16} = 24/65536 \approx 0.037\%$ del espacio binario total, por lo que el QAOA crudo ya está concentrando ~50× más masa de probabilidad en la región factible que un muestreo uniforme.

**6. ¿Qué limitaciones tiene el modelo 4×4?** Tres tipos: (a) tamaño que permite verificación por fuerza bruta pero que no representa la dimensionalidad de un problema de asignación realista; (b) degeneración monótona A1/A3 que reduce la dificultad combinatoria efectiva (sección 8); (c) cobertura demográfica y temporal restringida a usuarios de Ecobici en julio 2024.

**7. ¿Qué cambiaría si el dataset creciera?** El espacio de estados binarios crece como $2^{n^2}$ mientras el espacio factible solo como $n!$, por lo que la fracción factible colapsa exponencialmente; esto exige mixers que preserven restricciones (XY-mixer [Hadfield et al., 2019]) o penalizaciones $\lambda$ cuidadosamente recalibradas en lugar del mixer estándar usado aquí. La verificación clásica por fuerza bruta deja de ser viable para $n>10$ ($10! = 3.6\times 10^6$ permutaciones, manejable; $15! \approx 1.3\times 10^{12}$, ya no). La reparación clásica por `linear_sum_assignment` escala como $O(n^3)$, lo cual sigue siendo barato, pero el número de muestras necesarias para cubrir un espacio factible que crece factorialmente sí escala mal.

**8. ¿Qué riesgos éticos existen y cómo se mitigaron?** Los riesgos son tres: (i) inferencia indebida de preferencias por género o edad si el resultado se interpreta como recomendación operativa; (ii) sesgo de representatividad por usar únicamente datos de Ecobici; (iii) sobreinterpretación de un experimento educativo como evidencia para política pública. Mitigaciones: el dataset original es agregado estadístico sin identificadores personales (no PII por construcción); la sección 10 declara explícitamente que el resultado no es recomendación operativa; las limitaciones de representatividad están documentadas en la sección 8.

**9. Si se usó hardware real, ¿cómo compara contra QAOA local?** No se ejecutó hardware real en esta entrega (`USAR_IBM_QUANTUM = False`); la comparación queda fuera del alcance de este informe. La infraestructura para activarla está implementada en la Parte D del notebook (celdas 30-37) pero no se utilizó por restricciones de acceso a la cola del plan abierto de IBM Quantum. Queda como extensión inmediata.

**10. Si se usó reparación clásica, ¿qué parte del resultado corresponde al postprocesamiento híbrido?** Sí se aplicó reparación clásica al QAOA local (celda 79: `repair_x_to_feasible_assignment`, proyecta cada bitstring observado al matching factible más cercano mediante `linear_sum_assignment` sobre $S_{norm} + 0.15 \cdot \text{observed\_matrix}$, donde el bono de 0.15 favorece preservar la información de la muestra cuántica cuando ya es factible o casi factible). Efectos medidos: la factibilidad pasó de 1.6% (QAOA crudo, 32/2000) a 100% por construcción (QAOA reparado); la probabilidad de observar el óptimo clásico exacto pasó de 0.15% a 15.6%. **Esta mejora no debe atribuirse al circuito cuántico**: corresponde íntegramente al postprocesamiento clásico determinista, que proyectaría cualquier muestra — incluido ruido uniforme — al matching factible de mayor score. Se reporta como salida del pipeline híbrido (cuántico + clásico), nunca como desempeño cuántico puro.

## 10. Advertencia ética

Este proyecto es educativo. El modelo no debe usarse para tomar decisiones reales sobre operación del sistema Ecobici, asignación de recursos de movilidad, ni perfilamiento de usuarios por género o edad. Los datos utilizados son agregados estadísticos públicos, sin ningún identificador personal. El resultado de QAOA es una demostración del comportamiento de un algoritmo cuántico variacional sobre una instancia pequeña, no una recomendación operativa para SEMOVI ni para el sistema Ecobici. La salida de QAOA no debe presentarse como una recomendación real de política pública, salud, empleo, vivienda o asignación de recursos.

## Referencias

- Farhi, E., Goldstone, J. & Gutmann, S. (2014). A Quantum Approximate Optimization Algorithm. *arXiv:1411.4028*.
- Hadfield, S., Wang, Z., O'Gorman, B., Rieffel, E. G., Venturelli, D. & Biswas, R. (2019). From the Quantum Approximate Optimization Algorithm to a Quantum Alternating Operator Ansatz. *Algorithms*, 12(2), 34.
- Lucas, A. (2014). Ising formulations of many NP problems. *Frontiers in Physics*, 2, 5.
