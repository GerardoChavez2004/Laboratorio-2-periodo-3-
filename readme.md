# Dashboard ejecutivo de operaciones — Clínica

Dashboard de una sola página en **Power BI Desktop** que analiza la actividad de una clínica entre **noviembre de 2025 y octubre de 2026**. Los datos se leen directamente desde una base **PostgreSQL** (`clinica`) alojada en **Neon**, y el reporte responde cuatro preguntas de negocio sobre servicios, sedes y segmentos de clientes.

![Dashboard ejecutivo](dashboard.png)

## Contenido

1. [Objetivo](#1-objetivo)
2. [Resultados principales](#2-resultados-principales)
3. [Fuente de datos y conexión](#3-fuente-de-datos-y-conexión)
4. [Modelo de datos](#4-modelo-de-datos)
5. [Medidas y columnas calculadas (DAX)](#5-medidas-y-columnas-calculadas-dax)
6. [Diseño del dashboard](#6-diseño-del-dashboard)
7. [Respuestas a las preguntas de análisis](#7-respuestas-a-las-preguntas-de-análisis)
8. [Notas metodológicas y limitaciones](#8-notas-metodológicas-y-limitaciones)

---

## 1. Objetivo

Construir un dashboard ejecutivo que responda:

1. Identificar los servicios que concentran la actividad registrada.
2. ¿Qué categorías y servicios tienen más operaciones completadas por mes?
3. ¿Cómo se distribuye el importe completado por sede?
4. ¿Qué segmentos concentran más operaciones por servicio?

## 2. Resultados principales

| Indicador | Valor |
|---|---|
| Importe completado | ≈ $12.31 mil |
| Operaciones completadas | 91 |
| Ticket promedio | $135.27 |
| Tasa de completadas | 91.0 % (91 de 100 operaciones) |

- **Pediatría y Ultrasonido** concentran la actividad: cada uno tiene aproximadamente el doble de operaciones que cualquiera de los otros cuatro servicios.
- **Norte** genera el **51.5 %** del importe completado.
- La actividad completada crece de 3 operaciones en noviembre de 2025 a un **pico de 12 en julio y 13 en agosto de 2026**, y baja a 7 en septiembre y octubre.
- El segmento **Ocasional** es mayoritario en cuatro de los seis servicios; **Nuevo** nunca es el segmento principal.

## 3. Fuente de datos y conexión

- **Motor:** PostgreSQL alojado en Neon, base de datos `clinica`.
- **Herramienta:** Power BI Desktop, conector nativo *Base de datos PostgreSQL*.
- **Modo de conectividad:** Importar, con autenticación básica y conexión cifrada (SSL).
- Las credenciales **no se incluyen** en este repositorio.

### Alternativas evaluadas

El requisito era conectar directamente a Databricks, Power BI o Data Studio (Looker Studio). Se eligió Power BI por lo siguiente:

- **Databricks Free Edition** restringe el acceso saliente a internet a un conjunto limitado de dominios, por lo que una conexión directa a Neon es poco viable.
- **DBeaver + exportación a CSV** funciona, pero obliga a mover los datos manualmente.
- **Power BI Desktop** incluye el proveedor de PostgreSQL y se conecta a Neon sin pasos adicionales.

## 4. Modelo de datos

Cinco tablas con esquema en estrella:

| Tabla | Descripción | Campos principales |
|---|---|---|
| `operaciones` | Una fila por operación (100 filas) | `operacion_id`, `cliente_id`, `sede_id`, `fecha`, `estado`, `canal`, `actualizado_en` |
| `detalle_operacion` | Líneas de servicio de cada operación | `detalle_id`, `operacion_id`, `producto_id`, `cantidad`, `precio_unitario`, `descuento` |
| `catalogo` | Servicios y categorías | `producto_id`, `nombre`, `categoria`, `precio_lista` |
| `clientes` | Clientes y segmento (30 clientes) | `cliente_id`, `nombre`, `segmento`, `fecha_registro` |
| `sedes` | Sedes (3) | `sede_id`, `nombre`, `ciudad` |

**Relaciones** (uno a muchos, filtro en una sola dirección):

- `clientes[cliente_id]` → `operaciones[cliente_id]`
- `sedes[sede_id]` → `operaciones[sede_id]`
- `operaciones[operacion_id]` → `detalle_operacion[operacion_id]`
- `catalogo[producto_id]` → `detalle_operacion[producto_id]`

**Dominios relevantes:**

- `estado`: `completada`, `pendiente`, `cancelada`.
- `canal`: Digital, Teléfono, Presencial.
- `segmento`: Nuevo, Ocasional, Frecuente.
- Servicios por categoría: Consulta (Consulta general, Pediatría), Diagnóstico (Laboratorio básico, Ultrasonido), Especialidad (Cardiología), Prevención (Vacunación).

## 5. Medidas y columnas calculadas (DAX)

### Medidas

```DAX
Operaciones =
DISTINCTCOUNT('public detalle_operacion'[operacion_id])

Operaciones completadas =
CALCULATE([Operaciones], 'public operaciones'[estado] = "completada")

Importe =
SUMX('public detalle_operacion',
     'public detalle_operacion'[cantidad] * 'public detalle_operacion'[precio_unitario]
     - 'public detalle_operacion'[descuento])

Importe completado =
CALCULATE([Importe], 'public operaciones'[estado] = "completada")

Ticket promedio =
DIVIDE([Importe completado], [Operaciones completadas])

Tasa de completadas =
DIVIDE(
    CALCULATE(COUNTROWS('public operaciones'), 'public operaciones'[estado] = "completada"),
    COUNTROWS('public operaciones')
)
```

> **Supuesto:** el campo `descuento` se interpreta como un monto restado al subtotal. En los datos revisados su valor es 0.

### Columnas calculadas (tabla `operaciones`)

```DAX
Mes = DATE(YEAR('public operaciones'[fecha]), MONTH('public operaciones'[fecha]), 1)

Mes aaaa-mm = FORMAT('public operaciones'[fecha], "yyyy-MM")
```

`Mes` alimenta el segmentador de fechas. `Mes aaaa-mm` es texto y se usa en la matriz para evitar la jerarquía automática de fechas de Power BI y mantener el orden cronológico.

## 6. Diseño del dashboard

Una sola página con tema oscuro, organizada de lo general a lo específico:

| Zona | Contenido |
|---|---|
| Encabezado | Título y periodo analizado |
| Filtros | Segmentador de `Mes` y segmentador de `canal` (afectan todos los visuales) |
| Tarjetas KPI | Importe completado, Operaciones completadas, Ticket promedio, Tasa de completadas |
| Fila de gráficos | Barras (servicios), dona (importe por sede), barras apiladas al 100 % (segmentos por servicio) |
| Matriz | Operaciones completadas por categoría, servicio y mes, con formato condicional |
| Nota al pie | Aclaración metodológica sobre el conteo de operaciones |

Criterios de diseño: un título por visual que enuncia la pregunta que responde, subtítulo con el hallazgo principal donde aplica, etiquetas de datos legibles sobre fondo oscuro y una nota metodológica visible.

## 7. Respuestas a las preguntas de análisis

### 1. ¿Qué servicios concentran la actividad registrada?

*Visual: gráfico de barras, medida `Operaciones` (todos los estados).*

**Pediatría (≈34) y Ultrasonido (≈33)** concentran cerca de la mitad de los registros por servicio. Consulta general, Vacunación, Cardiología y Laboratorio básico se ubican entre 16 y 17 operaciones cada uno.

### 2. ¿Qué categorías y servicios tienen más operaciones completadas por mes?

*Visual: matriz categoría > servicio × mes, medida `Operaciones completadas`.*

| Categoría | Total | Servicios |
|---|---|---|
| **Consulta** | 48 | Pediatría 33, Consulta general 15 |
| **Diagnóstico** | 43 | Ultrasonido 29, Laboratorio básico 14 |
| **Prevención** | 17 | Vacunación 17 |
| **Especialidad** | 15 | Cardiología 15 |

Operaciones completadas por mes:

| Mes | nov-25 | dic-25 | ene-26 | feb-26 | mar-26 | abr-26 | may-26 | jun-26 | jul-26 | ago-26 | sep-26 | oct-26 |
|---|---|---|---|---|---|---|---|---|---|---|---|---|
| Total | 3 | 5 | 6 | 7 | 7 | 7 | 8 | 9 | 12 | 13 | 7 | 7 |

- **Por categoría:** Consulta lidera en seis meses (nov, dic, abr, ago, sep, oct), Diagnóstico en cuatro (feb, mar, may, jun) y hay empate en enero y julio.
- **Por servicio:** Pediatría lidera o empata en el primer lugar en 10 de 12 meses; Ultrasonido la supera en febrero y marzo.
- **Tendencia:** crecimiento sostenido hasta el pico de julio–agosto de 2026 y una caída posterior a 7 operaciones mensuales.

### 3. ¿Cómo se distribuye el importe completado por sede?

*Visual: dona, medida `Importe completado`.*

| Sede | Importe completado | Participación |
|---|---|---|
| **Norte** | $6.34 mil | 51.51 % |
| **Sur** | $3.35 mil | 27.23 % |
| **Centro** | $2.62 mil | 21.26 % |

La sede Norte concentra más de la mitad del importe; Sur y Centro se reparten el resto.

### 4. ¿Qué segmentos concentran más operaciones por servicio?

*Visual: barras apiladas al 100 %, medida `Operaciones` por segmento de cliente.*

| Servicio | Frecuente | Nuevo | Ocasional | Segmento principal |
|---|---|---|---|---|
| Pediatría | 17.65 % | 32.35 % | 50.00 % | Ocasional |
| Ultrasonido | 21.21 % | 27.27 % | 51.52 % | Ocasional |
| Consulta general | 64.71 % | 29.41 % | ≈ 5.9 % | Frecuente |
| Vacunación | 29.41 % | 11.76 % | 58.82 % | Ocasional |
| Cardiología | 31.25 % | ≈ 6.3 % | 62.50 % | Ocasional |
| Laboratorio básico | 62.50 % | 31.25 % | ≈ 6.3 % | Frecuente |

- **Ocasional** domina en Pediatría, Ultrasonido, Vacunación y Cardiología.
- **Frecuente** domina en Consulta general y Laboratorio básico.
- **Nuevo** no es el segmento principal en ningún servicio; su máximo es 32.35 % en Pediatría.

## 8. Notas metodológicas y limitaciones

- **Una operación puede incluir varios servicios.** Por eso los conteos por servicio y categoría suman más que el total de operaciones: en la matriz, las categorías suman 123 líneas de servicio frente a 91 operaciones completadas.
- **Estados incluidos.** El gráfico de servicios y el de segmentos usan la medida `Operaciones` (todos los estados). La matriz, la dona y las tarjetas usan solo operaciones **completadas**. Por eso, por ejemplo, Pediatría aparece con ≈34 en las barras y con 33 en la matriz.
- **Importe.** Se calcula como `cantidad × precio_unitario − descuento` a nivel de línea de detalle.
- **Periodo.** Doce meses, de noviembre de 2025 a octubre de 2026.
- **Datos no completados.** Hay 9 operaciones pendientes o canceladas, excluidas de los indicadores de operaciones completadas e importe.