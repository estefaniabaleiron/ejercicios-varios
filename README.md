# Documentación Técnica de Transformación de Datos y ETL

**Autora:** Estefanía Baleiron  

---

## 1. Transformaciones Realizadas y Secuencia de Pasos

Para garantizar la calidad, la integridad y un rendimiento óptimo en el modelo analítico, el proceso de Extracción, Transformación y Carga (ETL) se ejecutó en la siguiente secuencia lógica:

1. **Ingesta y Creación de la Tabla Origen (`base`):**  
   Se importó el conjunto de datos crudos creando la consulta staging `base`, la cual actúa como fuente centralizada (*Single Source of Truth*).
2. **Promoción de Encabezados:**  
   Se promovió la primera fila como encabezados de columna para estandarizar las denominaciones de las variables.
3. **Limpieza de Espacios y Formato de Texto:**  
   Se aplicaron las funciones de `Limpiar` (*Trim*) y `Poner en mayúsculas cada palabra` en campos de texto (`NOM_CLI`, `CIUDAD`) para eliminar caracteres invisibles, espacios sobrantes y unificar criterios de entrada.
4. **Detección y Tratamiento de Calidad de Datos (Duplicados y Nulos):**  
   Se utilizaron las herramientas de inspección de calidad de columna (*Column Quality*) para corregir anomalías en claves primarias e importes.
5. **Tipado Explícito de Datos:**  
   Se asignaron manualmente los tipos de datos adecuados para cada columna, anulando la detección automática para evitar conversiones imprevistas.
6. **Separación Dimensional (Modelado en Estrella):**  
   Se crearon las consultas referenciadas `D_CLIENTES` y `F_VENTAS` a partir de la tabla `base`.
7. **Selección de Columnas y Depuración de Redundancias:**  
   Se mantuvieron únicamente los atributos relevantes en cada tabla (`D_CLIENTES` con atributos cualitativos; `F_VENTAS` con métricas e identificadores).
8. **Optimización del Modelo:**  
   Se deshabilitó la opción **"Habilitar carga"** en la consulta origen `base` para evitar duplicar el consumo de memoria en el motor VertiPaq.

---

## 2. Justificación Técnica de los Tipos de Datos Elegidos

La asignación explícita de tipos de datos es fundamental para garantizar la compresión de almacenamiento en memoria y prevenir fallos en la ejecución de medidas DAX:

| Campo / Variable | Tipo de Dato Asignado | Justificación Técnica |
| :--- | :--- | :--- |
| **`COD_CLI` / `COD_PROD`** | **Texto (`String`)** | Aunque utilicen caracteres numéricos, funcionan como identificadores categóricos. Definirlos como texto evita que el motor intente realizar agregaciones matemáticas (sumas o promedios) y optimiza el indexado de relaciones. |
| **`ID_VENTA`** | **Número entero (`Int64`)** | Clave primaria secuencial sin decimales. Utiliza un menor volumen de almacenamiento en memoria que un tipo texto y permite agrupamientos rápidos. |
| **`CANTIDAD`** | **Número entero (`Int64`)** | Representa unidades físicas discretas vendidas. No requiere precisión decimal. |
| **`PRECIO_UNITARIO` / `TOTAL_VENTA` / `DESCUENTO`** | **Número decimal / Moneda (`Decimal/Currency`)** | Variables monetarias y porcentuales. Garantizan precisión aritmética en cálculos de márgenes e ingresos sin pérdidas por redondeo. |
| **`FECHA_VENTA` / `FECHA_ALTA`** | **Fecha (`Date`)** | Se removió la componente de hora (`Time`) dado que el análisis comercial se consolida a nivel diario. Esto reduce la cardinalidad del campo y habilita el uso eficiente de funciones de *Time Intelligence* en DAX. |
| **`NOM_CLI` / `CIUDAD` / `SEGMENTO`** | **Texto (`String`)** | Variables cualitativas descriptivas necesarias para dimensiones de filtrado y segmentación (*Slicers*). |

---

## 3. Resolución de Valores Nulos y Duplicados

### A. Tratamiento de Duplicados
* **Diagnóstico:** Los registros duplicados en datos transaccionales provocan una sobreestimación en las métricas de ingresos y distorsionan los indicadores clave.
* **Solución y Criterio:**
  * En la tabla `D_CLIENTES`, se aplicó la acción `Quitar duplicados` sobre la columna `COD_CLI`.
  * **Justificación:** Un modelo dimensional requiere estricta unicidad en la clave primaria de las tablas de dimensión para posibilitar relaciones **1 a Muchos (1:N)** válidas hacia la tabla de hechos.
  * En `F_VENTAS`, se depuraron las filas íntegramente duplicadas para preservar únicamente transacciones reales únicas.

### B. Tratamiento de Valores Nulos
* **Diagnóstico:** La presencia de valores `null` en claves impide establecer relaciones entre tablas, mientras que en campos categóricos genera la etiqueta `(Blank)` en las visualizaciones.
* **Solución y Criterio:**
  * **En Claves Relacionales (`COD_CLI`):** Se eliminaron aquellas filas con código de cliente nulo, dado que una transacción huérfana no puede asignarse a un cliente ni ser auditada correctamente.
  * **En Atributos Categóricos (`CIUDAD` / `SEGMENTO`):** Se empleó la función `Reemplazar los valores` sustituyendo `null` por la etiqueta `'Sin Especificar'`.
  * **Justificación:** Permite conservar la totalidad del volumen de ventas en `F_VENTAS` sin perder facturación ni deformar la distribución en los gráficos.
  * **En Campos Numéricos:** Se imputaron valores mediante la regla de $0$ o cálculo derivado ($Total = Cantidad \times Precio$), asegurando la continuidad de las agregaciones (`SUM`, `AVERAGE`).

---

## 4. Criterio de Separación: Datos del Cliente vs. Datos de la Transacción

La estructuración del modelo sigue los principios del **Modelado Dimensional (Esquema en Estrella / Star Schema)** sustentado en la metodología de Ralph Kimball y las reglas de normalización (3NF):

### A. Criterio de Granularidad y Entidad
* **Dimensión Cliente (`D_CLIENTES`):**
  * **Contenido:** Atributos descriptivos e invariantes del cliente (`COD_CLI`, `NOM_CLI`, `CIUDAD`, `FECHA_ALTA`).
  * **Granularidad:** 1 fila por cada cliente único.
* **Tabla de Hechos (`F_VENTAS`):**
  * **Contenido:** Transacciones, claves foráneas y métricas cuantitativas que ocurren periódicamente (`ID_VENTA`, `FECHA_VENTA`, `COD_CLI`, `COD_PROD`, `CANTIDAD`, `TOTAL_VENTA`).
  * **Granularidad:** 1 fila por cada evento o línea de venta.

### B. Justificación Técnica
1. **Eliminación de Redundancia y Ahorro de Memoria:** Replicar datos del cliente (como nombre o ciudad) en miles de filas transaccionales genera una desnormalización ineficiente. Aislar esta información reduce drásticamente el peso del archivo `.pbix`.
2. **Compresión Eficiente (Engine VertiPaq):** Power BI utiliza almacenamiento columnar. Las tablas de dimensión con baja cardinalidad logran índices de compresión superiores al 90%.
3. **Mantenibilidad del Modelo:** Ante una modificación en los datos del cliente (ej. cambio de domicilio o razón social), solo se actualiza un registro en `D_CLIENTES`, manteniendo la consistencia histórica sin alterar la tabla de hechos.
