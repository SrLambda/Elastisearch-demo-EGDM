# Elasticsearch-demo-EGDM

Demostración local de búsquedas y agregaciones en Elasticsearch con los datos de ejemplo oficiales de Kibana.

## Tamaño medido de los conjuntos de datos

Estos son los conteos sin filtros obtenidos en la instancia local de Elasticsearch 9.5.6. Las tres consultas respondieron con un fragmento correcto y ninguno fallido. Son cantidades de documentos, no tamaños en disco ni valores garantizados para otras instalaciones.

| Conjunto de datos | Índice de Elasticsearch | Documentos |
| --- | --- | ---: |
| Pedidos de comercio electrónico | `kibana_sample_data_ecommerce` | 4 675 |
| Vuelos | `kibana_sample_data_flights` | 13 014 |
| Registros web | `kibana_sample_data_logs` | 14 074 |

## 1. Iniciar el entorno

Necesitas Docker en ejecución, Docker Compose (`docker compose`), acceso a Internet para descargar las imágenes y dependencias en el primer arranque, y los puertos locales 9200, 5601 y 8888 libres.

Desde la raíz del repositorio:

```bash
docker compose up -d --build
docker compose ps
```

| Servicio | URL |
| --- | --- |
| Elasticsearch | <http://localhost:9200> |
| Kibana | <http://localhost:5601> |
| JupyterLab | <http://localhost:8888/lab/tree/work/00_indice.ipynb> |

Espera a que Kibana termine de iniciar antes de cargar los datos. Los servicios se publican solo en la interfaz local; Elasticsearch y Jupyter están configurados sin autenticación para esta demostración. No los expongas a la red.

## 2. Cargar los ejemplos y comprobar los conteos

1. En el buscador global de Kibana, abre **Integrations**. Dentro de esa página, busca **Sample Data** y abre la integración.
2. Instala desde la interfaz los tres conjuntos oficiales: pedidos de comercio electrónico, vuelos y registros web. Si ya están instalados, no necesitas volver a cargarlos. Consulta la [guía oficial de datos de ejemplo](https://www.elastic.co/docs/manage-data/ingest/sample-data) si la interfaz difiere.
3. Abre **Dev Tools** → **Console** y ejecuta una petición a la vez:

```http
GET kibana_sample_data_ecommerce/_count

GET kibana_sample_data_flights/_count

GET kibana_sample_data_logs/_count
```

Compara el campo `count` con la tabla y revisa que `_shards.failed` sea `0`. Si un índice no existe, comprueba que el conjunto correspondiente se haya instalado.

## 3. Ejecutar y contrastar los experimentos

Abre el [cuaderno índice](jupyter/notebooks/00_indice.ipynb) en JupyterLab y elige un experimento:

| Cuaderno | Qué muestra |
| --- | --- |
| [Búsqueda y ordenamiento](jupyter/notebooks/01_busqueda_ordenamiento.ipynb) | Filtra y ordena pedidos; devuelve hasta 10 por consulta con la configuración inicial. |
| [Precio y destino](jupyter/notebooks/02_experimento.ipynb) | Filtra vuelos por precio y país de destino; devuelve hasta 20 con la configuración inicial. |
| [Agrupación y promedio](jupyter/notebooks/03_experimento.ipynb) | Grafica el promedio de bytes para hasta cinco códigos de respuesta. Con `size: 0`, no devuelve documentos individuales. |

1. Ejecuta las celdas de código de arriba abajo, desde la conexión hasta la visualización. El kernel del contenedor usa `http://elasticsearch:9200`.
2. Copia la petición `GET …/_search` del experimento, incluido su cuerpo JSON, en la consola de Kibana. Si cambiaste los parámetros en Python, aplica los mismos cambios en la consola.
3. Compara los resultados devueltos (`hits.hits`) o, en el experimento de registros, los grupos y promedios de `aggregations` con la gráfica.

`_count` mide el total del índice. En cambio, `size` en una búsqueda limita los documentos que devuelve después de aplicar los filtros; en una agregación `terms`, el `size` de la agregación limita los grupos. Por eso el número de elementos graficados no representa el tamaño del conjunto. Esta demostración no mide latencia ni rendimiento.
