---
title: "Descargar, procesar, y acomodar imágenes en un mosaico hexagonal interactivo"
excerpt: Mosaicos hexagonales en HTML con múltiples imágenes desde un directorio.
category: rstats
tags:
  - perros
  - API
  - magick
  - hexágonos
  - css
header:
  image: /assets/images/featureMatrixInd.jpg
---

{% include puptile.html %}

El paquete [hexsession](https://hexsession.liomys.mx){:target="_blank"} sigue en desarrollo activo, y ahora ya funciona con conjuntos arbitrarios de imágenes. Como ejemplo, aquí dejo una guía de los pasos que podemos seguir para:

- descargar imágenes aleatorias de perros de la [API de Dog CEO](https://dog.ceo/dog-api/){:target="_blank"}
- redimensionar y convertir todas las imágenes a blanco y negro en un solo paso
- armar un mosaico hexagonal de las imágenes, cada una con un URL opcional

Siempre se me olvida cómo trabajar con APIs y como procesar varias imágenes en simultáneo, así que espero que esto sea útil para otros y que también me quede de referencia.

El resultado final está hasta arriba,se ve bien y este tipo de mosaicos se pueden usar para paquetes de software y sus logos, o tal vez para personas en una organización con fotos y enlaces al perfil de cada integrante.

Como siempre, el código en estos bloques es reproducible con una conexión a internet y los paquetes relevantes instalados. 

## Descargar fotos de perros

Podemos descargar fotos de perros de la API de Dog CEO. Esta API tiene un endpoint para obtener fotos de diferentes razas. Las imágenes vienen del [Stanford Dogs Dataset](http://vision.stanford.edu/aditya86/ImageNetDogs/){:target="_blank"} más contribuciones de la comunidad. Podemos usar `httr2` para enviar una solicitud GET que devuelve una lista de URLs de imágenes. Para llevarla leve con los servidores, hice toda la iteración aquí haciendo pausas (`slowly` con `purrr`). Primero obtenemos la lista de razas, seleccionamos algunas al azar, agregamos dos que me gustan y luego descargamos las fotos.

{% highlight r %}
library(httr2)
library(purrr)
library(stringr)
library(magick)

# definir función para obtener una imagen aleatoria de una raza
randomDog <- function(breed) {
  # construir URL
  url <- paste0("https://dog.ceo/api/breed/", breed, "/images/random")
  # solicitud
  req <- request(url)
  resp <- req |>
    req_perform()
  # respuesta JSON
  data <- resp_body_json(resp)
  data$message
}

# todas las razas
allbreeds <- 
  request("https://dog.ceo/api/breeds/list") |> 
  req_perform() |>
  resp_body_json()
breedsvec <- simplify(allbreeds$message)

# lentamente
rate <- rate_delay(0.8)
slow_randomdog <- slowly(\(x) randomDog(x), rate = rate, quiet = FALSE)
# 15 razas aleatorias más dos que me gustan
randomdogsvec <- sample(breedsvec, 15)
randomdogsvec <- c(randomdogsvec, "mastiff/bull","retriever/golden")
# obtener las URLs 
dogphotourls <- map(randomdogsvec, slow_randomdog)

# carpeta y rutas para guardar las imágenes
fs::dir_create("dogimgs")
savepaths <- paste0("dogimgs/",str_replace(randomdogsvec,"/","-"),".jpg") 
slowdl <- slowly(\(x,y) download.file(x,y), rate = rate, quiet = FALSE)
walk2(dogphotourls,savepaths,\(x,y) slowdl(x,y))

{% endhighlight %}

## Procesar las imágenes

Podemos usar `magick` para redimensionar todas las imágenes a una altura uniforme de 230px, pasarlas a blanco y negro, y exportar cada una al disco en una nueva carpeta. El truco aquí fue aplicar `as.list()` al objeto de imágenes apiladas de magick en el paso final.

{% highlight r %}
# directorio y rutas para las imágenes procesadas
fs::dir_create("dogimgs/processedimgs")
processedimgpaths <- paste0("dogimgs/processedimgs/",basename(savepaths))

# leer y procesar
allimgs <- image_read(savepaths)
procimgs <- allimgs |> 
  image_scale("230x") |>  # redimensionar
  image_convert(colorspace = 'gray') |> # desaturar
  image_contrast() # más contraste 

# escribir en disco
walk2(as.list(procimgs),processedimgpaths,\(x,y) image_write(x,y))

{% endhighlight %}

## Crear el mosaico

Ahora que hexsession acepta conjuntos arbitrarios de rutas de imágenes, podemos trabajar con un vector de rutas. Agregué la misma URL a cada imagen solo para mostrar que la salida HTML funciona.

{% highlight r %}
# reiniciar sesión primero para que no haya paquetes cargado
dogpaths <- fs::dir_ls("dogimgs/processedimgs/")
cturls <- rep("https://betterpet.com/why-are-dogs-so-cute/",length(dogpaths))
hexsession::make_tile(local_images = dogpaths,local_urls = cturls)
{% endhighlight %}

Para mostrar la salida HTML en esta publicación, puse el archivo HTML autocontenido en **_includes/** y me refiero a él con {% raw %} {% include %} {% endraw %}.

Quedo al pendiente para cualquier duda.
salu-2
