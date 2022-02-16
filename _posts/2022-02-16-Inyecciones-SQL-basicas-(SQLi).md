---
layout: post
title: Hello World
description: blablabla
tags: [ write-ups , seguridad , sqli ]
---

Con este post inauguro un blog. Un blog que no sé dónde me llevará, ni hasta dónde seré capaz de actualizar. Pretendo simplemente llevar un registro de todo lo que aprendo en el mundo de la informática. Ciberseguridad, DevSecOps, DevOps, snippets de código y cosas por el estilo serán el foco principal.

Allá voy.

##Introducción a SQLi

En el post de hoy voy a tratar de indagar en las inyecciones SQL. Unas de las vulnerabilidades de las que más se habla, y posiblemente una de las más peligrosas, ya que puede exponer todos los datos de nuestro servidor.

El caso más típico y más sencillo es la manipulación de un request cualquiera. Imaginemos una página cuya URL es algo como

`https://mi-tienda.com/productos?categoria=Coches`

Coches es el parámetro que se utilizará en el código SQL para buscar todos los productos que corresponden a esa categoría, con un comando similar a este:

`SELECT * FROM productos WHERE categoria = 'Coches'`

Si el comando no está saneado de la forma correcta, ( y espero escribir otro post pronto sobre ello ), podríamos obtener mucha más información simplemente escribiendo en la URL parámetros que se traduzcan en comandos SQL. Muchas de las inyecciones SQL son sencillas de ejecutar, pero para visualizarlo mejor, 
voy a utilizar los [Labs de PortSwigger](https://portswigger.net/web-security/all-labs), ya que tienen muchos niveles, son gratuítos y por ende son perfectos para iniciarse en las inyecciones SQL (SQLi).

###Lab #1: SQL injection vulnerability in WHERE clause allowing retrieval of hidden data.

Tal y como aparece en la descripción, el lab es sensible a inyecciones SQL. Nada más entrar, navego por las diferentes categorías, y compruebo que la url cambia el parámetro _category_. Por tanto, el comando SQL que se ejecuta es algo como

`SELECT * FROM productos WHERE category = 'Corporate+gifts' AND released = 1`

Siendo released=1 los productos que se pueden mostrar. Pues bien, con añadir sencillamente  un 

`'+OR+1=1–` a la url, es decir: `https://.../filter?category=Corporate+gifts’+OR+1=1–`, el comando debería cambiar a algo así como: 

`SELECT * FROM products WHERE category = 'Corporate+gifts' OR 1=1--'`

Sabiendo que 1=1 va a retornar siempre verdadero (true), la inyección nos devolverá todos los productos de todas las categorías.

**Solución:** `https://.../filter?category=Corporate+gifts’+OR+1=1–`

###Lab #2: SQL Injection vulnerability allowing login bypass

En la descripción nos dicen que el lab contiene una vulnerabilidad en el _login_, y que debemos intentar acceder como administradores (nombre de usuario _administrator_).

Pues bien, el sistema será parecido. Con sólo añadir un `'--` al usuario _administrator_ el comando que enviaremos a la base de datos será algo como

`SELECT * FROM users WHERE username = 'administrator'--' AND password = ' '`

Es decir, nos saltamos la parte en la que el comando comprueba el _password_. 

**Solución:** nuestro nombre de usuario será `administrator'--` y el password será `' '`



##Ataques SQLi usando el comando UNION

El comnado  UNION nos es útil a la hora de realizar ataques SQL. 
Por ejemplo, nos pueden servir para obtener información de la tabla sobre la que estamos ejecutando un comando y sobre otras tablas. Básicamente, un comando UNION nos permite extender el comando SELECT de manera que un comando como:

SELECT x , y FROM productos UNION SELECT a , b FROM clientes

Nos devolvería dos columnas con los valores x e y además de otras dos columnas con los valores a y b. El problema al que a menudo nos enfrentamos es que ambos comandos han de tener el mismo número de columnas.

###Lab #3 : 


## Some great heading (h2)

Proin convallis mi ac felis pharetra aliquam. Curabitur dignissim accumsan rutrum. In arcu magna, aliquet vel pretium et, molestie et arcu.

Mauris lobortis nulla et felis ullamcorper bibendum. Phasellus et hendrerit mauris. Proin eget nibh a massa vestibulum pretium. Suspendisse eu nisl a ante aliquet bibendum quis a nunc. Praesent varius interdum vehicula. Aenean risus libero, placerat at vestibulum eget, ultricies eu enim. Praesent nulla tortor, malesuada adipiscing adipiscing sollicitudin, adipiscing eget est.

### Blockquotes (h3)

Praesent varius interdum vehicula. Aenean risus libero, placerat at vestibulum eget, ultricies eu enim. Praesent nulla tortor, malesuada adipiscing adipiscing sollicitudin, adipiscing eget est.

> This quote will _change_ your life. It will reveal the <i>secrets</i> of the universe, and all the wonders of humanity. Don't <em>misuse</em> it.

### Code blocks (h3)

Vestibulum lacus tortor, ultricies id dignissim ac, bibendum in velit. Proin convallis mi ac felis pharetra aliquam. Curabitur dignissim accumsan rutrum.

```javascript
function sayHello(name) {
  if (!name) {
    console.log("Hello World");
  } else {
    console.log(`Hello ${name}`);
  }
}
```

In arcu magna, aliquet vel pretium et, molestie et arcu. Mauris lobortis nulla et felis ullamcorper bibendum. Phasellus et hendrerit mauris.

##### Inline code, `pacman` (h5)

In arcu magna, aliquet vel pretium et, molestie et arcu. Mauris lobortis nulla et felis ullamcorper bibendum. Phasellus et hendrerit mauris.

### Oh hai, an unordered list!!

In arcu magna, aliquet vel pretium et, molestie et arcu. Mauris lobortis nulla et felis ullamcorper bibendum. Phasellus et hendrerit mauris.

- First item, yo
- Second item, dawg
- Third item, what what?!
- Fourth item, fo sheezy my neezy

### Oh hai, an ordered list!!

In arcu magna, aliquet vel pretium et, molestie et arcu. Mauris lobortis nulla et felis ullamcorper bibendum. Phasellus et hendrerit mauris.

1. First item, yo
2. Second item, dawg
3. Third item, what what?!
4. Fourth item, fo sheezy my neezy
