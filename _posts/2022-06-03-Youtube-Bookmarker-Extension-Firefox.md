---
layout: post
title: Youtube Bookmarker | Extensión para Firefox.
description: Nueva extensión para guardar tus momentos favoritos de Youtube
tags: [ Youtube , Javascript, Firefox, Extension ]
---

# Indice

1. [Introducción](#introduction)

2. [¿Qué es?](#que)

3. [Configuración](#configuracion)


# Introducción<a id="introduction"></a>

Desde hace algún tiempo quería intentar hacer algún tipo de extensión para Firefox. En algunas webs y blogs había visto cómo hacerlos, y parecía un proyecto interesante para llevar a cabo con Javascript. Así que hoy me he animado adaptando la versión `Chrome Youtube Bookmarker` de [raman-at-pieces](https://github.com/raman-at-pieces) al navegador Firefox. 

Para ello he tenido que cambiar algunos parámetros, ya que cada navegador tiene sus peculiaridades, y ya de paso he decidido cambiar un poco el estilo y algunas funciones que no terminaban de devolver los valores esperados.

Mi idea es ir desarrollando esta herramienta poco a poco. En primer lugar pienso añadir la función de editar las diferentes bookmarks con algún título más descriptivo (ya que ahora sólo aparecen como Bookmark at 00:00:23, o el segundo que corresponda...). Pero para ello necesito algo de tiempo. Una vez que haya hecho estos cambios, intentaré subir la extensión al repositorio de Firefox.

# ¿Qué es? <a id="que"></a>

Se trata de una extensión desarrollada en Javascript que permite guardar marcas de tiempo en los vídeos de Youtube. Cada vez que estés reproduciendo un vídeo y quieras guardar tu momento favorito para volver a él más tarde, puedes hacer click en el icono que aparece en el reproductor y se guardará el `timestamp` de ese momento concreto. Si haces click en la extensión, puedes volver a reproducir el vídeo desde ese momento. 

Puedes guardar tantas marcas como desees en un vídeo. Si navegas a otro vídeo y vuelves al original, se habrán quedado las marcas tal y como las guardaste, es decir, las marcas se guardan en relación al vídeo que estés reproduciendo.

# Configuración <a id="configuracion"></a>

Puedes clonar el proyecto con:

```
git clone https://github.com/BdeVallejo/youtube-bookmark-mozilla-extension
```

Una vez clonado, puedes probarlo mediante estos pasos:

   - Abre tu navegador Firefox.
   - Ve a about:debugging.
   - Haz click en "Cargar complemento temporal", que está dentro de "Este Firefox".
   - Selecciona el manifest.json que está en la carpeta del proyecto
   - Ve a Youtube y reproduce cualquier vídeo.

