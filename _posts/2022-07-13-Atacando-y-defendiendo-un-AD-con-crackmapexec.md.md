---
layout: post
title: Atacando (y defendiendo) un Domain Controller con crackmapexec
description: Configuro un Domain Controller para después atacarlo y aprender a defenderlo.
tags: [ AD , seguridad, testing, crackmapexec ]
---

# Indice

1. [Introducción](#introduction)

2. [Creación de usuarios](#creacion)

3. [Protocolos](#protocolos)

4. [Ataque](#ataque)

5. [Defensa](#defensa)



# Introducción<a id="introduction"></a>

En los últimos días he estado configurando un Domain Controller con VMWare para crear usuarios ficticios e intentar hackearlo de varias maneras. El primer ejercicio que he hecho ha sido con [crackmapexec](https://github.com/Porchetta-Industries/CrackMapExec). Ésta es una herramienta tremendamente útil para el pentesting ya que, entre otras cosas, hace uso de varios protocolos (winrm, http, smb, ssh ...), y dentro de cada uno de ellos, podemos ejecutar diferentes acciones.

# Creación de usuarios<a id="creacion"></a>

Siguiendo el ejemplo de [Mr. John Hammond](https://www.youtube.com/c/JohnHammond010) he creado una [herramienta](https://github.com/BdeVallejo/Breaking-Active-Directory) en `Powershell` que permite crear un `json` con varios usuarios (por defecto 10, pero se pueden crear tantos como desees) para después añadirlos a un directorio activo. Una vez realizado este proceso podemos intentar atacar el Controlador del Dominio, enumerar usuarios, etcétera.

Tal y como explico en la descripción de la herramienta, hay dos diferencias en mi código y el código que utiliza John. En primer lugar, mi herramienta está preparada para lidiar con nombres españoles y latinos, con acentos, eñes y nombres y apellidos compuestos.

En segundo lugar, John configura el Domain Controller de manera que no utiliza los requerimientos de complejidad de una contraseña que vienen por defecto en Windows Server 2022. ¿Qué quiere decir esto? Que un usuario podría tener una contraseña como `password123`, por ejemplo. En mi caso, he preferido ceñirme un poco al mundo real. Sólo un poco, porque las contraseñas que mi herramienta utiliza están incluídas en el famoso repositorio de [SecLists](https://github.com/danielmiessler/SecLists) de Daniel Miessler, y por tanto son relativamente. Pero si queréis utilizar el `rockyou.txt` , os aviso que muchas de las contraseñas que vienen incluídas en este diccionario no cumplen con las normas de Windows. Así que prácticamente podéis descartarlo para este tipo de ataques.

Un ejercicio más ceñido a la realidad sería el crear un diccionario con [Cewl](https://github.com/digininja/CeWL) antes de ejecutar el ataque con `crackmapexec`. Pero insisto, todas las contraseñas que he utilizado en mi herramienta están incluídas en `SecLists` ( más concretamente en /Passwords/Permutations).

En mi [Github](https://github.com/BdeVallejo/Breaking-Active-Directory) están descritos los pasos a seguir para crear los usuarios en el Domain Controller, así que saltaré esta parte aquí.

# Buscando el protocolo <a id="protocolos"></a>

Como he dicho antes, crackmapexec hace uso de varios protocolos (winrm, http, smb, ssh ...). Pero sin duda la joya de la corona smb, que corre generalmente por puerto 445. Así que merece la pena escanear con `nmap` nuestro Domain Controller para ver qué puertos están abiertos.

<img src="../../../../assets/images/ad-crackmapexec/nmap-scan.png"
     alt="Puerto 445 abierto"
     style="float: center; margin-right: 10px; " />

Puerto 445 abierto. Podemos ejecutar crackmapexec con
```
crackmapexec smb <IPDELDOMAINCONTROLLER>
```

Este escaneo rápido nos proporciona una información similar a la que nos daría nmap por ejemplo. Pero es un buen comienzo para identificar el nombre del DC.

<img src="../../../../assets/images/ad-crackmapexec/crackmapexec-smb-ip.png"
     alt="Información del DC"
     style="float: center; margin-right: 10px; " />

# Ataque <a id="ataque"></a>

Llega el momento de atacar. Para ello necesitamos una lista de usuarios y otra de contraseñas. ¿Y cómo la conseguimos? Una de las cosas que más me ha llamado la atención de siempre es la facilidad para saber el nombre de usuario de una compañia. Basta con hacer una búsqueda rápida en LinkedIn para tener una lista de correos electrónicos. Y en la mayoría de empresas, el formato es "Inicial del nombre""Primer apellido"@"Dominio". Es decir, que si trabajo en la empresa xyz mi correo será probablemente bdevallejo@xyz.com. No tiene mucho misterio el crear una lista con usuarios probablemente válidos. Y si quieres crear la lista y tienes acceso a un ordenador que está dentro de un dominio, basta con hacer `net user /domain` en Powershell para ver la lista completa de los usuarios. 

Para los passwords tenemos un problema algo mayor. Y es que Windows tiene unas políticas de seguridad que no permiten utilizar un password cualquiera ( 7 caracteres mínimo, cierta complejidad, obligación de renovarlo cada cierto tiempo... ). Así que lo más realista sería crear un crear un diccionario personalizado de contraseñas con [Cewl](https://github.com/digininja/CeWL), tras haber estudiado a nuestra hipotética víctima. antes de ejecutar el ataque con `crackmapexec`. En mi caso, todas las contraseñas que he utilizado para generar usuarios en el Directorio Activo están incluídas en `SecLists` ( más concretamente en /Passwords/Permutations).

Una vez tenemos la lista de usuarios y la de contraseñas, lanzamos crackmapexec para intentar validarlo. 

```
crackmapexec smb <IPDELDOMAINCONTROLLER> -u users.txt -p password.txt
```

Esto iniciará un ataque de fuerza bruta contra el Domain Controller hasta que encuentre un usuario y una contraseñas válidas.

<img src="../../../../assets/images/ad-crackmapexec/login_success.png"
     alt="Ataque por fuerza bruta"
     style="float: center; margin-right: 10px; " />

En este caso hemos dado con un usuario sin privilegios. Si hubiéramos dado con un Administrador, no aparecería la palabra `Pwned!`.

# Defensa <a id="defensa"></a>

Siempre imagino que este tipo de ataques por fuerza bruta haría saltar las alarmas de Windows por defecto. Lo normal es que un Domain Controller esté bien vigilado y monitorizado. Pero, por defecto no ocurre así. Por tanto, merece la pena emplear un poco de tiempo en fortificar un directorio activo para que un ataque tan sencillo de ejecutar como este no sea ignorado. 

Aunque esperaba encontrar un montón de logs con el [Event ID 4770 o 4771](https://docs.microsoft.com/es-es/windows/security/threat-protection/auditing/event-4771), correspondientes a errores a la hora de hacer login, lo cierto es que estos errores tienen que ver con Kerberos y por tanto se tratan de forma separada.

Sin embargo, el Instance 4625 se corresponde a [no se puede iniciar sesión en una cuenta](https://docs.microsoft.com/es-es/windows/security/threat-protection/auditing/event-4625) y filtrando por ahí salen un montón de erores como se puede comprobar. Así que ya sabéis, si quereís monitorizar el `password spraying` con crackmapexec, vigilad el  4625.

```
Get-EventLog -LogName Security -InstanceId 4625
```

<img src="../../../../assets/images/ad-crackmapexec/AD_log.png"
     alt="Logs del ataque por fuerza bruta"
     style="float: center; margin-right: 10px; " />
