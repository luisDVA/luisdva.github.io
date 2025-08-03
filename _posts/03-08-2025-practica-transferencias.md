# Transferencias (KUENM, MOP, procedimientos de transferencia))

En esta práctica exploraremos los diferentes procedimientos de
transferencia (Truncación, Extrapolación y Clamping). El objetivo es ver
como cambian las métricas de evaluación y las predicciones en cada uno
de los procedimientos. También exploraremos la métrica MOP (Mobility
Oriented-Parity), la cual permite caracterizar los niveles de
disimilitud entre un conjunto de condiciones de referencia y otro
conjunto de condiciones de interés. Para ello utilizaremos a la especie
*Ctenosaura similis* como sistema de estudio. Es nativa de México,
Nicaragua, Guatemala, El Salvador, Honduras, Belize, Costa Rica, Panama,
pero introducida en Florida (Uetz *et al*., 2025).

# PRIMERO CARGAREMOS LOS PAQUETES

``` r
library(tidyverse) # Manipulación, transformación de datos.
library(sf) # Trabajar con datos vectoriales.
library(rnaturalearth) # Descargar datos geoespaciales.
# library(rnaturalearthdata) # Proveer datos para rnaturalearth.
library(kuenm) # Herramientas para modelado de nicho ecológico.
library(ggplot2) # Creación de gráficos
library(terra)
library(ntbox)
```

# EXPLORAMOS LOS DATOS

## REGISTROS DE PRESENCIA

``` r
# Cargar registros de presencia
occs_train <- read_csv("occ_train.csv")
occs_test <- read_csv("occ_test.csv")
occs_ind <- read_csv("occ_ind.csv")

# Visualizar los registros
ggplot() +
  geom_sf(data = ne_countries(scale = "medium", returnclass = "sf"),
          fill = "lightgray", color = "black") +
  geom_point(data = occs_train, aes(x = lon, y = lat, colour = "Entrenamiento"),
             alpha = 0.5, size = 0.4) +
  geom_point(data = occs_test, aes(x = lon, y = lat, colour = "Validacion"),
             alpha = 0.5, size = 0.4) +
  geom_point(data = occs_ind, aes(x = lon, y = lat, colour = "Invasor"),
             alpha = 0.5, size = 0.4) +
  labs(x = "Longitud", y = "Latitud",
       title = expression("Registros de " * italic("C. similis"))) +
  coord_sf(xlim = c(-110, -75), ylim = c(5, 35)) + theme_void() +
  scale_color_manual(name = "Distribución", 
                     values = c("Entrenamiento" = "blue", "Validacion" = "red", 
                                "Invasor" = "green"),
                     labels = c("Entrenamiento" = "Entrenamiento", 
                                "Validacion" = "Validación",
                                "Invasor" = "Invasor"),
                     breaks = c("Entrenamiento", "Validacion", "Invasor"))
```

![](prac_2_files/figure-markdown_github/unnamed-chunk-2-1.png)

## VARIABLES AMBIENTALES

### ZONA NATIVA

``` r
# cargamos las variables
vars_nat_stack <- rast(list.files(path = "M_var/set_1", pattern = "\\.asc$", 
                                  full.names = TRUE))
plot(vars_nat_stack)

# BIO4 = Temperature Seasonality
# BIO5 = Max Temperature of Warmest Month
# BIO12 = Annual Precipitation
# BIO15 = Precipitation Seasonality
```

### ZONA DE INVASIÓN

``` r
# cargamos las variables
vars_inv_stack <- rast(list.files(path = "G_var/set_1/set_1", pattern = "\\.asc$", 
                                  full.names = TRUE))
plot(vars_inv_stack)
```

# CORREMOS LOS MODELOS

En esta sección, vamos a construir modelos con todas las combinaciones
posibles entre tipos de respuesta, multiplicadores de regularización y
sets de variables. Pero primero necesitamos definir los argumentos.

## Calibracion del modelo

