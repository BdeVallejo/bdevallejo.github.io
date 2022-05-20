---
layout: post
title: Attacktive Directory | THM Write-up
description: Write up de la máquina Attacktive Directory de la plataforma Try Hack Me
tags: [ write-ups , seguridad , TryHackMe, Windows, AD, Domain Controller ]
---

# Indice
1. [Introducción](#introduction)

2. [Welcome to Attacktive directory](#task3)

3. [Enumerating Users via Kerberos](#task4)

4. [Abusing Kerberos](#task5) 

5. [Back to the Basics](#task6) 

6. [Elevating Privileges within the Domain](#task7) 

7. [Flag Submission Panel](#task8)  


# Introducción <a id="introduction"></a>

Como usuario de Linux/Mac, a menudo me cuesta recordar conceptos básicos de Windows. Es por ello que últimamente estoy intentando enfocarme más en este sistema operativo. Para ello, hoy voy a jugar con la máquina Attacktive Directory de [TryHackMe](https://www.tryhackme.com) y así poder repasar conceptos de autenticación, directorio activo, kerberos, etcétera. Todo esto me servirá para el examen de Certified Ethical Hacker v11 que quiero realizar en las próximas semanas.

NOTA: Los pasos 1 y 2 me los salto porque simplemente tratan de configurar la máquina e instalar los programas y scripts necesarios. 

# Welcome to Attacktive directory  <a id="task3"></a>

Este primer paso es sencillo. Simplemente trata de enumerar el `SMB`, que como ya sabemos corre por el puerto 139 (a través de NETBIOS) o bien a través del 445 (TCP). Para enumerarlo podemos usar `enum4linux`o bien nmap de la siguiente forma:

```
nmap -p 445 –script=smb-enum-shares.nse,smb-esnum-users.nse IPDELAVICTIMA
```

Pero para responder a la segunda pregunta, bastará con tirar de enum4linux con:

```
enum4linux -M IPDELAVICTIMA
```
(el parámetro M es porque sólo necesitamos saber el Domain Name de la máquina. Podríamos haber utilizado el parámetro -a para hacer una enumeración completa).

Las dos primeras preguntas estarían respondidas. Tan sólo queda la tercera, en la que nos preguntan cuál es el TLD (Top Level Domain) que no es válido y se suele utilizar. Generalmente se utiliza `.com`aunque también se suelen utilizar .home, .lan, .domain, .corp, .local... etcétera. Es este último el que es inválido , tal y como explican [en este hilo](https://serverfault.com/questions/1006000/is-tld-local-not-a-local-tld-anymore).

# Enumerating Users via Kerberos <a id="task4"></a>

Kerberos es el protocolo de autenticación que se usa por defecto en Windows. Utiliza una encriptación de llave secreta y por resumir mucho lo que hacemos cuando intentamos acceder como usuarios a una máquina Windows mediante este protocolo es lo siguiente:

 - Hacemos una petición a un Authentication Server (AS). Éste nos responde.

 - Hacemos otra petición al Ticket Granting Server (TGS). Éste nos devuleve un ticket.

- Una vez realizado este proceso ya podemos autenticarnos contra el Service Server (SS) enviando los mensajes que hemos recibido en los pasos anteriores. El Service Server validará los mensajes y así confirmar nuestra identidad.

NOTA: En la propia descripción de la máquina, nos avisan que las nuevas versiones de Kerberos no permiten la enumeración de usuarios mediante `Kerbrute`, que es lo que utilizaremos en el ejercicio. Así que han tenido que modificar la máquina sobre la que estamos trabajando para que funcione, pero en un caso de enumeración real habría que buscar otras vías. 

Me bajo el userlist, passwordlist y el propio kerbrute de github con:

```
git clone https://github.com/Sq00ky/attacktive-directory-tools
cd attacktive-directory-tools
chmod +x kerbrute
./kerbrute
```

Nos aparecen una lista de opciones, donde veo que el comando `userenum` sirve para ver usuarios. En el apartado anterior ya nos habían "chivado" que el dominio del AD era `spookysec.local`. Por tanto, tras ojear la documentación de kerbrute, pruebo lo siguiente para enumerar usuarios:

```
./kerbrute userenum -d IPDELAVICTIMA -dc spookysec.local userlist.txt
```

Nos dará como resultado una lista de usuarios, entre los que destacan "svc-admin" y "backup". 

# Abusing Kerberos <a id="task5"></a>

Ahora vamos a ejecutar un ataque de tipos `ASREPRoasting`, en el que abusamos de una configuración poco segura que supone la opción "No requiere Pre-Autenticación". Esta opción nos permite no tener una ID válida para pedir un ticket al Ticket Granting Server (TGS), y obviamente esto conlleva cierto riesgo. 

Mediante la herramienta `GetNPUsers.py` de `Impacket` podemos ejecutar este tipo de ataque si hay usuarios que tienen este opción configurada. Para ello necesitamos contar con uno, así que vamos a probar con el de svc-admin.

La primera pregunta es, por tanto, svc-admin.

Ahora voy a utilizar `GetNPUsers.py`, que está en la carpeta "/opt/impacket/examples". Es importante echar un ojo a las opciones antes de ejecutar el comando para entenderlo:

```
cd /opt/impacket/examples
./GetNPUsers.py
// (miro las opciones)
GetNPUsers.py -dc-ip IPVICTIMA spookysec.local/svc-admin
```

Me pide un password. Doy a ENTER y me aparece un hash "$krb5asrep$23...". En la descripción nos remiten a la página de [Hashcat](https://hashcat.net/wiki/doku.php?id=example_hashes) para identificar el hash. Si filtro por "Kerberos" aparecen unos cuantos, pero el que más se aproxima al que he obtenido es `18200 	Kerberos 5, etype 23, AS-REP ` (mode 18200). 

Ahora queda crackear el hash con hashcat. Para ello primero copio el hash y lo meto en un archivo "hash.txt". 

```
hashcat -a 0 -m 17299 hash.txt ~/attacktive-directory-tools/passwordlist.txt
```
En unos segundos nos devuelve el hash crackeado. 

NOTA: El password aparece al final del hash, antes de la información donde nos aparece la sesión, el estatus, etcétera. Empieza por manage... 

# Back to the Basics <a id="task6"></a>

Ahora vamos intentar acceder con el usuario y la contraseña apropiadas al servidor SMB. Para ello utilizaremos `smbclient` (primera pregunta respondida). Si hacemos:

````
smbclient
````

Nos saldrá una lista de opciones, entre ellas -L para listar hosts (segunda pregunta). Para listar, por tanto, los shares:

````
smbclient -L IPVICTIMA -U svc-admin
````

Nos pide la contraseña que hemos obtenido antes, y nos aparecen 6 shares.Para el siguiente bloque de preguntas es necesario conectarse al share de `backup`. 

````
smbclient //IPVICTIMA/backup -U svc-admin
````

Y tras meter la contraseña, estamos dentro. Veo que hay un archivo llamado `backup_credentials.txt`. Me lo bajo:

````
ls
GET backup_credentials.txt
````

Una vez en mi máquina local, lo abro y veo que es un hash. Lo copio y lo trato de descodificar en una herramienta online ( busca con google, hay un montón), para descubrir que se trataba de una codificación en Base-64, por lo que podría haberlo hecho con `base64 -d` cómodamente en mi Linux. De todo se aprende...

# Elevating Privileges within the Domain <a id="task7"></a>

Aparentemente, según nos indican, tenemos la cuenta de backup del Domain Controller. Es decir, que todos los datos del Domain Controller se sincronizan con esta cuenta. Así que con la herramienta `secretsdump.py` del Impacket, podemos ver todos los passwords (hasheados, eso sí).

```
cd /opt/impacket/examples
.secretsdump.py
// miro el manual
secretsdump.py -just-dc backup@backup2517860@IPVICTIMA
```


(Al principio del output viene la respuesta a la pregunta 1).

La pregunta 3 es engañosa. Nos preguntan por cómo podemos autenticarnos sin necesidad del password. La respuesta es `Pass The Hash`, pero con éste método si que necesitamos el hash del password. A mi entender, está un poco mal formulada...

La siguiente pregunta es sobre [Evil-WinRM](https://github.com/Hackplayers/evil-winrm), y en el enlace podemos ver la respuesta ( - H por si eres vago/a y no quieres mirarlo).


# Flag Submission Panel <a id="task8"></a>

Ahora con la herramienta [Evil-WinRM](https://github.com/Hackplayers/evil-winrm) vamos a conectarnos a cada uno de los escritorios remotos mediante el hash obtenido de sus passwords en el ejercicio anterior. Primero lo instalaré y luego lo ejecutaré con el hash de Administrator:

````
gem install evil-winrm
evil-winrm -i IPVICTIMA -u Administrator -H HASHDELADMINISTRATOR

C:\Users\Administrator\Documents> cd \users
C:\Users> cd svc-admin
C:\Users\svc-admin> cd Desktop
C:\Users\svc-admin\Desktop> cat user.txt.txt
````

Y repetimos para el usuario backup y Administrator. 