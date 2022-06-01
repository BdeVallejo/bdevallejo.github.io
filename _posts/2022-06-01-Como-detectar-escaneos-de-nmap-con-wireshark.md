---
layout: post
title: Cómo detectar escaneos de nmap con Wireshark
description: Cómo detectar escaneos de nmap con Wireshark
tags: [ Wireshark , nmap ]
---

# Indice

1. [Introducción](#introduction)

2. [Controla tus puertos raros](#puertos-raros)

3. [Conceptos básicos de flags TCP](#basicos)
    1. [Full Handshake](#full-handshake)
    2. [Stealth Scan](#ss)
    3. [Null Scan](#ns)

4. [Window Size](#window-size)



# Introducción<a id="introduction"></a>

Como probablemente todos sabéis, `nmap` es la herramienta de escaneo por excelencia. En su página web tienen hasta una sección [dedicada a las películas en las que aparece esta herramienta](https://nmap.org/movies/).

Por otro lado, [Wireshark](https://www.wireshark.org/) es una de la herramientas de análisis de protocolos de red más utilizadas y más completas que existen.

En el post de hoy voy a probar diferentes escaneos en mi red local con nmap y tratar de filtrarlos con Wireshark para ver de qué manera se pueden detectar los escaneos y de qué maneras se pueden intentar evadir esas detecciones.


# Controla tus puertos raros <a id="puertos-raros"></a>

El primer método puede parecer banal, pero lo incluyo igualmente.

Una de las maneras mediante las cuales podemos detectar un escaneo de puertos es filtrando por puertos raros. Es decir, si yo tengo un servicio web corriendo, será normal que reciba peticiones dirigidas al puerto 80 o 443. Pero si recibo peticiones al puerto 1234, donde no es habitual que haya servicios corriendo, puedo sospechar algo raro.

Incluyo una capturas de pantalla donde se refleja el tráfico dirigido al puerto 443 y filtrado con: 

```
tcp.port==443
``` 


<img src="../../assets/images/Detectar-escaneos-nmap/imagen5.png"
     alt="Trafico dirigido al puerto 443"
     style="float: center; margin-right: 10px; " />

Y otra del tráfico dirigido al puerto 1234:

<img src="../../assets/images/Detectar-escaneos-nmap/imagen6.png"
     alt="Trafico dirigido al puerto 1234"
     style="float: center; margin-right: 10px; " />


# Conceptos básicos de Flags TCP<a id="basicos"></a>

Como es bien sabido, es frecuente realizar escaneos con nmap jugando con el Flag TCP que se envía. Por ejemplo, el escaneo más frecuente que se utiliza en contextos de CTF es el `Stealth Scan` en el que se envían las flags SYN,SYN/ACK y Reset, sin completar el llamado TCP handshake y por tanto sin completar la conexión. Otro de los más famosos es el `xmas scan` en el que se mandan las flags FIN, PSH y URG. Si no tenéis ni idea de lo que hablo, echad un ojo a [este post que lo explica bastante bien](https://www.eduardocollado.com/2020/03/13/flags-de-tcp/). 

La idea básica es que si queremos detectar uno de estos escaneos podemos hacerlo si conocemos qué valor se asigna a cada flag TCP. Con estos datos en la mano podemos aplicar un primer filtrado para determinar si alguien ha hecho algún escaneo y qué tipo de escaneo ha realizado:

| Cabecera |  Valor| 
|---	|---	|
|   SYN	|   1|
|   SYN/ACK	|2|
|   ACK|   4| 
|   DATA|   8| 
|   FIN|   16| 
|   RESET|  32| 

Una vez conocidos estos valores, uno de los filtros TCP de wireshark que nos ayuda a detectar escaneos es `tcp.completeness`. Vamos a verlo con ejemplos.

## Full Handshake <a id="full-handshake"></a>

Para empezar voy a ejecutar un escaneo básico (full-handshake) con nmap mientras capturo el tráfico con Wireshark.

````
nmap IP
````

Una vez capturado este tráfico, voy intentar filtrar por el campo tcp.completeness. La conexión que se ha debido establecer habrá sido SYN -> SYN/ACK -> ACK -> RESET. Voy a sumar todos los valores ( 1 + 2 + 4 + 32 ) y me sale un valor de 39. Por tanto en Wireshark:

```
tcp.completeness==39
```

En este caso sólo he abierto uno de los puertos de la máquina víctima para hacer el ejercicio más claro: el puerto 1234. 

<img src="../../assets/images/Detectar-escaneos-nmap/imagen1.png"
     alt="Captura de Full Handshake en Wireshark"
     style="float: center; margin-right: 10px; " />


La captura de pantalla no es muy buena, pero en ella se ven sólo las conexiones desde la máquina atacante a la máquina víctima al puerto 1234.

Ahora voy a filtrar por todas las peticiones que no han sido exitosas. Es decir, aquellas que hayan resultado del tipo SYN -> RST/ACK. Ahora filtraré por el número 37 ( 1 + 4 + 32 ).

```
tcp.completeness==37
```

<img src="../../assets/images/Detectar-escaneos-nmap/imagen2.png"
     alt="Captura de Full Handshake en Wireshark 2"
     style="float: center; margin-right: 10px; " />


Me aparecen un montón de resultados, uno por cada puerto cerrado.

## Stealth Scan <a id="ss"></a>

Ahora continuaré haciendo un Stealth Scan;

````
nmap -sS IP
````

La conexión que se ha debido establecer para el puerto abierto habrá sido SYN -> SYN/ACK -> RESET, ya que no se ha establecido la conexión al completo sino que ha sido interrumpida. Si sumo ( 1 + 2 + 32 ):

```
tcp.completeness==35
```

Y ahora el resultado es de tan solo tres paquetes, todos ellos correspondientes al puerto 1234:

<img src="../../assets/images/Detectar-escaneos-nmap/imagen3.png"
     alt="Captura de Stealth Scan en Wireshark"
     style="float: center; margin-right: 10px; " />


Así podríamos seguir filtrando para averigüar el tipo de escaneo que se está ejecutando y si ha sido exitoso o no.

## Null Scan <a id="ns"></a>

Una de las formas de evitar este tipo de filtros sería enviar un Null Scan.

En el caso de hacer un Null Scan con nmap, no se envían Flags TCP. Por increíble que parezca nmap tiene esta opción (-sN). ¿Y cómo es posible saber si hay servicios corriendo en este caso? Pues bien, nmap calcula que si no enviamos ninguna flag TCP y no hay respuesta, significa que o bien hay un firewall o bien el puerto está abierto. De lo contrario, la respuesta tendría una flag RST. 

Desde Wireshark podemos saber si se está utilizando este tipo de escaneo. Para ello deberíamos utilizar el filtro `tcp.flags`.

En concreto, con 

```
tcp.flags==0
```
nos aparecerían escaneos TCP Null.


# Window Size <a id="window-size"></a>

Otro de los aspectos a tener en cuenta cuando se hace un escaneo Stealth con nmap es el Window Size. Este parámetro suele tener un valor variable, y no es más que el valor (en bytes) que nuestro dispositivo puede recibir en un momento dado.   

Nmap usa por defecto una Window Size de 1024 en los paquetes SYN cuando hacemos un Stealth Scan. Por tanto sólo habría que establecer un filtro que sea: 

```
tcp.window_size_value == 1024
```

<img src="../../assets/images/Detectar-escaneos-nmap/image4.png"
     alt="Window Size en nmap"
     style="float: center; margin-right: 10px; " />

En el caso de un full handshake el Window Size es de 64240.

NOTA: He estado mirando otros blogs y vídeos de Youtube y algunos establecen esta cifra en 65535. Yo pongo la cifra que aparece en mis capturas de Wireshark. 

¿Hay manera de evitar esto como atacantes/pentesters? Por supuesto. Tal y como indican en su [página](https://seclists.org/nmap-dev/2015/q3/52) podemos evitar mandar un window size fijo de 1024 bytes con nmap mediante el parámetro --win.