``` r
# Nombre del archivo con todos los registros de presencia nativos
occ_joint <- "occ_joint.csv"
# Nombre del archivo que usaremos para entrenar nuestros modelo candidatos
occ_tra <- "occ_train.csv"
# Este es el nombre de la carpeta que contiene los sets con las variables de calibracion
M_var_dir <- "M_var"
# Nombre que tendrá el archivo que contiene los codigos para crear los modelos candidatos
batch_cal <- "Candidate_models"
# Nombre de la carpeta donde se guardaran nuestros modelos candidatos (la carpeta se genera automaticamente al correr la funcion)
out_dir <- "Candidate_Models"
# Aqui asignamos los multiplicadores de regularización, podemos crear un vector
reg_mult <- seq(2, 4, 1)
# Aqui los tipos de respuesta, pueden revisar los argumentos con la ayuda de la función utilizando help(kuenm) o ??kuenm
f_clas <- c("l", "q")
# Esta es la ruta donde esta nuestro archivo de Maxent
maxent_path <- "C:/Users/tsunnyk/Documents/models"
# Agregar argumentos adicionales
# args <- "maximumbackground=10000 biasfile=D:\\YOURPATH\\bias_layer_01.asc"
# Mantener en FALSE si desea correr en la terminal. Si se desea correr el analisis dentro de la consola de R cambiar a TRUE (consume más memoria RAM)
wait <- FALSE
# Mantener en TRUE para correr el analisis
run <- TRUE
# Función para crear los modelos candidatos
kuenm_cal(occ.joint = occ_joint, occ.tra = occ_tra, M.var.dir = M_var_dir, 
          batch = batch_cal, out.dir = out_dir, reg.mult = reg_mult, f.clas = f_clas, 
          maxent.path = maxent_path, wait = wait, run = run)
```

## Evaluación y selección de los mejores modelos

``` r
# Archivo que usaremos para evaluar nuestros modelos candidatos
occ_test <- "occ_test.csv"
# carpeta donde se guardaran los resultados de la evaluación (la carpeta se crea automaticamente)
out_eval <- "Calibration_results"
# Este es el umbral de omisión permitida (es definido por el usuario)
threshold <- 5
# Porcentaje de puntos que usaremos para el calculo de la ROC parcial
rand_percent <- 50
# numero de iteraciones
iterations <- 100
# Si se usa FALSE se eliminaran los modelos candidatos despues de la evaluación
kept <- TRUE
# Criterio de selección del modelo (recomendado seleccionar con base en OR y AICc) 
selection <- "OR_AICc"
# Función para realizar la evaluación
cal_eval <- kuenm_ceval(path = out_dir, occ.joint = occ_joint, occ.tra = occ_tra, 
                        occ.test = occ_test, batch = batch_cal,
                        out.eval = out_eval, threshold = threshold, rand.percent = rand_percent,
                        iterations = iterations, kept = kept, selection = selection)    
```

## ¿Cuantos mejores modelos obtuvimos?

``` r
# cargamos los resultados de la evaluación
best_mod <- read_csv("Calibration_results/selected_models.csv")

view(best_mod)
print(best_mod)
print(best_mod$Model)
```

## construimos el modelo final (aquí transferimos)

``` r
# Nombre que tendrá el archivo batch (igual que en la calibración)
batch_fin <- "final_models"
# Nombre que tendrá la carpeta donde se guardarán el/los modelos finales
mod_dir <- "Final_Models"
# Número de replicar del modelo
rep_n <- 10
# tipo de replica que usaremos ("Bootstrap", "Crossvalidate" o "Subsample")
rep_type <- "Bootstrap"
# Hacer TRUE si se desea iniciar un proceso de jackknife (variables ambientales)
jackknife <- FALSE
# Formato de salida de los modelos ("raw", "logistic", "cloglog" o "cumulative")
out_format <- "cloglog"
# Hacer verdadero si se desea proyectar a otros tiempos o regiones geograficas (variables deben tener el mismo nombre que en "M_var")
project <- TRUE
# Directorio que contiene los diferentes sets donde se proyectaran los analisis
G_var_dir <- "G_var"
# Tipo de extrapolación a utilizar ("all", "ext_clam", "ext" y "no_ext")
ext_type <- "all"
# Hacer TRUE si se desea hacer un analisis MESS (riesgo de extrapolación)
write_mess <- FALSE
# Si se hace TRUE se mostraran las celdas donde se realizó clamping
write_clamp <- FALSE
# Mismo argumento que en la calibración, hacer TRUE se si desea correr en la consola de R
wait1 <- FALSE
# Si se pone FALSE, el modelo se debe construir de manera manual con el archivo batch
run1 <- TRUE

# Despues de tener los argumentos asignados procedemos a correr la función
kuenm_mod(occ.joint = occ_joint, M.var.dir = M_var_dir, out.eval = out_eval, 
          batch = batch_fin, rep.n = rep_n, rep.type = rep_type, jackknife = jackknife,
          out.dir = mod_dir, out.format = out_format, project = project, 
          G.var.dir = G_var_dir, ext.type = ext_type, 
          write.mess = write_mess, write.clamp = write_clamp, 
          maxent.path = maxent_path, wait = wait1, run = run1)
```

# VISUALIZAMOS LOS MODELOS

