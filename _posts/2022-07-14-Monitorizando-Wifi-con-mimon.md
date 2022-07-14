---
layout: post
title: Monitorizando una Wifi con mimon
description: Monitorizando una red Wifi con la herramienta mimon
tags: [ Wifi, mimon ]
---

# Indice

1. [Introducción](#introduction)

2. [¿Qué es mimon?](#quees)

3. [¿Cómo funciona?](#como)


# Introducción<a id="introduction"></a>

Uno de los proyectos que tenía últimamente entre manos era una herramienta para monitorizar la Wifi. Tras unos días trabajando en ella por fin la he terminado y subido a mi repositorio de [Github](https://github.com/BdeVallejo/mimon).

He de agradecer las valiosísimas lecciones del maestro [S4vitar](https://github.com/s4vitar) cuyos [tutoriales de bash](https://www.youtube.com/watch?v=Mwt_RbdpJhk&t=4351s) me han servido como anillo al dedo para desarrollar esta herramienta.

# ¿Qué es mimon?<a id="quees"></a>

Mimon es un sencillo script en bash que escanea la red inalámbrica que le indiquemos en un momento dado. Como vemos en la imagen, la herramienta comprueba primero si tenemos todo lo necesario instalado antes de iniciar su trabajo.

<img src="../../../../assets/images/mimon/wifi.png"
     alt="Comprobando las herramientas"
     style="float: center; margin-right: 10px; " />

A partir de ahí, continúa el escaneo y nos avisa cuando haya algún dispositivo nuevo conectándose a nuestra red. ¿Y para qué puede servirnos? Puede avisarnos por ejemplo si sospechamos que algún vecino está conectándose a nuestra Wifi, o cuando alguna persona con la que convivimos se acerca a casa y su móvil se conecta a la Wifi antes de que abra la puerta... en fin, las posibilidades son variadas.

<img src="../../../../assets/images/mimon/new-device.png"
     alt="mimon nos avisa cuando alguien se conecta a la wifi"
     style="float: center; margin-right: 10px; " />

# ¿Cómo funciona?<a id="como"></a>

El único requerimiento a nivel técnico es contar con una tarjeta de red capaz de actuar en modo monitor. Una vez satisfecho ese requisito, tan sólo tenemos que clonar el repositorio, correr la herramienta indicándole con el parámetro "-n" la tarjeta de red y listo.

```
git clone https://github.com/BdeVallejo/mimon
cd mimon
sudo ./mimon.sh -n <tarjetadered> 
```
