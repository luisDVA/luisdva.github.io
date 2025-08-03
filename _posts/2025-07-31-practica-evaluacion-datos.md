---
layout: single
title: "Partición de Datos y Evaluación de Modelos "
author: "M. Jimena G. Burgos"
date: 2025-07-31
categories: [practice]
permalink: /resources/practica-evaluacion-datos/ # Esta será la URL de tu post
toc: true # Activa la Tabla de Contenido
toc_label: "Contenidos de la Práctica" # Título de la TOC
toc_icon: "fa-solid fa-book" # Icono para la TOC
toc_sticky: false # Hace que la TOC se mantenga visible al hacer scroll
toc_levels: 1..3 # Muestra encabezados H1, H2, H3 en la TOC
author_profile: true # Muestra el perfil del autor configurado en _config.yml
read_time: true # Muestra el tiempo de lectura estimado
comments: false # Desactiva comentarios si no los quieres
share: true # Activa botones de compartir
show_date: true # Asegura que la fecha se muestre
---
# Introducción
En esta práctica utilizaremos los registros de presencia de la especie *Leopardus wiedii*, 
la paquetería *ENMEval* 2.0 (Kass *et al*., 2025) y el algoritmo
MaxEnt (Phillips *et al*., 2006) para explorar los diferentes métodos de
partición de datos (registros de presencia), utilizados en la modelación de nichos ecológicos.

