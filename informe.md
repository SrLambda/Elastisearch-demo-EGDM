## Experimento 3: Impacto de la estrategia de ejecución al agrupar y promediar en Elasticsearch
**Autora:** María Ignacia Parra Muñoz  
**Dataset:** `kibana_sample_data_logs` (Registros web de ejemplo de Kibana)

### 1. Pregunta e hipótesis
**Pregunta:** Al agrupar registros web por código de respuesta HTTP (`200`, `404`, `503`) y calcular el promedio de `bytes` transferidos, ¿cuánto cambia el tiempo de respuesta de Elasticsearch según la forma interna en que ejecuta el cálculo?

**Hipótesis:**
1. Activar la memoria caché (`request_cache=true`) dará el menor tiempo de respuesta, porque al repetir exactamente la misma consulta de resumen (`size: 0`), Elasticsearch devuelve el resultado ya guardado sin volver a calcular.
2. Al desactivar la caché, los métodos que leen columnas ya preparadas en disco (`doc_values`) serán mucho más rápidos que calcular el campo al vuelo (`runtime_mappings`), porque este último obliga a abrir el documento JSON completo de cada registro uno por uno.

### 2. Diseño del experimento
* **Qué se modifica (variable independiente):** La forma interna en que Elasticsearch resuelve exactamente la misma agrupación (`terms`) y promedio (`avg`), probando 4 configuraciones distintas:
  1. **Caché activa (`request_cache=true`):** Reutiliza el resultado final guardado en memoria para evitar recalcular.
  2. **Estándar sin caché (`global_ordinals`):** Lee las columnas precalculadas (`doc_values`) y reemplaza cada texto por un número pequeño (por ejemplo, `"200" → 0`, `"404" → 1`, `"503" → 2`) para sumar directamente en un arreglo. Es la opción que usa Elasticsearch por defecto.
  3. **Tabla hash sin caché (`execution_hint: "map"`):** También lee las columnas precalculadas (`doc_values`), pero agrupa los textos directamente usando una tabla hash en memoria.
  4. **Cálculo al vuelo sin caché (`runtime_mappings`):** No usa datos preindexados; crea un campo temporal durante la consulta leyendo el JSON original (`_source`) y ejecutando un pequeño código (script *Painless*) registro por registro.
* **Qué se mide (variables dependientes):**
  * **Mediana total (ms):** Tiempo completo de la petición medido desde Python (20 repeticiones, tras 3 ejecuciones de calentamiento).
  * **ES `took` promedio (ms):** Tiempo promedio reportado por Elasticsearch.
  * **Profile API (`µs`):** Tiempo interno puro que tarda el motor en hacer la agrupación, medido en microsegundos con `"profile": true` para observar el trabajo real sin el ruido de la red.
* **Qué se mantiene fijo (control):** El mismo índice (`kibana_sample_data_logs`), exactamente el mismo cálculo (`terms` sobre el código HTTP y `avg` sobre `bytes`), `"size": 0` (sin traer registros individuales) y el mismo contenedor Docker (`elasticsearch:9.5.6`, un solo nodo con límite de 1 GB de RAM).

### 3. Resultados
Primero se verificó que las 4 configuraciones devolvieran exactamente los mismos promedios: código `200` con `5897,85` bytes, `404` con `5049,24` bytes y `503` con `0,00` bytes. Esto confirma que la comparación es justa: misma tarea y mismo resultado, cambiando solo el método interno.

| Configuración | Cómo trabaja internamente | Mediana total (ms) | ES `took` prom. (ms) | Tiempo interno Profile (µs) |
| :--- | :--- | :---: | :---: | :---: |
| **1. Caché activa** | Devuelve el resultado guardado en caché | 3,28 | 1,05 | N/A (caché) |
| **2. Estándar** | Columnas listas (`doc_values`) + arreglos (`global_ordinals`) | 3,72 | 1,05 | 1.145,6 |
| **3. Tabla hash** | Columnas listas (`doc_values`) + tabla hash (`map`) | 5,72 | 2,50 | 2.074,2 |
| **4. Al vuelo** | Abre cada JSON (`_source`) + script *Painless* (`runtime_mappings`) | 94,02 | 91,75 | 93.962,4 |

*(Insertar aquí el gráfico comparativo generado en la sección 5 del cuaderno `03_experimento.ipynb`)*

### 4. Conclusión y escalabilidad
Los resultados confirman ambas hipótesis y muestran con claridad por qué Elasticsearch es rápido para resumir datos cuando están bien modelados:

* **Por qué el cálculo al vuelo es el más lento:** `runtime_mappings` tardó `94,02 ms` en total y `93.962,4 µs` en cálculo interno, resultando **82 veces más lento** por dentro que el método estándar (`1.145,6 µs`). Esto ocurre porque, al no existir una columna precalculada, el motor debe descomprimir el JSON completo (`_source`) y ejecutar el script *Painless* en cada uno de los ~14.000 registros. Su única ventaja es la flexibilidad: permite crear agrupaciones nuevas sobre la marcha sin reindexar ni ocupar disco extra.
* **Por qué los arreglos ganan a la tabla hash:** Tanto `global_ordinals` como `map` leen las columnas ya ordenadas (`doc_values`) sin abrir los JSON, por lo que ambos responden en pocos milisegundos. Sin embargo, `map` (`2.074,2 µs`) fue **1,8 veces más lento** internamente que `global_ordinals` (`1.145,6 µs`), ya que operar con cadenas de texto en una tabla hash requiere más trabajo de CPU que sumar directamente en un arreglo usando números enteros.
* **El efecto de la caché:** `request_cache=true` obtuvo el menor tiempo total (`3,28 ms`, casi **29 veces más rápido** que `runtime_mappings`). Su ventaja frente al método estándar sin caché (`3,72 ms`) parece pequeña en esta prueba porque procesar ~14.000 registros con `doc_values` toma apenas ~1 ms, y el resto del tiempo (~2 ms) corresponde al viaje HTTP entre Python y Docker.

**Cómo escalaría a 10 millones de registros (> 1 GB):** Se trabajó con la muestra de ~14.000 registros debido al límite de `1 GB` de RAM del contenedor local. Si el volumen creciera unas 700 veces hasta alcanzar 10 millones de registros en un solo nodo:
* El método al vuelo (`runtime_mappings`) crecería de forma lineal al abrir cada JSON y pasaría de ~94 ms a **más de un minuto**, volviéndolo inviable para paneles en tiempo real a menos que antes se filtren muy pocos documentos (por ejemplo, solo la última hora).
* El método estándar sobre `doc_values` (`global_ordinals`) escalaría de forma controlada dividiendo el índice en varios *shards* (particiones) repartidos en distintos nodos para calcular los grupos en paralelo y luego unir los resultados, mientras la caché (`request_cache`) absorbería las consultas repetitivas.