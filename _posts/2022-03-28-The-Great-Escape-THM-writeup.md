---
layout: post
title: The Great Escape. Máquina de Try Hack Me
description: Write up de mi experiencia con la máquina.
tags: [ Docker , TryHackMe , seguridad, write-ups, Git ]
---

# Indice
1. [Introducción](#introduction)

2. [Reconocimiento](#reconocimiento)
    1. [nmap](#nmap)
    2. [gobuster](#gobuster)
    3. [robots.txt](#robots)
    4. [wfuzz](#wfuzz)

3. [Inyección de comandos](#inyeccion)

3. [Git](#git)

4. [Port-Knocking](#port-knocking)

5. [Docker](#docker)

6. [Not so well-known](#well-known)

# Introducción <a id="introduction"></a>

Hoy voy a intentar una nueva máquina de Try Hack Me! Se trata de un ejercicio en el que de alguna manera se tocará Docker. Algo en lo que llevo inmerso un tiempo así que voy a probar mis habilidades. 

Tal y como nos dicen en el Task 2 , la primera flag está escondida en la webapp.  Así que abro el navegado y entro en ella con la IP de la máquina. Se trata de una aplicación con lecciones de fotografía. Los cursos no se pueden ver sin hacer login, y cuando intento registrarme me aparece un mensaje que no se pueden hacer nuevos usuarios para evitar rogue accounts, esto es, una cuenta hecha sin autorización. 

Intento varias combinaciones usuario/contraseña para loguearme, sin éxito. 


# Reconocimiento <a id="reconocimiento"></a>

## nmap<a id="nmap"></a>

Comienzo con un reconocimiento básico con nmap:

```
nmap -sS --min-rate 5000 -p- --open -v -n -Pn IPDELAMAQUINA
```
Me aparecen dos puertos abiertos : 22/tcp y 80/tcp, esto es, un puerto para conectarnos mediante ssh y otro http.

## gobuster<a id="gobuster"></a>

Tiro de gobuster para buscar directorios :
```
gobuster dir -u http://IPDELAMAQUINA -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64
```
Sin éxito. Obtengo sin embargo un error extraño que no creo haber visto antes, “error the server returns a status code that matches the provided options for non existing urls… ” así que [investigo un poco](https://infinitelogins.com/2020/09/05/dealing-gobuster-wildcard-and-status-code-errors/)


Por lo visto, el servidor responde con un status 200, así que gobuster no puede encontrar directorios “ocultos” porque siempre obtiene una respuesta del servidor de tipo “OK”. Así que afino un poco mi búsqueda: 

```
gobuster -u http://IPDELAMAQUINA -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -s "204,301,302,307,401,403"
```

Consigo obtener un directorio /api . Pruebo y obtengo un mensaje: `Nothing to see here, move along...` 

## robots.txt<a id="robots"></a>

Hora de probar si algún archivo robots.txt nos dá alguna pista. Navego a http://IPDELAMAQUINA/robots.txt y ahora sí, encuentro algunos directorios:

````
User-agent: *
Allow: /
Disallow: /api/
# Disallow: /exif-util
Disallow: /*.bak.txt$
````

Pruebo con `/exif-util` y parece ser lo que busco. Un servicio donde podemos subir archivos desde nuestro ordenador o bien mediante una url. Intento subir archivos .php de prueba para inyectar código o incluso una reverse shell, sin éxito. Incluso modificando el magic number con hexedit para intentar saltarme la validación me aparece un error. 

## wfuzz<a id="wfuzz"></a>

Ahora voy a investigar un poco el directorio `/*.bak.txt` que aparece en robots.txt y los archivos que puede contener. Wfuzz es la herramienta perfecta para ello:

```
wfuzz -u http://IPDELAMAQUINA/FUZZ.bak.txt -w root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt --hc 404 
```

No obtengo éxito. Y sin embargo estoy seguro de que debe de haber algo, así que no me desanimo y creo un pequeño diccionario con las palabras más utilizadas de la web con `cewl`. Como no hay demasiadas, añado algunas que están relacionadas como admin, photos, login, log-in, exif, exif-util, api, photo, photography..

````
cewl http://IPDELAMAQUINA -w /root/Desktop/dict.txt
````

Y lo intento de nuevo:

```
wfuzz -u http://IPDELAMAQUINA/FUZZ.bak.txt -w /Desktop/dict.txt --hc 404 
```

Y por fin, éxito: `exif-util.bak.txt` existe. Y tiene un script que básicamente llama a la url http://api-dev-backup:8080/exif al que añade un parámetro ‘url’ cuando utilizamos la utilidad de exif-util.


# Inyección de comandos <a id="inyeccion"></a>

Vuelvo a http://IPDELAMAQUINA/exif-util y en el apartado de ‘From URL’ intento inyectar parámetros de tipo http://api-dev-backup:8080/exif?url=whoami … sin demasiado éxito por el momento.

Sigo probando hasta que me doy cuenta que puedo inyectar un segundo parámetro, de tipo comando de linux. 
```
http://api-dev-backup:8080/exif?url=123;whoami
```

Nos devuelve ‘root’. Bien, somos root. Pruebo más como
```
http://api-dev-backup:8080/exif?url=123;ls -la /etc/passwd
http://api-dev-backup:8080/exif?url=123;ls -la /root
```

Este último devuelve algo interesante: un archivo `dev-note.txt`:
```

Hey guys,

Apparently leaving the flag and docker access on the server is a bad idea, or so the security guys tell me. I've deleted the stuff.

Anyways, the password is fluffybunnies123

Cheers,

Hydra
```

# Git <a id="git"></a>

El archivo nos da una pista. Parece ser que antes existía una flag y un acceso a docker desde el servidor. Por tanto, es posible que podamos encontrarlo si Hydra ha utilizado git e indagamos un poco en los commits antiguos. 

Efectivamente, utiliza git. Leo primeramente el `.gitconfig` que contiene lo siguiente.
```
email = hydragyrum@example.com
name = Hydra
```

Me muevo por los diferentes archivos de git, encontrando que hay varios commits detallados en :
```
http://api-dev-backup:8080/exif?url=123;cat /root/.git/logs/HEAD
```

Pero si intento acceder a los branches me dice `Request contains banned words`. Pruebo entonces a saltarme la resticción ejecutando diferentes comandos de git y buscando diferencias entre los commits con un comando como este:

```
http://api-dev-backup:8080/exif?url=123;git --git-dir /root/.git/ diff 

```

Y encuentro lo siguiente:

```
I got tired of losing the ssh key all the time so I setup a way to open up the docker for remote admin.
-
-Just knock on ports 42, 1337, 10420, 6969, and 63000 to open the docker tcp port.
-
-Cheers,
-
-Hydra
\ No newline at end of file
diff --git a/flag.txt b/flag.txt
deleted file mode 100644
index aae8129..0000000
--- a/flag.txt
+++ /dev/null
@@ -1,3 +0,0 @@
-You found the root flag, or did you?
THM{....}
```

# Port-knocking<a id="port-knocking"></a>

Ahora tengo que ejecutar un [port-knocking](https://es.wikipedia.org/wiki/Golpeo_de_puertos) para abrir el puerto de docker. 

```
knock IPDELAMAQUINA 1337 10420 6969 63000
```

Vuelvo a ejecutar un escaneo de puertos con nmap y tenemos un nuevo puerto abierto: 2375. 

# Docker<a id="docker"></a>

Cruzo los dedos para que Docker esté configurado para ejecutarlo remotamente:

```
docker -H IPDELAMAQUINA:2375 ps
```

Funciona. 

Responde. Tenemos 4 imágenes , frontend, exif-api-dev, exif-api y endlessh. 

El próximo paso requiere una pequeña explicación. 

En algún punto, los desarrolladores de docker decidieron que, para evitar escribir sudo delante de cualquier instrucción que diéramos, se crearía un grupo llamado ‘docker’ donde cualquier usuario tiene acceso como root al daemon de Docker. Dicho de otra forma, si estamos en el grupo ‘docker’ es como si fuéramos usuarios root. 

Por otro lado, conviene saber que Docker utiliza UNIX sockets por defecto para interactuar con el docker engine. Sin embargo, si utilizamos docker desde otro ordenador, o si queremos usarlo desde herramientas como Portainer o Jenkins, nos conectaremos mediante un TCP socket .Esto es lo que acabamos de hacer, y funciona.
Además, como somos un usuario con privilegios para ejecutar comandos (estamos dentro del grupo ‘docker’) , podemos montar el directorio ‘host’ en un nuevo contenedor para luego conectarnos a él y revelar todos los datos del sistema operativo. En este caso lo haremos utilizando la imagen frontend que ya está en el sistema. Luego nos conectamos a este directorio mediante una shell tras haber hecho chroot al directorio que acabamos de montar. 

Explicado tiene bastante miga, pero el comando es tan sencillo como:

```
docker -H IPDELAMAQUINA:2375 run -v /:/mnt --rm -it frontend chroot /mnt sh
```

Ya somos usuarios root. Si navegamos a /root veremos el archivo flag.txt

# Not so well-known<a id="well-known"></a>

Me falta una flag y la verdad que no sé dónde puede estar. Try Hack Me me da una pista, que está en un directorio well-known. Pero no se me ocurre nada. Así que voy a ser sincero: tuve que buscar la solución. Desconocía el directorio y no se me ocurría dónde podía encontrarlo.

Si navego a /home/hydra/docker-escape-compose/front/dist/ encuentro el directorio `.well-known`. Y dentro un archivo “security.txt” que me pide hacer un ping a api/fl46 con un Head request. Eso sí puedo hacerlo:

```
curl --head  http://10.10.71.243/api/fl46
```

Obtengo la flag y con ello completo el ejercicio.