## GEOGRAFICAMENTE

### EN LA ZONA NATIVA

``` r
# Nativos
raster_ne_nativo <- rast("Final_Models/M_2_F_l_set_1_NE/Ctenosaura_similis_avg.asc")

occs_joint <- read_csv("occ_joint.csv")
occs_joint_spatvector <- vect(occs_joint, geom=c("lon", "lat"), crs = "EPSG:4326")

# graficamos
plot(raster_ne_nativo, main = "NE", axes = FALSE, legend = TRUE) 
points(occs_joint_spatvector, pch = 16, cex = 0.5, col = "red")
```

### EN LA ZONA DE INVASIÓN

``` r
# Invasores
raster_ec_invasor <- rast("Final_Models/M_2_F_l_set_1_EC/Ctenosaura_similis_set_1_avg.asc")
raster_e_invasor <- rast("Final_Models/M_2_F_l_set_1_E/Ctenosaura_similis_set_1_avg.asc")
raster_ne_invasor <- rast("Final_Models/M_2_F_l_set_1_NE/Ctenosaura_similis_set_1_avg.asc")

occs_ind_spatvector <- vect(occs_ind, geom = c("lon", "lat"), crs = "EPSG:4326")

# graficamos
par(mfrow = c(1, 3))
# Grafica cada raster con su título específico
plot(raster_ec_invasor, main = "EC", axes = FALSE, legend = FALSE)
points(occs_ind_spatvector, pch = 16, cex = 1, col = "red")
plot(raster_e_invasor, main = "E", axes = FALSE, legend = FALSE)
points(occs_ind_spatvector, pch = 16, cex = 1, col = "red")
plot(raster_ne_invasor, main = "NE", axes = FALSE, legend = TRUE)
points(occs_ind_spatvector, pch = 16, cex = 1, col = "red")
```

### CURVAS DE RESPUESTA

CARGAR IMAGENES

## Podemos evaluar el modelo en la zona de transferencia

``` r
occs_inv <- read_csv("occ_ind.csv") %>%
            dplyr::select(longitude = lon, latitude = lat)

raster_ec_invasor <- raster("Final_Models/M_2_F_l_set_1_EC/Ctenosaura_similis_set_1_avg.asc")
raster_e_invasor <- raster("Final_Models/M_2_F_l_set_1_E/Ctenosaura_similis_set_1_avg.asc")

thres <- 5
rand_perc <- 50
iterac <- 500

## corremos la función 
p_roc_E <- pROC(raster_e_invasor, occs_inv, n_iter = 500, E_percent = 5, 
                boost_percent = 50)
p_roc_E$pROC_summary

p_roc_EC <- pROC(raster_ec_invasor, occs_inv, n_iter = 500, E_percent = 5, 
                boost_percent = 50)
p_roc_EC$pROC_summary


E_ev <- p_roc_E$pROC_results[4] 
EC_ev <- p_roc_EC$pROC_results[4]

E_ev["tipo_ext"] <- "E"
EC_ev["tipo_ext"] <- "EC"

eval <- rbind.data.frame(E_ev, EC_ev)

ggplot() + geom_boxplot(data = eval, aes(x = tipo_ext, y = auc_ratio))
```

# ANALISIS DE EXTRAPOLACIÓN

## CORRER EL MOP

``` r
# cargamos las variables
mvars_stack <- raster::stack(list.files(path = "M_var/set_1", pattern = "\\.asc$", 
                                  full.names = TRUE))
gvars_stack <- raster::stack(list.files(path = "G_var/set_1/set_1", pattern = "\\.asc$", 
                                  full.names = TRUE))
mop_res <- mop(M_stack = mvars_stack, G_stack = gvars_stack, percent = 10, 
               comp_each = 2000)
```

## VISUALIZARLO

``` r
plot(mop_res)
```

# EJERCICIO PARA USTEDES, IMAGINEMOS QUE EL SITIO NATIVO DE C. ES FLORIDA E INVADIÓ MEXICO

# Y CENTROAMERICA.. ¿CÓMO SE VEN LOS RESULTADOS?

# PREGUNTAS ¿EN QUÉ ESCENARIO SE PREDICE MEJOR LA INVASIÓN?

# ¿QUÉ CAMBIA EN LAS MÉTRICAS DE EVALUACIÓN?

# ¿CÓMO CAMBIA EL MOP?

# ¿CÓMO CAMBIAN LAS CURVAS DE RESPUESTA?

# ¿QUÉ PROCEDIMIENTO DE TRANSFERENCIA USARIAN DEPENDIENDE DEL ESCENARIO?
