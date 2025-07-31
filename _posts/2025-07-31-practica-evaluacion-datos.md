---
layout: single
title: "Práctica: Particiones de Datos y Evaluación de Modelos de Distribución de Especies"
author: "M. Jimena G. Burgos"
date: 2025-07-31 14:30:00 -0500 # Hora de Mérida, Yucatán
categories: [practice]
permalink: /resources/practica-evaluacion-datos/ # Esta será la URL de tu post
toc: true # Activa la Tabla de Contenido
toc_label: "Contenidos de la Práctica" # Título de la TOC
toc_icon: "fa-solid fa-book" # Icono para la TOC
toc_sticky: true # Hace que la TOC se mantenga visible al hacer scroll
toc_levels: 1..3 # Muestra encabezados H1, H2, H3 en la TOC
author_profile: true # Muestra el perfil del autor configurado en _config.yml
read_time: true # Muestra el tiempo de lectura estimado
comments: false # Desactiva comentarios si no los quieres
share: true # Activa botones de compartir
show_date: true # Asegura que la fecha se muestre
---
# Introducción

En esta práctica utilizaremos los registros de la especie Leopardus
wiedii, la paquetería ENMEval 2.0 (Kass et. al 2025) y el algoritmo
MaxEnt (Phillips et al. 2006) para explorar diferentes métodos de
partición de datos en la modelación de distribución de especies. El
objetivo de esta práctica es comparar tipos de partición de datos,
evaluando cómo cada enfoque maneja la autocorrelación espacial entre los
datos de entrenamiento y validación, así como su efecto en las métricas.

# Calibración

## Carga de datos biológicos y ambientales

``` r
# Leer datos biológicos
datos <- read.csv("occs/OCCS_lw.csv")
occs <- datos[,c("long","lat")]

# Listar archivos .tif en la carpeta "envs" dentro del proyecto 
envs.files <- list.files(path = "envs", pattern = ".tif", full.names = TRUE)

# Crear el raster stack con las rutas completas
envs <- rast(envs.files)

# Graficamos para ver los registros
plot(envs[[3]])
points(occs, pch = 19)
```

![](/assets/images/datos-biologicos-1.png)

## Carga de capa de sesgo

``` r
### Cargamos una capa de sesgo para dirigir los puntos de background
biasfile = rast("biaslayer/biaslayer_HREFBiv.tif")
biaslayer <- biasfile
# Es necesario que no existan NAs en el raster
# Por lo que asignamos un valor de 0 a los NAs
values(biasfile)[is.na(values(biasfile))] <- 0
n.bg <- 1000 # este sera la cantidad de puntos de background 
bg <- xyFromCell(biasfile, 
                 sample(ncell(biasfile),
                        n.bg, prob=values(biasfile))) %>% as.data.frame()
colnames(bg) <- colnames(occs)
head(bg)
```

    ##        long      lat
    ## 1 -89.89583 21.22917
    ## 2 -89.81250 18.31250
    ## 3 -89.22917 18.68750
    ## 4 -87.97917 19.10417
    ## 5 -88.43750 19.68750
    ## 6 -88.10417 20.97917

``` r
# Graficamos los puntos de background sobre la capa de sesgo
plot(biaslayer, main = "Capa de sesgo")
points(bg, pch=3 , cex = 0.4, col="white")
```

![](/assets/images/capa-sesgo-1.png)

## Función pROC

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

## Particiones aleatorias

Los dos métodos siguientes no tienen en cuenta la autocorrelación
espacial entre los registros de validación y entrenamiento. Por lo que,
es importante considerar que la partición aleatoria puede llevar a la
sobreestimación del modelo si los registros de presencia de calibración
y evaluación están próximas entre sí, debido a que las localidades
utilizadas para evaluar el modelo no son independientes de las
utilizadas para calibrarlo.

### Jackknife (leave-one-out)

Principalmente, al trabajar con conjuntos de datos relativamente
pequeños (e.g. 25 registros de presencia), los usuarios pueden optar la
validación cruzada k-fold, en el que el número de bins (k) o grupos es
igual al número de registros de presencia (n) (Pearson et al., 2007;
Shcheglovitova y Anderson, 2013). Esto se conoce como partición
jackknife o “dejar uno fuera” (Hastie et al., 2009).

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
                          user.eval = proc) #Métrica adicional (pROC)
```

    ##   |                                                                              |                                                                      |   0%  |                                                                              |=======================                                               |  33%  |                                                                              |===============================================                       |  67%  |                                                                              |======================================================================| 100%

### Random k-fold

El método de «k-fold aleatorio» divide aleatoriamente los registros de
presencia en un número de (k) bins o grupos especificado por el usuario
(Hastie et al., 2009). Este método es ideal utilizarlo con conjuntos de
registros de presencia grandes.

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

## Particiones geográficamente estructuradas

Los métodos de partición son variaciones de las particiones
geográficamente estructuradas (Radosavljevic y Anderson, 2014).
Básicamente, estos métodos dividen los registros de presencia y
background en grupos de evaluación basados en reglas espaciales. El
objetivo es reducir la autocorrelación espacial entre los registros
utilizados en la validación y el entrenamiento, evitando inflar el
rendimiento del modelo, al menos para los conjuntos de datos resultantes
de un muestreo sesgado (Veloz 2009, Wenger y Olden 2012, Roberts et
al. 2017).

### Bloques

``` r
# El método de "bloque" particiona los datos según la mediana de la latitud 
# y longitud que dividen los registros de presencia en cuatro grupos espaciales de
# igual número o lo más próximos posible.

block <- get.block(occs, bg, orientation = "lat_lon")

# El objeto resultante es una lista de dos vectores que proporcionan la
# designación del bloque o bin para cada registro de presencia y background
table(block$occs.grp)
```

    ## 
    ##  1  2  3  4 
    ## 10 10 10  9

``` r
# La primera dirección divide los registros de presencia en dos grupos, y la 
# segunda divide cada uno de ellos en dos grupos adicionales, resultando en cuatro grupos.
# Tanto los registros de presencia como los de background se asignan a cada uno de 
# los cuatro bloques o bins, según su posición con respecto a estas líneas 

# bloques de registros de presencia
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

Estos generan cuadrículas como un tablero de ajedrez a lo largo de la
extensión del área de calibración y dividen los registros de presencia
en grupos según su ubicación en el tablero. A diferencia del método de
bloques, ambos métodos de tablero de ajedrez subdividen el área de
calibración equitativamente, aunque no garantizan el mismo número de
localidades de ocurrencia en cada bin o grupo.

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