# Descargamos la carpeta del proyecto
Primero, necesitamos descargar la carpeta del proyecto [aqui](https://drive.google.com/drive/folders/1rlSLD5f-CssSEgztnzTlzS7Z_KUywj8z?usp=drive_link)

## Cargamos los paquetes de R
Para esta práctica, utilizaremos diferentes paquetes de R. Vamos a cargarlos, si no los tienes instalados, utiliza la función
*install.packages()* o visita la página de ayuda del paquete.
``` r
library(ENMeval) # Calibración y evaluación de modelos de nicho ecológico.
library(terra)   # Manejo y análisis de datos espaciales (raster/vector).
library(dplyr)   # Manipulación y transformación de datos tabulares (filtrar, seleccionar).
library(ggplot2) # Creación de gráficos
library(kuenm)   # Calibración y evaluación de modelos de nicho ecológico.
library(dismo) # Implementación de algoritmos para modelos de distribución de especies (SDM).
library(ecospat) # Used to estimate CBI
```

## Cargamos los datos de presencia y capas ambientales
Antes de iniciar con la modelación, necesitamos cargar nuestros datos de presencia, y variables ambientales
``` r
# Leer datos biológicos
occs <- read.csv("occs/occs_lw.csv")

# Listar archivos .tif en la carpeta "envs" dentro del proyecto 
envs.files <- list.files(path = "envs", pattern = ".tif", full.names = TRUE)

# Crear el raster stack con las rutas completas
envs <- raster::stack(envs.files)

# Graficamos para ver los registros
plot(envs[[3]])
points(occs, pch = 19)
```
![](/assets/images/datos-biologicos-1.png)

## Generamos los puntos de fondo (background)
El algoritmo maxent trabaja con datos de presencia, y puntos de fondo (background). Especificamente, 
el paquete *ENMeval* necesita un objeto que contenga las coordenadas geográficas de dichos puntos de trasfondo.
``` r
n.bg <- 1000 # número de puntos de bg
vals_capa <- values(envs[[3]]) # Obtener los valores del raster
celdas_validas <- which(!is.na(vals_capa)) # Encuentra las celdas que NO son NA
bg <- xyFromCell(envs[[3]], sample(celdas_validas, n.bg, prob = vals_capa[celdas_validas], replace = TRUE)) %>% as.data.frame()

# graficar los puntos de bg
plot(envs[[3]], main = "")
points(bg, pch = 3 , cex = 0.4, col = "white")
colnames(bg) <- colnames(occs)
```
![](/assets/images/bg.png)

## Función pROC
En el paquete *ENMeval*, es posible cargar nuestras funciones personalizadas. Aqui podemos definir una del paquete *kuenm*
``` r
# Definimos una función personalizada para implementar la pROC
proc <- function(vars) {
  proc <- kuenm::kuenm_proc(vars$occs.val.pred, c(vars$bg.train.pred, vars$bg.val.pred))
  out <- data.frame(proc_auc_ratio = proc$pROC_summary[1], 
                    proc_pval = proc$pROC_summary[2], row.names = NULL)
  return(out)
}
```

# Particiones
*ENMeval* ofrece diversas formas de particionar las localidades de presencia y 
de fondo en grupos para el entrenamiento y la validación de los modelos. 
Los usuarios **deben considerar cuidadosamente los objetivos de su estudio y la influencia 
del sesgo espacial al decidir el método de partición de datos**.

## Particiones aleatorias
Los dos métodos siguientes no tienen en cuenta explicitamente la estructura 
geográfica al momento de particionar los registros de validación y entrenamiento.

### Random k-fold
El método de «k-fold aleatorio» divide aleatoriamente los registros de
presencia en un número de (k) bins o grupos, especificado por el usuario
(Hastie *et al*., 2009). Este método es ideal utilizarlo con conjuntos de
registros de presencia grandes. En la calibración, se utilizarán (k)-1
para entrenar el modelo y se usará el grupo no incluido para validar. 
Esto se repetirá n veces para el número de k.

``` r
rand <- get.randomkfold(occs, bg, k = 5) #En este caso elegimos 5
evalplot.grps(pts = occs, pts.grp = rand$occs.grp, envs = envs)
```

![](/assets/images/randomkfold-particion-1.png)

``` r
# Corremos el modelo
e.mx_kfold <- ENMevaluate(occs = occs, 
                        envs = envs,
                        bg = bg, 
                        algorithm = 'maxnet', 
                        partitions = 'randomkfold', 
                        tune.args = list(fc = c("L","LQ","LQP"), rm = 1),
                        user.eval = proc)
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |=======================                                               |  33%  |                                                                              |===============================================                       |  67%  |                                                                              |======================================================================| 100%

### Jackknife (leave-one-out)
Es un tipo especial de k-fold donde el número de grupos (k) es igual al número total de registros.
Es útil al trabajar con conjuntos de datos relativamente
pequeños (e.g. 25 registros de presencia; Pearson *et al*., 2007;
Shcheglovitova y Anderson, 2013). Esto se conoce como partición
jackknife o “dejar uno fuera” (Hastie *et al*., 2009).

``` r
jack <- get.jackknife(occs, bg)
evalplot.grps(pts = occs, pts.grp = jack$occs.grp, envs = envs)
```

![](/assets/images/jackknife-particion-1.png)

``` r
# Corremos el modelo
e.mx_jack <- ENMevaluate(occs = occs, #Registros de presencia
                          envs = envs, #Capas ambientales
                          bg = bg, #Puntos de background
                          algorithm = 'maxnet', #Algoritmo
                          partitions = 'jackknife', #Método de partición
                          tune.args = list(fc = c("L","LQ","LQP"), rm = 1),
                          user.eval = proc) # Métrica adicional (pROC)
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |=======================                                               |  33%  |                                                                              |===============================================                       |  67%  |                                                                              |======================================================================| 100%

## Particiones geográficamente estructuradas
Estos métodos dividen los registros de presencia y
background en grupos de evaluación basados en reglas espaciales. El
objetivo es reducir la autocorrelación espacial entre los registros
utilizados en la validación y el entrenamiento
(Veloz 2009, Wenger y Olden 2012, Roberts et al. 2017).

### Bloques
El método de "bloque" particiona los datos según la mediana de la latitud 
y longitud que dividen los registros de presencia en cuatro grupos espaciales de
igual número o lo más próximos posible.
La primera dirección divide los registros de presencia en dos grupos, y la 
segunda divide cada uno de ellos en dos grupos adicionales, resultando en cuatro grupos.
Tanto los registros de presencia como los de background se asignan a cada uno de 
los cuatro bloques o bins, según su posición con respecto a estas líneas.
``` r
block <- get.block(occs, bg, orientation = "lat_lon")

```
El objeto resultante es una lista de dos vectores que proporcionan la
designación del bloque o bin para cada registro de presencia y background
``` r
table(block$occs.grp)
```
    ## 
    ##  1  2  3  4 
    ## 10 10 10  9

``` r
evalplot.grps(pts = occs, pts.grp = block$occs.grp, envs = envs) + 
  ggplot2::ggtitle("Spatial block partitions: occurrences")
```

![](/assets/images/bloques-visualizacion-1.png)

``` r
# bloques de background
evalplot.grps(pts = bg, pts.grp = block$bg.grp, envs = envs) + 
  ggplot2::ggtitle("Spatial block partitions: background")
```

![](/assets/images/bloques-visualizacion-2.png)

``` r
# Corremos el modelo
e.mx_block <- ENMevaluate(occs = occs, 
                     envs = envs,
                     bg = bg, 
                     algorithm = 'maxnet', 
                     partitions = 'block', 
                     tune.args = list(fc = c("L","LQ","LQP"), rm = 1),
                     user.eval = proc)
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |=======================                                               |  33%  |                                                                              |===============================================                       |  67%  |                                                                              |======================================================================| 100%

### Checkerboard básico
Este método genera cuadrículas como un tablero de ajedrez a lo largo de la
extensión del área de calibración y dividen los registros de presencia
en grupos según su ubicación en el tablero. A diferencia del método de
bloques, ambos métodos de tablero de ajedrez subdividen el área de
calibración equitativamente, aunque no garantizan el mismo número de
localidades de presencia en cada bin o grupo.
El método de tablero de ajedrez básico divide los puntos en k = 2 grupos
utilizando un patrón de tablero de ajedrez simple.

``` r
cb1 <- get.checkerboard(occs, envs, bg,
                         aggregation.factor=30) #puedes cambiar
                          #este parámetro para crear "cajas" más grandes.
# visualizamos presencias
evalplot.grps(pts = occs, pts.grp = cb1$occs.grp, envs = envs)
```

![](/assets/images/checkerboard1-particion-1.png)

``` r
# visualizamos background
cb1$bg.grp <- cb1$bg.grp[1:nrow(bg)]
evalplot.grps(pts = bg, pts.grp = cb1$bg.grp, envs = envs)
```

![](/assets/images/checkerboard1-particion-2.png)

``` r
# Corremos el modelo
e.mx_check1 <- ENMevaluate(occs = occs, 
                         envs = envs,
                         bg = bg, 
                         algorithm = 'maxnet', 
                         tune.args = list(fc = c("L","LQ","LQP"), rm = 1),
                         partitions = "user",
                         user.grp = cb1,
                         user.eval = proc)
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |=======================                                               |  33%  |                                                                              |===============================================                       |  67%  |                                                                              |======================================================================| 100%

### Checkerboard jerárquico
El método de tablero de Checkerboard jerárquico genera k = 4 grupos
espaciales mediante un enfoque de dos escalas anidadas. Este método
aplica dos factores de agregación secuenciales: primero crea una grilla
gruesa que divide el área de estudio, y luego subdivide cada celda de
esta grilla en celdas más finas, generando un patrón espacial
jerárquico.

A diferencia del checkerboard básico que produce solo 2 grupos, este
método ofrece una partición más granular con 4 grupos, manteniendo la
separación geográfica pero proporcionando mayor flexibilidad para
conjuntos de datos grandes o áreas de estudio complejas. Los registros
de presencia y background se asignan a cada grupo según su ubicación en
el patrón jerárquico resultante.

``` r
cb2 <- get.checkerboard(occs, envs, bg, aggregation.factor=c(10,10))
# visualizamos presencias
evalplot.grps(pts = occs, pts.grp = cb2$occs.grp, envs = envs)
```

![](/assets/images/checkerboard2-particion-1.png)

``` r
# visualizamos background
cb2$bg.grp <- cb2$bg.grp[1:nrow(bg)]
evalplot.grps(pts = bg, pts.grp = cb2$bg.grp, envs = envs)
```

![](/assets/images/checkerboard2-particion-2.png)

``` r
# Corremos el modelo
e.mx_check2 <- ENMevaluate(occs = occs, 
                          envs = envs,
                          bg = bg, 
                          algorithm = 'maxnet', 
                          tune.args = list(fc = c("L","LQ","LQP"), rm = 1),
                          partitions = "user",
                          user.grp = cb2,
                          user.eval = proc)
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |=======================                                               |  33%  |                                                                              |===============================================                       |  67%  |                                                                              |======================================================================| 100%

# Métricas
## Resultados de evaluación

``` r
# Podemos ver los resultados de cualquiera de nuestros modelos
# vemos el de jackknife
e.mx_check1@results
```

    ##    fc rm   tune.args auc.train cbi.train auc.diff.avg auc.diff.sd auc.val.avg
    ## 1   L  1   fc.L_rm.1 0.6882654     0.856   0.05944181  0.06494259   0.6557393
    ## 2  LQ  1  fc.LQ_rm.1 0.6859250     0.889   0.05831112  0.06185649   0.6574916
    ## 3 LQP  1 fc.LQP_rm.1 0.6958308     0.853   0.05533106  0.07734586   0.6531214
    ##   auc.val.sd cbi.val.avg  cbi.val.sd or.10p.avg or.10p.sd or.mtp.avg  or.mtp.sd
    ## 1 0.05012123       0.537 0.008485281  0.1818182 0.2571297 0.04545455 0.06428243
    ## 2 0.04913016       0.505 0.062225397  0.2045455 0.2892710 0.06818182 0.09642365
    ## 3 0.04954809       0.346 0.244658946  0.1818182 0.2571297 0.04545455 0.06428243
    ##   proc_auc_ratio.avg proc_auc_ratio.sd proc_pval.avg proc_pval.sd     AICc
    ## 1           1.160328        0.07275590             0            0 897.6468
    ## 2           1.176942        0.05242395             0            0 900.4820
    ## 3           1.156025        0.07684133             0            0 899.8585
    ##   delta.AICc     w.AIC ncoef
    ## 1   0.000000 0.6356401     4
    ## 2   2.835218 0.1540109     5
    ## 3   2.211729 0.2103490     5

## Compilación de resultados
``` r
# Asignamos los resultados de la evaluacion en un dataframe
res_block <- eval.results(e.mx_block)
res_check1 <- eval.results(e.mx_check1)
res_check2 <- eval.results(e.mx_check2)
res_jack <- eval.results(e.mx_jack)
res_kfold <- eval.results(e.mx_kfold)
```

## Comparación y visualización
``` r
# Agregamos una columna para distinguir los tipos de partición
res_block["Tipo_particion"] <- "Bloque"
res_check1["Tipo_particion"] <- "CheckerboardBasico"
res_check2["Tipo_particion"] <- "CheckerboardJerarquico"
res_jack["Tipo_particion"] <- "Jackknife"
res_kfold["Tipo_particion"] <- "Random-k-fold"

# Unimos los dataframes
data_results <- rbind(res_block, res_check1, res_check2, res_jack, res_kfold)

data_long <- data_results %>%
  # Seleccionar las columnas necesarias
  select(Tipo_particion, or.10p.avg, auc.val.avg ) %>%
  # Convertir a formato largo directamente (sin agrupar ni promediar)
  pivot_longer(
    cols = c(or.10p.avg, auc.val.avg),
    names_to = "Metrica",
    values_to = "Valor"
  ) %>%
  # Renombrar las métricas para que se vean mejor en la gráfica
  mutate(
    Metrica = case_when(
      Metrica == "or.10p.avg" ~ "OR", 
      Metrica == "auc.val.avg" ~ "AUC",
      TRUE ~ Metrica
    )
  )
```

``` r
# Visualizamos las métricas de los distintos tipos de partición
ggplot(data_long, aes(x = Tipo_particion, y = Valor, color = Tipo_particion, fill = Tipo_particion)) +
  geom_boxplot(alpha = 0.7) +
  facet_wrap(~Metrica, scales = "free_y") +
  theme_minimal() +
  labs(
       x = "Tipo de partición",
       y = "Valor") +
  theme(axis.text.x = element_text(angle = 45, hjust = 1),
        legend.position = "none")
```

![](/assets/images/grafico-metricas-1.png)

# Visualización de modelos
## Selección de modelos óptimos
``` r
# Utilizamos un protocolo secuencial para elegir un modelo de cada tipo de partición:
# la menor tasa de omisión y el máximo AUC
opt.seq_block <- res_block %>% 
  filter(or.10p.avg == min(or.10p.avg)) %>% 
  filter(auc.val.avg == max(auc.val.avg))

opt.seq_check1 <- res_check1 %>% 
  filter(or.10p.avg == min(or.10p.avg)) %>% 
  filter(auc.val.avg== max(auc.val.avg))

opt.seq_check2 <- res_check2 %>% 
  filter(or.10p.avg == min(or.10p.avg)) %>% 
  filter(auc.val.avg == max(auc.val.avg))

opt.seq_jack <- res_jack %>% 
  filter(or.10p.avg == min(or.10p.avg)) %>% 
  filter(auc.val.avg == max(auc.val.avg))

opt.seq_kfold <- res_kfold %>% 
  filter(or.10p.avg == min(or.10p.avg)) %>% 
  filter(auc.val.avg == max(auc.val.avg))
```

## Extracción de modelos
``` r
# Ahora seleccionamos solo ese modelo de nuestro conjunto de modelos
mod.seq_block <- eval.models(e.mx_block)[[opt.seq_block$tune.args]]
mod.seq_check1 <- eval.models(e.mx_check1)[[opt.seq_check1$tune.args]]
mod.seq_check2 <- eval.models(e.mx_check2)[[opt.seq_check2$tune.args]]
mod.seq_jack <- eval.models(e.mx_jack)[[opt.seq_jack$tune.args]]
mod.seq_kfold <- eval.models(e.mx_kfold)[[opt.seq_kfold$tune.args]]
```

## Predicciones y mapas
``` r
# Finalmente seleccionamos una predicción
pred.seq_block <- eval.predictions(e.mx_block)[[as.character(opt.seq_block$tune.args)]]
pred.seq_check1 <- eval.predictions(e.mx_check1)[[as.character(opt.seq_check1$tune.args)]]
pred.seq_check2 <- eval.predictions(e.mx_check2)[[as.character(opt.seq_check2$tune.args)]]
pred.seq_jack <- eval.predictions(e.mx_jack)[[as.character(opt.seq_jack$tune.args)]]
pred.seq_kfold <- eval.predictions(e.mx_kfold)[[as.character(opt.seq_kfold$tune.args)]]
```

``` r
# Graficamos los modelos seleccionados de cada tipo de partición
par(mfrow=c(1, 3))
plot(pred.seq_block, main="Block")
plot(pred.seq_kfold, main="Random-k-fold")
plot(pred.seq_check1, main="Checkerboard básico")
```

![](/assets/images/mapas-finales-1.png)

``` r
par(mfrow=c(1, 2))
plot(pred.seq_check2, main="Checkerboard jerárquico")
plot(pred.seq_jack, main="Jackknife")
```

![](/assets/images/mapas-finales-2.png)
