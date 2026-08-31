# Configuración automática de NSGA-II mediante optimización de hiperparámetros

Este repositorio contiene el código, los datos y los resultados asociados al Trabajo Fin de Máster, cuyo objetivo es estudiar la **configuración automática del algoritmo evolutivo multiobjetivo NSGA-II** mediante técnicas de optimización de hiperparámetros.

El estudio emplea **Optuna** para explorar diferentes configuraciones de NSGA-II mediante siete estrategias de muestreo (*samplers*) sobre los **20 problemas de la suite ZCAT**. A partir de los resultados obtenidos se realiza un análisis estadístico del rendimiento, un estudio detallado de la importancia e interacción de los hiperparámetros y, finalmente, se construye y valida una **configuración maestra** de NSGA-II.

---

## Objetivos

El trabajo persigue los siguientes objetivos principales:

* Evaluar el rendimiento de diferentes *samplers* de Optuna en la configuración automática de NSGA-II.
* Comparar los *samplers* sobre los 20 problemas de la suite ZCAT mediante el hipervolumen normalizado.
* Determinar si existen diferencias estadísticamente significativas entre las estrategias de optimización.
* Analizar la velocidad de convergencia y el coste computacional asociado a cada *sampler*.
* Identificar los hiperparámetros de NSGA-II con mayor influencia sobre el rendimiento.
* Estudiar las interacciones entre hiperparámetros y los patrones presentes en las configuraciones de mayor rendimiento.
* Identificar patrones de configuración comunes a través de la suite ZCAT.
* Construir una configuración maestra a partir de las configuraciones de mayor rendimiento.
* Comparar experimentalmente dicha configuración maestra con una configuración base de NSGA-II.

---

## Estructura del repositorio

```text
.
├── notebooks/
│   ├── 00_optimizacion.ipynb
│   ├── 01_extraccion_datos.ipynb
│   ├── 02_ranking_global.ipynb
│   ├── 03_estadistica.ipynb
│   ├── 04_convergencia.ipynb
│   ├── 05_analisis_hiperparametros.ipynb
│   └── 06_validacion_final.ipynb
│
├── database/
│   └── tfm_final.db
│
├── processed/
│   └── [ficheros CSV generados durante el análisis]
│
├── figures/
│   ├── auc/
│   ├── historial/
│   └── [otras figuras generadas]
│
└── README.md
```

### `notebooks/`

Contiene los notebooks que implementan las diferentes fases del análisis. Los notebooks están numerados siguiendo el orden lógico del flujo experimental.

### `database/`

Contiene la base de datos SQLite utilizada por Optuna.

El fichero `tfm_final.db` constituye la **fuente primaria de información de los estudios de optimización**, almacenando los *trials*, hiperparámetros, valores objetivo, estados y demás información asociada a los experimentos.

### `processed/`

Contiene los ficheros CSV generados durante las diferentes etapas del procesamiento y análisis de los resultados.

### `figures/`

Contiene las figuras generadas durante el trabajo, organizadas en subcarpetas según el análisis al que pertenecen.

---

# Flujo de trabajo

El análisis completo sigue el siguiente flujo:

```text
                 ┌──────────────────────┐
                 │ 00 · Optimización    │
                 │      Optuna          │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │   database/          │
                 │   tfm_final.db       │
                 └──────────┬───────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ 01 · Extracción      │
                 │     de datos         │
                 └──────────┬───────────┘
                            │
                            ▼
              ┌─────────────────────────────┐
              │  study_summary.csv          │
              │  master_results.csv         │
              └─────────────┬───────────────┘
                            │
                            ▼
                 ┌──────────────────────┐
                 │ 02 · Ranking global │
                 └──────────┬───────────┘
                            │
                            ▼
              study_summary_normalized.csv
                            │
             ┌──────────────┴──────────────┐
             ▼                             ▼
 ┌──────────────────────┐       ┌──────────────────────┐
 │ 03 · Análisis       │       │ 04 · Convergencia    │
 │     estadístico     │       │     y eficiencia     │
 └──────────────────────┘       └──────────────────────┘
             │
             ▼
 ┌──────────────────────────────────────────┐
 │ 05 · Importancia e interacciones de      │
 │      hiperparámetros                     │
 │                                          │
 │      → selección de estudios             │
 │      → fANOVA                            │
 │      → interacciones                     │
 │      → análisis transversal               │
 │      → configuración maestra              │
 └──────────────────────┬───────────────────┘
                        │
                        ▼
             ┌──────────────────────┐
             │ 06 · Validación final│
             │     de la configuración
             │     maestra           │
             └──────────────────────┘
```

