---
title: Viaje redondo para hojas de cálculo con forgts y gtxlsx
excerpt: "Planillas con formato"
category: rstats
tags: 
  - gt
  - spreadsheets
  - xlsx
  - openxlsx2
header: 
  overlay_image: /assets/images/featurepls.png
  overlay_filter: "0.4"
---

En distintas presentaciones o publicaciones en esta misma página ya he mencionado a [forgts](https://luisdva.github.io/forgts/){:target="_blank"}, un paquete que lee una hoja de cálculo con formato y produce una tabla `gt` con el mismo formato para celdas, bordes, y texto. La idea es mostras información de hojas de cálculo con formato en presentaciones y reportes en distintos formatos (PDF, HTML) sin tener que estar pegando capturas de pantalla.

Hace poco [Jan Marvin Garbuszus](https://github.com/JanMarvin){:target="_blank"} publicó [gtxlsx](https://github.com/JanMarvin/gtxlsx){:target="_blank"}, un paquete que hace lo contrario: toma una tabla de clase `gt` y la exporta a un libro de `openxlsx2` (es decir, una hoja de cálculo de Excel), conservando el formato. Esto significa que ahora podemos hacer el viaje redondo con una hoja de cálculo con formato: primero leerla en R con `forgts`, (opcionalmente) hacer algo con el objeto `gt`, y luego escribirla de vuelta con `gtxlsx`.

## forgts

<figure>
    <a href="/assets/images/forgtslogo.png"><img src="/assets/images/forgtslogo.png" style="height:138px"></a>
</figure>

`forgts` lee los datos de una hoja de cálculo, importa el formato de texto y celdas (tipo y color de fuente, bordes de celda, rellenos, etc.) y lo aplica a una versión `gt` de los datos.

`gtxlsx` toma una tabla `gt` y la coloca como celdas en un libro de `openxlsx2`, conservando el encabezado, grupos de filas,y otros componentes de estilo.

## Ejemplo

Podemos usar el archivo `rodentsheet.xlsx` que ya viene incluido con `forgts`. Así se ve:

<figure>
    <a href="/assets/images/spsheetog.png"><img src="/assets/images/spsheetog.png"></a>
</figure>

Importamos con `forgts`:

{% highlight r %}
library(forgts)
library(gt)
library(openxlsx2)
library(gtxlsx)

example_spreadsheet <- system.file("extdata/rodentsheet.xlsx", package = "forgts")
gt_tbl <- forgts(example_spreadsheet)
gt_tbl
{% endhighlight %}

Esto nos da un objeto `gt` con el mismo formato que la hoja de cálculo original:

<figure>
    <a href="/assets/images/rdout.png"><img src="/assets/images/rdout.png"></a>
</figure>

Ahora podemos escribirlo de vuelta a una hoja de cálculo con `gtxlsx`:

{% highlight r %}
wb <- wb_workbook()$add_worksheet(grid_lines = FALSE)
wb <- wb_add_gt(wb, gt_tbl)

# guardarlo
wb_save(wb, "rodentsheet_roundtrip.xlsx")

# o abrirlo interactivamente
if (interactive()) wb$open()
{% endhighlight %}

El archivo resultante tiene los mismos datos y formato que el original: rellenos de celda, estilos de fuente, bordes y todo.

## ¿A quién le importa?

Las hojas de cálculo no van a desaparecer. Poder leerlas en R, trabajar con ellas como tablas presentables, y escribirlas de vuelta significa que podemos quedarnos en R durante flujos de trabajo que empiezan y terminan en Excel sin perder el formato que hace que los datos sean interpretables, aunque esto implique prácticas cuestionables de organización de datos.



---

**Lecturas interesantes**:  
- [Documentación de forgts](https://luisdva.github.io/forgts/){:target="_blank"}  
- [Repositorio de gtxlsx](https://github.com/JanMarvin/gtxlsx){:target="_blank"}  
- [openxlsx2](https://janmarvin.github.io/ox2/){:target="_blank"}
