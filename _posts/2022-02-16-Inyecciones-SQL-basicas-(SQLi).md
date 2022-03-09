---
layout: post
title: Inyecciones SQL básicas (SQLI)
description: Introducción a las inyecciones SQL (SQLI) y write ups de algunos labs de Portswigger 
tags: [ write-ups , seguridad , SQLi, portswigger ]
---

# Indice
1. [Introducción a SQLi](#introduction)

    1. [Lab #1: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data.](#lab1)
    2. [Lab #2: SQL Injection vulnerability allowing login bypass](#lab2)

2. [Ataques SQLi usando el comando UNION](#comandounion)
    1. [Lab #3 : SQL injection UNION attack, determining the number of columns returned by the query](#lab3)
    2. [Lab #4 : SQL injection UNION attack, finding a column containing text](#lab4)
    3. [Lab #5 : SQL injection UNION attack, retrieving data from other tables](#lab5)
    4. [Lab #6 : SQL injection UNION attack, retrieving multiple values in a single column](#lab6)
3. [Links](#links)    


Con este post inauguro un blog. Un blog que no sé dónde me llevará, ni hasta dónde seré capaz de actualizar. Pretendo simplemente llevar un registro de todo lo que aprendo en el mundo de la informática. Ciberseguridad, DevSecOps, DevOps, snippets de código y cosas por el estilo serán el foco principal.

Allá voy.

# Introducción a SQLi<a id="introduction"></a>

En el post de hoy voy a tratar de indagar en las inyecciones SQL. Unas de las vulnerabilidades de las que más se habla, y posiblemente una de las más peligrosas, ya que puede exponer todos los datos de nuestro servidor.

El caso más típico y más sencillo es la manipulación de un request cualquiera. Imaginemos una página cuya URL es algo como

`
https://mi-tienda.com/productos?categoria=Coches
`

Coches es el parámetro que se utilizará en el código SQL para buscar todos los productos que corresponden a esa categoría, con un comando similar a este:

`SELECT * FROM productos WHERE categoria = 'Coches'`

Si el comando no está saneado de la forma correcta, ( y espero escribir otro post pronto sobre ello ), podríamos obtener mucha más información simplemente escribiendo en la URL parámetros que se traduzcan en comandos SQL. Muchas de las inyecciones SQL son sencillas de ejecutar, pero para visualizarlo mejor, 
voy a utilizar los [Labs de PortSwigger](https://portswigger.net/web-security/all-labs), ya que tienen muchos niveles, son gratuítos y por ende son perfectos para iniciarse en las inyecciones SQL (SQLi).



## Lab #1: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data.<a id="lab1"></a>

Tal y como aparece en la descripción, el lab es sensible a inyecciones SQL. Nada más entrar, navego por las diferentes categorías, y compruebo que la url cambia el parámetro _category_. Por tanto, el comando SQL que se ejecuta es algo como

`SELECT * FROM productos WHERE category = 'Corporate+gifts' AND released = 1`

Siendo released=1 los productos que se pueden mostrar. Pues bien, con añadir sencillamente  un 

`'+OR+1=1–` a la url, es decir: `https://.../filter?category=Corporate+gifts'+OR+1=1–`, el comando debería cambiar a algo así como: 

`SELECT * FROM products WHERE category = 'Corporate+gifts' OR 1=1--'`

Sabiendo que 1=1 va a retornar siempre verdadero (true), la inyección nos devolverá todos los productos de todas las categorías.

**Solución:** 

`https://.../filter?category=Corporate+gifts’+OR+1=1–`



## Lab #2: SQL Injection vulnerability allowing login bypass <a id="lab2"></a>

En la descripción nos dicen que el lab contiene una vulnerabilidad en el _login_, y que debemos intentar acceder como administradores (nombre de usuario _administrator_).

Pues bien, el sistema será parecido. Con sólo añadir un `'--` al usuario _administrator_ el comando que enviaremos a la base de datos será algo como

`SELECT * FROM users WHERE username = 'administrator'--' AND password = ' '`

Es decir, nos saltamos la parte en la que el comando comprueba el _password_. 

**Solución:** 
nuestro nombre de usuario será `administrator'--` y el password será `' '`




# Ataques SQLi usando el comando UNION<a id="comandounion"></a>

El comnado  UNION nos es útil a la hora de realizar ataques SQL. 
Por ejemplo, nos pueden servir para obtener información de la tabla sobre la que estamos ejecutando un comando y sobre otras tablas. Básicamente, un comando UNION nos permite extender el comando SELECT de manera que un comando como:

SELECT x , y FROM productos UNION SELECT a , b FROM clientes

Nos devolvería dos columnas con los valores x e y además de otras dos columnas con los valores a y b. El problema al que a menudo nos enfrentamos es que ambos comandos han de tener el mismo número de columnas.



## Lab #3 : SQL injection UNION attack, determining the number of columns returned by the query <a id="lab3"></a>

El reto consiste precisamente en buscar el número de columnas de la tabla category:

Nada más abrir la página del lab, hago click en cualquiera de las categorías (en mi caso Food & Drink). Tan sólo hay que añadir UNION SELECT y algún valor que no genere ninguna respuesta inadecuada en el comando, para lo cual el parámetro NULL nos viene perfecto. Así que hemos de añadir ‘+UNION+SELECT+NULL– a la url. Si todo va bien, nos devolverá una página con el status 200. De no ser así, habrá que ir añadiendo ,NULL hasta que funcione, y así sabremos el número de columnas de la tabla. 

**Solución:**  
`https://..?category=Food+%26+Drink'+UNION+SELECT+NULL,NULL,NULL--`

Es decir, 3 columnas. 



## Lab #4 : SQL injection UNION attack, finding a column containing text<a id="lab4"></a>

De nuevo, nuestro lab contiene una vulnerabilidad en category. Ya sabemos que hay 3 columnas, pero ahora debemos averigüar qué columna es de tipo string.

Con lo que hemos visto hasta ahora, resulta sencillo. Simplemente habría que ir reemplazando los NULL del ejercicio anterior por un parámetro de tipo string (de tipo texto) , por ejemplo ‘a’ (una vez dentro del lab hay una pequeña descripción que dice que tenemos que hacer que la base de datos nos devuelva ‘BVWP3H’)

**Solución:** 
`https://...filter?category=Clothing%2c+shoes+and+accessories'+UNION+SELECT+NULL,'BVWP3H',NULL–`

Es decir, la segunda columna es de tipo string. 

Con esta información podemos crear una petición a nuestra medida y rellenar la respuesta con información de otras tablas mediante el parámetro UNION. Es decir, añadir a la petición legítima otra petición que devuelve el mismo tipo de valor pero dirigida a una tabla diferente. 

El siguiente reto ayuda a entenderlo mejor.



## Lab #5 : SQL injection UNION attack, retrieving data from other tables<a id="lab5"></a>

Se trata de un lab similar a los dos anteriores, pero en esta ocasión hay que acceder a una tabla users que a su vez contiene dos columnas, username y password. En la vida real podría tratarse de un ejemplo totalmente válido, ya que muchas bases de datos tienen una tabla similar. Pero recomiendo echar un ojo a este enlace si lo que queremos es saber qué tablas existen en la base de datos y qué columnas tienen.

Empiezo inyectando un ataque mediante UNION añadiendo '+UNION+SELECT+NULL– a la url. No funciona. Añado otro NULL más y ahora sí. 

`https://…/filter?category=Gifts'+UNION+SELECT+NULL,NULL– `

Funciona. Hay dos columnas. 

Deduzco que ambas son de tipo string pero por si acaso, compruebo. 

`https://..filter?category=Gifts'+UNION+SELECT+'a','b''–`

Todo correcto. Tan sólo queda reemplazar los valores ‘a’ y ‘b’ del parámetro anterior por username y password y añadir la tabla sobre la que queremos lanzar la búsqueda (FROM users en este caso). 

**Solución:** 
`https://acb31fdf1efcae16c02c9680008d005d.web-security-academy.net/filter?category=Gifts'+UNION+SELECT+username,password+FROM+users–`

Y listo. En la respuesta se añadieron los usuarios wiene, administrator y carlos. Copio el password de administrator para hacer log in y reto superado.

Ahora bien, ¿qué occuriría si la tabla sobre la que quiero ejecutar mi petición sólo dispone de una columna y yo quiero obtener información de dos columnas? El siguiente reto va precisamente de eso:



## Lab #6 : SQL injection UNION attack, retrieving multiple values in a single column<a id="lab6"></a>

Una vez más, empiezo inyectando un ataque mediante UNION añadiendo '+UNION+SELECT+NULL– a la url. No funciona. Añado otro NULL más y ahora sí. 

`https://…/filter?category=Gifts'+UNION+SELECT+NULL,NULL– `

Funciona. Hay dos columnas. 

Ahora bien, en esta ocasión creo que sólo una de ellas será de tipo string. Lo compruebo con 

`https://ac211fd71f9c8c3bc02c4a59002a009f.web-security-academy.net/filter?category=Corporate+gifts'+UNION+SELECT+'a','b'–`

que no funciona. Cambio a :

```https://ac211fd71f9c8c3bc02c4a59002a009f.web-security-academy.net/filter?category=Corporate+gifts'+UNION+SELECT+'a',NULL–```

Y tampoco. Y por fin:
```https://ac211fd71f9c8c3bc02c4a59002a009f.web-security-academy.net/filter?category=Corporate+gifts'+UNION+SELECT+NULL,'a'–```

Me da una respuesta válida. ¿Y ahora, cómo obtengo el username y password en una sola petición? Mi primer instinto es hacer dos separadas, una para el username y otra para el password. Y efectivamente, funciona. Obtengo los nombres de usuario y los passwords por separado, pero no sería difícil combinarlos ( hay sólo 3 ). 

El ejercicio sin embargo no va de eso, así que investigo un poco más. En este enlace vienen algunos ejemplos de string concatenation útiles. Pruebo con el primero para bases de datos Oracle y bingo. 

**Solución:** 
`https://../filter?category=Corporate+gifts'+UNION+SELECT+NULL,username||password+FROM+users–`

Retorna el administrador junto al password. Hago log in y reto superado. 

#  Lab #7 :(SQL injection attack, querying the database type and version on Oracle)<a id="lab7"></a>

Ahora vamos a intentar averigüar el tipo de base de datos y la versión, lo cual puede ser el inicio de un vector de ataque.

Para ello estos dos enlaces pueden ser de gran utilidad. Necesitamos obtener la version y el nombre de la base de datos, y tal como aquí aparece esto se define como [BANNER en Oracle](https://docs.oracle.com/en/database/oracle/oracle-database/18/refrn/V-VERSION.html)

Por otra parte, hay que recordad que los ataques a una base de datos de Oracle hay que especificar una tabla desde la que obtener los resultados. Es decir, una tabla tras el parámetro FROM. Por ejemplo: UNION SELECT ‘abc’ FROM tabla.  Y tal y como aparece aquí, las diferentes bases de datos difieren un poco en la manera de llamar a la versión. https://portswigger.net/web-security/sql-injection/examining-the-database

Tras realizar los pasos anteriores para definit el número de columnas, y teniendo en cuenta todo lo anterior (y que estamos ante una base de datos de ORACLE) , la inyección resulta relativamente sencilla: 

https://acae1fd31e0f14cac0487b430057003d.web-security-academy.net/filter?category=Food+%26+Drink%27+UNION+SELECT+BANNER,null+FROM+v$version–



# Links <a id="links"></a>
Aquí dejo un par de links interesantes para más información.

https://owasp.org/www-community/attacks/SQL_Injection_Bypassing_WAF
https://portswigger.net/web-security/sql-injection/cheat-sheet

