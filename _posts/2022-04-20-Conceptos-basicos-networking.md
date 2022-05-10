---
layout: post
title: Conceptos básicos de Networking
description: Algunos de los conceptos básicos de networking explicados
tags: [ networking ]
---

# Indice

1. [Introducción](#intro)

2. [Binario a decimal](#binario)

3. [Clases de redes](#clases)

4. [Subnetting](#subnetting)

5. [CIDR](#cidr)

# Introducción <a id="intro"></a>

Hoy quiero hacer un repaso de algunos conceptos básicos de networking. Desde siempre ha sido una parte que me ha costado interiorizar, pero con la ayuda de todos los recursos disponibles en internet y varias horas de tutoriales, artículos, etc, me encuentro cada vez más cómodo con estos conceptos. 

Escribirlos en este artículo me ayudará a recordarlos mejor. Y aunque aquí sólo me centro en los conceptos más básicos, resulta crucial entenderlos en profundidad.


# Binario a decimal<a id="binario"></a>

Una de las partes que considero más importantes y que  a menudo se dan por entendidas es la conversión de decimal a binario, y de binario a decimal. 


Una dirección IPv4 consiste en 32 bits separados en 4 secciones de 8 bits. Por ejemplo, si tenemos la dirección 192.168.1.1 la podríamos convertir a: 11000000.10101000.00000001.00000001

El tener a mano una tabla como esta puede ayudar bastante. Por ejemplo el número 192:

- 128 —> 1
- 64   —> 1
- 32   —> 0
- 16   —> 0
- 8     —> 0
- 4     —> 0
- 2     —> 0
- 1     —> 0

Si sumamos 128 + 64 tenemos como resultado 192. 

# Clases de redes<a id="clases"></a>

Una dirección IP se divide en dos partes. La parte de `network` y la parte de `nodo`. 

Teniendo esto en cuenta, podemos identificar 5 clases de redes.

- Clase A -> NETWORK.NODO.NODO.NODO
- Clase B -> NETWORK.NETWORK.NODO.NODO
- Clase C -> NETWORK.NETWORK.NETWORK.NODO
- Clase D y Clase E son para multicast  y fines de investigación y , así que se suelen obviar.

De tal forma, que podemos identificar ante qué clase de red estamos si nos fijamos en el primer byte. Si está comprendido entre el 0 y el 127 estamos ante una red de clase A. Si está entre el 128 y el 191 es una red de clase B. Y si es mayor que 192, ante una de clase C.

Los rangos de IP a su vez se puede dividir en subnets y en hosts. Una subnet es simplemente una subred, es decir, una red con varios dispositivos que a su vez están en una red mayor. Y los hosts son las direcciones IP que podemos asignar a estos dispositivos. El proceso de asignar subnets y hosts se llama `subnetting`.

# Subnetting<a id="subnetting"></a>

Para poder dividir una rango de IPs en subnets y hosts, tenemos que aplicar lo que se conoce como máscara de subred. 

Por ejemplo, si tenemos una dirección de clase C 192.168.0.0 y quisiéramos conectar 100 dispositivos a esa red, lo más sencillo sería tener una máscara de subred que fuera 255.255.255.0. ¿Y por qué?

Como hemos visto, una dirección IPv4 consiste en 32 bits separados en 4 secciones de 8 bits. Si convertimos 255.255.255.0 a binario, tendríamos 11111111 11111111 11111111 00000000. Los últimos 8 ceros nos indican el número de hosts que podemos conectar a esa red. Si hacemos el cálculo  28 nos da como resultado 256 dispositivos (habría que restarle dos, ya que típicamente la primera dirección se utiliza como un identificador de la subnet y la última como broadcast address , y se utiliza para mandar información a todos los dispositivos conectados a una subnet). Así que en total podríamos albergar 254 dispositivos. 

Este ejemplo sería el típico de una oficina pequeña o una vivienda. Ahora imaginemos que queremos configurar un espacio con una red de clase B con la dirección 172.16.0.1. En ella hay que alojar 100 subnets y unos 400 dispositivos en cada una.

En ese caso, si la máscara de subred fuera 255.255.255.0 , tendríamos espacio para 254 dispositivos. ¿Y de cuántas subnets podríamos disponer? Ahora hay que hacer el cálculo contando el número de “1” de los dos últimos octetos. Es decir, 8. 28 nos da como resultado 256 subnets. Así que hay que modificar la máscara para poder alojar 400 dispositivos. Para que nos cuadre todo, podríamos utilizar  la máscara 255.255.254.0. Traducido a binario es 1111111 11111111 11111110 00000000. El número de hosts sería 9 ( lo 9 últimos 0) , así que    29 nos da un total de 512 dispositivos (menos las dos IPs que no podemos utilizar, 510 en total). Y el número de subnets sería 27 , es decir 128. 

Afortunadamente, existen [calculadoras como esta](https://www.subnet-calculator.com/subnet.php?net_class=C) para ayudarnos.


# CIDR<a id="cidr"></a>

Cuando se habla de rangos de IP, a menudo se utilizan `/24` o bien `/16`.  Esta notación se conoce como `CIDR notation`. Es una fórmula más moderna, pero básicamente se refiere a los mismo. 

Es decir, si hablamos de una subnet con /16, se puede decir que la subnet es 255.255.0.0. Si nos referimos a una subnet con /24, será 255.255.255.0. 