---

# 00 · Optimización con Optuna

**Notebook:** `00_optimizacion.ipynb`

Este notebook constituye la fase experimental inicial del trabajo. En él se ejecutan los estudios de optimización de hiperparámetros de NSGA-II utilizando Optuna.

Se evalúan **7 *samplers*** sobre **20 problemas ZCAT**, dando lugar a un total de:

**7 samplers × 20 problemas = 140 estudios de optimización.**

Cada estudio explora diferentes configuraciones de los hiperparámetros de NSGA-II y registra el valor de hipervolumen obtenido.

Los resultados se almacenan en la base de datos SQLite:

```text
database/tfm_final.db
```

Esta base de datos es posteriormente utilizada como fuente primaria por los análisis posteriores.

---

# 01 · Extracción y consolidación de datos

**Notebook:** `01_extraccion_datos.ipynb`

### Objetivo

Extraer los estudios almacenados en la base de datos SQLite de Optuna y consolidarlos en estructuras tabulares que sirvan como fuente de datos para los análisis posteriores.

### Salidas

| Fichero              | Granularidad                    | Uso                                   |
| -------------------- | ------------------------------- | ------------------------------------- |
| `master_results.csv` | Un *trial* por fila             | Análisis de convergencia y eficiencia |
| `study_summary.csv`  | Un estudio por fila (140 filas) | Ranking global y análisis posteriores |

`study_summary.csv` contiene un resumen de los 140 estudios realizados, mientras que `master_results.csv` conserva la información a nivel de *trial*.

---

# 02 · Análisis global y rankings

**Notebook:** `02_ranking_global.ipynb`

### Pregunta principal

> ¿Qué *sampler* obtiene el mejor rendimiento global y de forma consistente a través de la suite ZCAT?

Este notebook utiliza `study_summary.csv` para:

1. Normalizar el hipervolumen de cada problema a la escala `[0,1]`.
2. Calcular el ranking de cada *sampler* dentro de cada problema.
3. Obtener rankings medios globales.
4. Generar visualizaciones comparativas.
5. Exportar los resultados normalizados.

### Salida principal

```text
processed/study_summary_normalized.csv
```

Este fichero constituye la principal fuente de datos para los análisis estadísticos posteriores.

---

# 03 · Análisis estadístico

**Notebook:** `03_estadistica.ipynb`

### Objetivo

Determinar si existen diferencias estadísticamente significativas entre los siete *samplers* evaluados sobre los 20 problemas ZCAT y cuantificar la magnitud de dichas diferencias.

Se emplean métodos no paramétricos habituales en la comparación de algoritmos de optimización:

### Friedman

Permite determinar si existen diferencias globales entre los siete *samplers*, considerando los 20 problemas como bloques.

### Nemenyi

Cuando el test de Friedman resulta significativo, se utiliza un análisis post-hoc de Nemenyi para identificar qué pares de *samplers* presentan diferencias significativas.

### Critical Difference Diagram

Permite representar visualmente los rankings medios y los grupos de métodos cuyo rendimiento no presenta diferencias estadísticamente significativas.

### Vargha--Delaney $A_{12}$

Se utiliza como medida del tamaño del efecto para complementar los valores p y cuantificar la magnitud de las diferencias entre pares de *samplers*.

