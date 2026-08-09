# Olist E-Commerce – Azure Data Pipeline Assessment

## Arquitectura implementada

```
Kaggle (CSV) → Azure Blob Storage (raw/) → Azure Data Factory (Copy Data) → Azure SQL Database → vw_sales (view) → SQL Queries
```

**Servicios usados:**
- Azure Storage Account (Blob Storage)
- Azure Data Factory
- Azure SQL Database

---

## Parte 2: Estructura del Storage

Container: `ecommerce`

```
ecommerce/
├── raw/
│   ├── olist_orders_dataset.csv
│   ├── olist_customers_dataset.csv
│   └── olist_order_payments_dataset.csv
└── processed/
```

**Por qué esta estructura:** `raw/` conserva los archivos exactamente como llegaron, sin modificarlos — es la fuente de la verdad si algo falla más adelante en el pipeline y hay que reprocesar. `processed/` queda reservada para datos ya transformados, separando claramente la etapa de ingesta de la etapa de procesamiento (patrón raw → processed común en ingeniería de datos). El container se configuró con acceso privado (no anónimo) para no exponer los datos públicamente.

---

## Parte 8: Preguntas conceptuales

**1. ¿Por qué usarías Azure Storage para guardar los archivos originales?**

Porque separa la ingesta de datos crudos de su procesamiento posterior. Si algo falla al cargar a SQL, el archivo original sigue disponible para reprocesar sin necesidad de volver a descargarlo. Además, almacenar archivos planos en Blob Storage es mucho más económico que guardarlos directamente en una base de datos, y sirve como respaldo histórico de la fuente original.

**2. ¿Cuál es la diferencia entre Azure Blob Storage y Azure Data Lake Storage Gen2?**

Blob Storage es almacenamiento de objetos plano: los archivos se guardan con nombres que simulan carpetas mediante prefijos, pero no existe una jerarquía real. Data Lake Storage Gen2 es Blob Storage con el **namespace jerárquico** habilitado, lo que permite carpetas reales y operaciones más eficientes sobre ellas (renombrar, mover, listar). Gen2 está pensado para cargas de trabajo analíticas de gran volumen y se integra de forma nativa con herramientas como Spark o Databricks.

**3. ¿Qué es un Linked Service en Azure Data Factory?**

Es la definición de conexión hacia un sistema externo (por ejemplo, un Storage Account o una base de datos SQL): contiene la información de cómo autenticarse y llegar a ese recurso. No mueve datos por sí solo, solo habilita la conexión que luego usan los Datasets.

**4. ¿Qué es un Dataset en Azure Data Factory?**

Es la representación de una estructura de datos específica dentro de un Linked Service — por ejemplo, un archivo CSV puntual en el Storage o una tabla puntual en SQL Database. Define el formato y la ubicación exacta de los datos con los que trabajará una actividad.

**5. ¿Cuál es la diferencia entre un pipeline y una activity en Azure Data Factory?**

Un pipeline es el proceso completo que orquesta el flujo de trabajo. Una activity es un paso individual dentro de ese pipeline (por ejemplo, una operación Copy Data). Un pipeline puede contener una o varias activities, encadenadas o en paralelo.

**6. ¿Cómo protegerías las credenciales de Azure SQL Database?**

La opción más segura es usar **Managed Identity**, que le asigna a Data Factory una identidad propia dentro de Azure Active Directory, eliminando la necesidad de guardar usuario y contraseña en el Linked Service. Como alternativa, se puede almacenar el secreto en **Azure Key Vault** y hacer que Data Factory lo consulte desde ahí en tiempo de ejecución, en vez de tenerlo hardcodeado en la configuración.

**7. ¿Qué es Managed Identity?**

Es una identidad que Azure asigna automáticamente a un recurso (en este caso, Data Factory), permitiéndole autenticarse contra otros servicios de Azure (como SQL Database o Storage) sin necesidad de gestionar ni almacenar contraseñas o claves manualmente. Los permisos se otorgan directamente a esa identidad sobre el recurso destino.

**8. ¿Cómo monitorearías una ejecución fallida en Azure Data Factory?**

Desde la pestaña **Monitor** de ADF Studio, donde se puede ver el historial de ejecuciones de cada pipeline, su estado (Succeeded/Failed) y el detalle del error de cada activity individual. Adicionalmente, se pueden configurar alertas con Azure Monitor para recibir una notificación automática (correo, Teams, etc.) cuando un pipeline falla, sin depender de revisión manual.

**9. ¿Qué harías si este proceso tuviera que correr todos los días?**

Se agregaría un **Trigger** de tipo Schedule al pipeline, configurado con la frecuencia deseada (por ejemplo, diaria a una hora específica), para que Data Factory lo ejecute automáticamente sin intervención manual.

**10. ¿Cómo evitarías reprocesar archivos que ya fueron cargados?**

Algunas estrategias: mover los archivos ya procesados de `raw/` a `processed/` una vez cargados exitosamente, de modo que el pipeline solo lea archivos pendientes; usar un mecanismo de **watermark** (registrar la fecha del último archivo procesado y solo tomar los más recientes); o mantener una tabla de control que registre qué archivos ya fueron cargados exitosamente antes de volver a procesarlos.

---

## Mejoras para un entorno de producción

- Usar **Managed Identity** en lugar de credenciales SQL en los Linked Services.
- Implementar **Private Endpoints** para que el tráfico entre Data Factory, Storage y SQL Database no salga a internet público.
- Agregar **triggers automáticos** (schedule o event-based) en vez de ejecución manual.
- Incorporar **manejo de errores y reintentos** en las activities del pipeline.
- Versionar y parametrizar el pipeline para soportar múltiples fuentes/tablas sin duplicar lógica.
- Agregar validación de calidad de datos antes de la carga (schema validation, conteo de filas esperado, nulos inesperados).
