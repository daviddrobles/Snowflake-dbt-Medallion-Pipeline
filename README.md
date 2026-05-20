# Bundesliga Analytics Pipeline: Snowflake + dbt (Medallion Architecture)

Este proyecto implementa un pipeline de datos end-to-end utilizando **dbt (Data Build Tool)** sobre un data warehouse en **Snowflake**. El objetivo principal es transformar un dataset plano y desnormalizado en un modelo analítico robusto (Star Schema), aplicando buenas prácticas de ingeniería analítica, control de calidad y versionado de datos.

> 💡 **Nota de diseño:** Al ser un proyecto de ingeniería analítica independiente, la veracidad estadística de los datos deportivos es secundaria. El foco absoluto del proyecto es metodológico: validar la robustez de la arquitectura, el modelado dimensional y la lógica dbt/Snowflake ante un volumen complejo de relaciones, utilizando datos sintéticos para maximizar las casuísticas de prueba.

## 📌 Contexto y Origen de los Datos

El proyecto parte de un archivo `.csv` original con una única tabla desnormalizada (`JUGADORES`), que contenía información estática sobre futbolistas de la Bundesliga. 

Para estresar el pipeline y simular un entorno de producción real de alta complejidad, **se diseñó y generó un dataset sintético complementario** estructurado en dos entidades adicionales:
* `RESULTADOS`: Simulación estocástica de una temporada completa de partidos basada en los equipos del dataset original.
* `EVENTOS_PARTIDOS`: Datos granulares de eventos por partido (goles, minutos, MVPs) para habilitar análisis a nivel de jugador y equipo.

---

## 🏗️ Arquitectura del Pipeline (Medallion)

El flujo de datos sigue los principios de la arquitectura **Medallion**, asegurando la trazabilidad, modularidad y optimización del rendimiento en Snowflake.

### 1. Capa Staging & Base (Limpieza y Tipado)
* **Modelos Base:** Se encapsula el acceso directo a las sources de Snowflake en una subcapa `BASE` (`base__jugadores`, `base__resultados`, `base__eventos_partidos`). Aquí se realiza el tipado estricto (`CAST`), renombrado homogéneo de campos y el tratamiento de nulos (reemplazando `NULL` por valores por defecto como `'Desconocido'` para evitar fallos de propagación).
* **Normalización:** A partir de los modelos base, se rompe la tabla plana original en entidades atómicas (`stg_equipos`, `stg_agentes`, etc.), generando claves subrogadas únicas mediante *hashing* (`dbt_utils.generate_surrogate_key`) para garantizar la integridad referencial.

### 2. Capa Intermediate (Cálculos Complejos y Optimización)
Modelos diseñados para resolver la lógica de negocio pesada y evitar redundancias en la capa de explotación:
* `int_stats_equipos` y `int_goles_en_contra_equipos`: Agregaciones complejas que calculan rendimiento local/visitante, balance de goles y corrigen desviaciones en la distribución del dataset sintético.
* `int_stats_jugador`: Consolidación de métricas de rendimiento individual (goles totales, MVPs y promedio de minutos por gol).
* `int_stats_partidos`: Precalculado de KPIs globales de la competición (promedios de gol por partido, distribución de victorias).

### 3. Capa Marts / Core (Modelado Dimensional)
Explotación final estructurada bajo un diseño de **Esquema en Estrella (Star Schema)**, optimizado para herramientas de BI como Power BI.

* **Dimensiones:**
    * `dim_tiempo`: Generada dinámicamente para cubrir el rango temporal de la competición.
    * `dim_jugador`: Implementada de forma **Incremental** basada en la clave de negocio para simular la inserción eficiente de nuevos registros.
    * Resto de dimensiones (`dim_equipos`, `dim_agentes`, etc.) derivadas limpias desde Staging.
* **Tablas de Hechos (Fact Tables):**
    * `fact_resultados` y `fact_eventos_partidos`: Eventos transaccionales materializados como tablas estándar.
    * `fact_clasificacion`: Generación de la tabla de posiciones en tiempo real utilizando funciones de ventana analíticas (`RANK() OVER`) basadas en una métrica ponderada de rendimiento (puntos, *goal average* y victorias).
    * `fact_fichajes` y `fact_contratos`: Modelos **Incrementales** que auditan las transacciones de mercado y la vigencia temporal de los contratos (mediante flags booleanos dinámicos).

---

## 🔄 Gobierno del Dato y Características Avanzadas

### 🕒 Control de Históricos (Snapshots - SCD Type 2)
Se ha implementado una estrategia de **Slowly Changing Dimensions (SCD Tipo 2)** mediante el componente `snapshots` de dbt sobre la tabla de jugadores. Utilizando la estrategia `timestamp` basada en el campo `LOAD_AT`, el sistema es capaz de auditar e historizar cambios críticos en el ciclo de vida del jugador (variaciones de valor de mercado, cambios de club o actualizaciones de edad).

### 🧪 Robustez y Calidad (Testing)
El proyecto cuenta con una cobertura de **más de 300 tests automatizados**. Se combinan tests genéricos (`unique`, `not_null`, `relationships`) con configuraciones avanzadas para asegurar que ninguna anomalía de los datos sintéticos o de las cargas incrementales rompa la integridad del data warehouse.

### 🛠️ Modularidad y DRY (Macros)
Uso de macros personalizadas para automatizar lógica SQL repetitiva:
* `ObtenerValores.sql`: Abstracción dinámica para ejecutar selecciones distintas avanzadas necesarias en la normalización.
* `ObtenerVictorias.sql`: Lógica condicional reutilizable para la computación de resultados competitivos.

---

## 📊 Capa de Visualización (BI)
El modelo analítico final fue integrado con **Power BI**, diseñando un dashboard comercial/deportivo orientado a la toma de decisiones estratégicas. El reporte cuenta con segmentación avanzada, cálculo de KPIs dinámicos mediante DAX y paneles de rendimiento de equipos/jugadores. *(Nota: El modelo de datos en el archivo .pbix está optimizado para ejecutarse de forma ultraligera debido a que el 90% del procesamiento y modelado dimensional se realiza directamente en Snowflake a través de dbt).*