---

# 04 · Convergencia, eficiencia temporal y coste computacional

**Notebook:** `04_convergencia.ipynb`

Este notebook estudia la eficiencia del proceso de optimización desde una perspectiva complementaria al rendimiento final.

### Preguntas principales

1. ¿Qué *sampler* alcanza buenas configuraciones con menos *trials*?
2. ¿Se mantiene esta ventaja cuando se considera el tiempo real de ejecución?
3. ¿Cuál es el coste computacional acumulado de cada *sampler*?
4. ¿Existe un compromiso entre calidad final y coste computacional?

Para ello se utilizan los resultados a nivel de *trial* contenidos en:

```text
processed/master_results.csv
```

El análisis incluye la evolución del rendimiento, el número de *trials* necesarios para alcanzar determinados niveles de calidad, la duración de las ejecuciones y diferentes medidas de eficiencia temporal.

---

# 05 · Importancia e interacción de hiperparámetros

**Notebook:** `05_hiperparametros.ipynb`

Este notebook constituye el análisis detallado de la configuración de NSGA-II.

Una vez identificados los resultados globales de los *samplers*, se estudian las configuraciones que han producido los mejores resultados.

### Preguntas de investigación

* ¿Qué hiperparámetros tienen mayor influencia sobre el hipervolumen?
* ¿Es esta influencia consistente entre los diferentes problemas?
* ¿Coinciden los mejores *samplers* en los hiperparámetros relevantes?
* ¿Qué valores aparecen con mayor frecuencia entre los mejores *trials*?
* ¿Existen interacciones relevantes entre hiperparámetros?
* ¿Es posible obtener una configuración general de NSGA-II para la suite ZCAT?

### Selección de estudios

Para evitar analizar exclusivamente el *sampler* con mejor rendimiento global, se seleccionan los **dos mejores estudios de cada problema** según el HV normalizado.

Por tanto:

```text
20 problemas × 2 mejores estudios = 40 estudios
```

Estos 40 estudios constituyen la base del análisis detallado.

### Fuente primaria

Los estudios completos se recuperan desde:

```text
database/tfm_final.db
```

Esto permite trabajar directamente con los objetos `Study` de Optuna y utilizar herramientas como:

* importancia de hiperparámetros;
* fANOVA;
* gráficos de *slice*;
* gráficos de contorno;
* coordenadas paralelas;
* historial de optimización;
* análisis de interacciones.

### Análisis transversal

Además del análisis individual de los 40 estudios, se realiza un análisis conjunto para identificar patrones que trasciendan las particularidades de cada problema.

Se estudian:

* importancia media de los hiperparámetros;
* consistencia de la importancia entre problemas;
* mapa de calor problema–hiperparámetro;
* comparación entre todos los *trials* y el Top 10 %;
* distribución de operadores;
* correlaciones de Spearman;
* interacciones entre hiperparámetros;
* identificación de patrones comunes.

Finalmente, estos resultados se utilizan para construir una **configuración maestra** de NSGA-II.

La configuración maestra se obtiene mediante:

* **mediana** para hiperparámetros numéricos;
* **moda** para hiperparámetros categóricos.

Esta configuración constituye una síntesis representativa de las configuraciones de alto rendimiento y posteriormente se valida experimentalmente.

---

# 06 · Validación final de la configuración maestra

**Notebook:** `06_validacion_final.ipynb`

Este notebook constituye la fase final del estudio.

### Pregunta principal

> ¿La configuración maestra obtenida mediante la optimización de hiperparámetros produce mejores resultados que una configuración base de NSGA-II sobre el conjunto completo de problemas ZCAT?

Se comparan dos configuraciones:

### Configuración base

Se utiliza una parametrización fija de NSGA-II:

