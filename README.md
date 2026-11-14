# Análisis de Big Data con Apache Hadoop y MapReduce

Informe técnico de la Especialización en Analítica y Ciencia de Datos (Universidad de Cundinamarca). Este repositorio reúne el cuaderno, los scripts, los datos, los resultados y los logs del análisis de 50.000 reclamaciones de seguros procesadas con el paradigma MapReduce sobre Apache Hadoop.

## Resumen

Se ejecutaron dos jobs de MapReduce con Hadoop Streaming y scripts en Python:

1. **Job 1:** cantidad, monto total, promedio y máximo de las reclamaciones por ramo.
2. **Job 2:** cantidad de siniestros por mes, usando un *combiner*.

Los resultados se validaron contra un cálculo equivalente en pandas y se presentan con gráficas. El cuaderno incluye el marco conceptual, la explicación de cada celda y las conclusiones.

## Entorno

| Componente | Versión |
|---|---|
| Plataforma | Google Colab |
| Java | OpenJDK 8 |
| Apache Hadoop | 3.3.6 (modo *standalone*) |
| Python | 3 (pandas, NumPy, matplotlib) |

Hadoop se ejecuta en modo *standalone*: usa el sistema de archivos local y el `LocalJobRunner`, sin HDFS ni YARN. Es MapReduce real (mismo `.jar` y mismos counters), pero sin las ventajas de un clúster como paralelismo entre nodos, localidad de datos y tolerancia a fallos.

## Estructura del repositorio

```
.
├── analisis_bigdata_mapreduce.ipynb   # cuaderno con el desarrollo completo
├── scripts/
│   ├── mapper_ramo.py                 # Job 1: emite (ramo, valor_reclamado)
│   ├── reducer_ramo.py                # Job 1: n, total, promedio y máximo por ramo
│   ├── mapper_mes.py                  # Job 2: emite (AAAA-MM, 1)
│   └── reducer_suma.py                # Job 2: suma por clave (reducer y combiner)
├── data/
│   └── siniestros.csv                 # dataset simulado (50.000 reclamaciones)
├── output/
│   ├── output_job1/                   # resultados del Job 1
│   └── output_job2/                   # resultados del Job 2
├── logs/
│   ├── job1.log                       # log de ejecución con counters
│   └── job2.log
└── evidencias/                        # gráficas y capturas
```

## Dataset

El dataset es **simulado**, generado dentro del cuaderno con semilla fija (`numpy.random.default_rng(42)`), por lo que es reproducible. Representa reclamaciones de una aseguradora ficticia.

| Columna | Descripción |
|---|---|
| `id_siniestro` | Identificador único |
| `fecha` | Fecha del siniestro (`AAAA-MM-DD`), entre 2024 y 2025 |
| `ciudad` | Ciudad donde ocurrió |
| `ramo` | Motos, Mascotas, Salud u Hogar |
| `cobertura` | Cobertura afectada |
| `estado` | Aprobado, Rechazado o En revisión |
| `valor_reclamado` | Monto reclamado en COP |

Se incluyen 50 filas dañadas a propósito (valor vacío o `N/A`) para comprobar que el mapper las descarta y las reporta con un counter personalizado.

## Cómo ejecutarlo

1. Abrir `analisis_bigdata_mapreduce.ipynb` en Google Colab.
2. Ejecutar *Entorno de ejecución → Ejecutar todo*, en orden y sin saltar celdas.

El cuaderno instala Java y descarga Hadoop (~700 MB), genera el dataset, escribe los scripts, ejecuta los jobs y produce las gráficas. Colab es un entorno temporal: si la sesión se reinicia, hay que volver a ejecutar todo desde la primera celda.

### Comando de ejecución (Job 1)

```bash
hadoop jar $HADOOP_HOME/share/hadoop/tools/lib/hadoop-streaming-3.3.6.jar \
  -D mapreduce.job.name="Job1_Estadisticas_por_Ramo" \
  -files mapper_ramo.py,reducer_ramo.py \
  -mapper "python3 mapper_ramo.py" \
  -reducer "python3 reducer_ramo.py" \
  -input data/siniestros.csv \
  -output output_job1
```

El Job 2 usa el mismo formato, con `mapper_mes.py` y `reducer_suma.py`, e incorpora `-combiner "python3 reducer_suma.py"`. El directorio de salida no debe existir antes de ejecutar el job.

## Resultados principales

**Job 1: reclamaciones por ramo**

| Ramo | Reclamaciones | Monto total (millones COP) | Promedio (COP) | Máximo (COP) |
|---|---:|---:|---:|---:|
| Salud | 12.714 | 22.686 | 1.784.350 | 47.876.164 |
| Motos | 19.846 | 22.277 | 1.122.495 | 43.777.078 |
| Hogar | 7.422 | 6.581 | 886.658 | 23.692.101 |
| Mascotas | 9.968 | 2.375 | 238.288 | 3.785.752 |

- Salud y Motos concentran el 83,4 % del monto reclamado.
- Motos tiene más reclamaciones; Salud, el monto promedio más alto.
- El monto máximo supera al promedio entre 16 y 39 veces según el ramo (distribución de cola larga).

**Job 2: siniestros por mes**

Junio, julio, noviembre y diciembre promedian cerca de 2.676 siniestros mensuales, frente a unos 1.787 en el resto del año. El patrón se repite en 2024 y 2025.

**Counters relevantes**

| Job | Counter | Valor |
|---|---|---:|
| 1 | Map input records | 50.001 |
| 1 | Map output records | 49.950 |
| 1 | Calidad / filas_invalidas | 50 |
| 1 | Reduce input groups | 4 |
| 2 | Combine input records | 50.000 |
| 2 | Combine output records | 24 |

Los resultados de ambos jobs coinciden con el cálculo equivalente en pandas (sección 11 del cuaderno).

## Limitaciones

- Entorno *standalone*: no se aprecian el paralelismo real, la localidad de datos ni la tolerancia a fallos de un clúster.
- Dataset simulado y de tamaño reducido; la estacionalidad fue incorporada al generarlo.
- Las cifras dependen de la semilla utilizada.

## Referencias

Apache Software Foundation. (2023). *MapReduce tutorial*. Apache Hadoop 3.3.6 documentation. https://hadoop.apache.org/docs/r3.3.6/hadoop-mapreduce-client/hadoop-mapreduce-client-core/MapReduceTutorial.html

Apache Software Foundation. (2023). *Hadoop streaming*. Apache Hadoop 3.3.6 documentation. https://hadoop.apache.org/docs/r3.3.6/hadoop-streaming/HadoopStreaming.html

Dean, J., & Ghemawat, S. (2004). MapReduce: Simplified data processing on large clusters. *Proceedings of the 6th Symposium on Operating Systems Design and Implementation (OSDI '04)*, 137–150. USENIX Association.

White, T. (2015). *Hadoop: The definitive guide* (4.ª ed.). O'Reilly Media.