```text
pop_size        = 100
crossover       = SBX
crossover_prob  = 0.9
crossover_eta   = 20.0
mutation        = Polynomial Mutation
mutation_prob   = 1/n
mutation_eta    = 20.0
selection       = Tournament
selection_size  = 2
```

### Configuración maestra

Utiliza los operadores y valores obtenidos a partir del análisis transversal del Top 10 % de configuraciones de mayor rendimiento.

Ambas configuraciones se ejecutan bajo el **mismo presupuesto máximo de 50.000 evaluaciones**, garantizando que la comparación se realice bajo unas condiciones computacionales equivalentes en términos de número de evaluaciones.

### Evaluación

La comparación se realiza sobre los **20 problemas ZCAT** mediante:

* hipervolumen final;
* comparación estadística entre configuraciones;
* curvas de convergencia;
* evolución del HV durante la ejecución;
* comparación de frentes de Pareto en problemas representativos.

Para el análisis de convergencia se utilizan cuatro puntos de control:

```text
25 % → 12.500 evaluaciones
50 % → 25.000 evaluaciones
75 % → 37.500 evaluaciones
100 % → 50.000 evaluaciones
```

Además, se utilizan cinco problemas representativos para la visualización detallada de las curvas y de los frentes de Pareto:

```text
ZCAT1
ZCAT6
ZCAT11
ZCAT16
ZCAT20
```

Los frentes de Pareto permiten complementar la comparación cuantitativa del hipervolumen mediante una interpretación geométrica de la calidad, cobertura y densidad de las soluciones obtenidas.

---

# Datos y resultados

El repositorio separa los datos originales de los resultados procesados y de las figuras generadas.

## `database/`

Contiene la base de datos original de Optuna:

```text
database/
└── tfm_final.db
```

Este fichero contiene los estudios completos de optimización y constituye la fuente primaria para el análisis de hiperparámetros.

## `processed/`

Contiene los datos derivados de los experimentos y los análisis:

```text
processed/
├── study_summary.csv
├── study_summary_normalized.csv
├── master_results.csv
└── ...
```

Los ficheros adicionales generados durante los análisis se almacenan también en esta carpeta.

## `figures/`

Contiene todas las figuras generadas para el análisis y la memoria del TFM.

Las figuras se organizan en diferentes subcarpetas según su finalidad, por ejemplo:

```text
figures/
├── hiperparams/
├── transversal/
└── ...
```

---

# Reproducibilidad

Para reproducir el análisis completo se recomienda ejecutar los notebooks en orden:

```text
00 → 01 → 02 → 03 → 04 → 05 → 06
```

El orden es relevante, ya que varios notebooks utilizan como entrada los ficheros generados por etapas anteriores.

En particular:

```text
00_optimizacion
        ↓
   tfm_final.db
        ↓
01_extraccion_datos
        ↓
study_summary.csv
master_results.csv
        ↓
02_ranking_global
        ↓
study_summary_normalized.csv
        ↓
03_estadistica
04_convergencia
05_hiperparametros
        ↓
configuración maestra
        ↓
06_validacion_final
```

---

# Resumen del estudio

En conjunto, el repositorio implementa un flujo completo de **optimización, análisis y validación de hiperparámetros de NSGA-II**:

```text
Optimización automática
        ↓
140 estudios Optuna
        ↓
20 problemas ZCAT × 7 samplers
        ↓
Evaluación global
        ↓
Análisis estadístico
        ↓
Selección de los 40 mejores estudios
        ↓
Análisis de hiperparámetros
        ↓
Identificación de patrones comunes
        ↓
Configuración maestra
        ↓
Validación sobre ZCAT1–ZCAT20
```

De este modo, el proyecto no se limita a identificar qué estrategia de optimización obtiene mejores resultados, sino que utiliza la información generada durante la búsqueda para **caracterizar cómo debe configurarse NSGA-II** y comprobar posteriormente si los patrones identificados pueden trasladarse a una configuración conjunta con capacidad de mejorar el rendimiento sobre la suite ZCAT.
