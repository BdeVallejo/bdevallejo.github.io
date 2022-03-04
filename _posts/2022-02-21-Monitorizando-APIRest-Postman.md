---
layout: post
title: Monitorizando una API Rest con Postman
description: Tests sencillos para monitorizar una API Rest con Postman
tags: [ Postman , testing , API Rest ]
---

# Indice

1. [Introducción](#introduction)
2. [Ejemplos de la página de Postman](#ejemplos)
    1. [Validación de la respuesta (Schema validation)](#schema)
3. [Monitorizando el Stock mediante un For Loop](#stock)
4. [Probando la velocidad de respuesta](#velocidad)        

# Introducción<a id="introduction"></a>

Estoy creando una APIRest para un proyecto de una tienda virtual y antes de lanzarla voy a dedicarle un tiempo a probarla con Postman. Aquí narro mi experiencia y lanzo algunos consejos que pueden resultar útiles.

Si bien es cierto que algunas opciones de Postman son de pago, la versión gratuíta cumple de sobra con muchas de las necesidades. He realizado tan sólo algunos test básicos, pero las posibilidades son mucho más amplias.

# Ejemplos de la página de Postman <a id="ejemplos"></a>

En la web de Postman hay algunos [scripts interesantes](https://www.postman.com/postman/workspace/postman-api-monitoring-examples) que nos pueden servir de gran ayuda para comenzar. Están divididos en 5 secciones (Availability and Performance, JSON Schema Validation, Multi-Step Transaction, Check for Common API Vulnerabilities and Continuos API Testing). Seguro que volveré a esta sección en el futuro, pero de momento me interesa comprobar la validación de la respuesta. 


## Validación de la respuesta (Schema validation)<a id="schema"></a>

Entre los ejemplos disponibles en la web de Postman hay uno que me parece muy útil para comprobar la consistencia de los datos de la API, ya que es algo que en ocasiones conlleva un error y por tanto tiempo perdido en encontrar el fallo, al menos en mi caso. A veces una simple coma de más o de menos me lleva un buen rato localizar. 
 
Así que pruebo con una petición “GET” a mi  API. En la pestaña de Tests he personalizado el [Example 02](https://www.postman.com/postman/workspace/postman-api-monitoring-examples/request/13687875-b7b9e3ae-7427-4ac2-abb0-bad6e1953b94) de la página de ejemplos de Postman para así comprobar el JSON Schema que debería cumplir mi petición. 

Modifico los atributos para que se ajusten a mi API. Un pequeño ejemplo:

```
var schema = {
    "$schema": "http://json-schema.org/draft-07/schema",
    "$id": "http://example.com/root.json",
    "type": "object",
    "title": "My Schema",
    "description": "The root schema is the schema that comprises the entire JSON document.",
    "properties": {
        "id": {
            "type": "string"
        },
        "fields": {
            "type" : "object",
            "properties": {
                 "size": {
                     "type": "string"
                },
            "company": {
                "type": "string"
            },
            etcétera...       

             }
        }
    }
}
```
En definitiva, una forma un tanto manual pero sencilla de ver que mi API cumple con las especificaciones. 


# Monitorizando el Stock mediante un For Loop <a id="stock"></a>

En esta ocasión quiero crear algo un poco más útil, que me sirva para monitorizar el estado del stock de la tienda virtual que va a utilizar la API. Con un loop voy comprobando producto a producto el stock, de tal manera que si el stock de algún producto en la tienda virtual cuya API estoy desarrollando baja de 10 productos, el test falle. Por supuesto podría monitorizarlo de muchas otras maneras, pero me parece una forma cómoda y sencilla de hacerlo. Además, con la función monitor podemos automatizar el proceso aún más y comprobar de manera regular el stock de nuestra tienda, o cualquier otro parámetro. Incluso podríamos recibir un mail si la condición no se cumpliera.

```
const response = pm.response.json();
 
const item = response[0];
 
pm.test("Items en stock", () => {
   for (var i=0; i<response.length; i++) {
       pm.expect(response[i].stock).to.be.above(10);
   }   
 
})
```
 
# Probando la velocidad de respuesta<a id="velocidad"></a>

Otro caso para el cual Postman nos puede servir es para probar la velocidad de respuesta de nuestra API. Pongamos por ejemplo que el tiempo máximo de respuesta que queremos para nuestra API es de 500ms. Vamos a ejecutar un bloque de 100 peticiones con 200 milisegundos de diferencia. El script es tan simple como:
 
```
const respuesta = pm.response;
 
pm.test("Tiempo de respuesta", () => {
   pm.expect(respuesta.responseTime).below(500);
})
```
 
Una vez hecho esto, mandamos nuestra prueba al Runner y ahí configuramos el número de peticiones y el intervalo en milisegundos entre las mismas. 

Pero podemos hacer nuestro script algo más complejo y obtener el tiempo medio de respuesta, el tiempo máximo y el mínimo, por poner un ejemplo. 

```
const maxResponseTime = 500; //tiempo máximo de respuesta permitido
const ciclos = 50; //ciclos que vamos a ejecutar
const diff = 250;//250ms de diferencia entre ciclos
arrayRespuestas = []; // array donde iremos guardando tiempos de respuesta
 
i=0;
function request() {
  pm.sendRequest({
      url: pm.collectionVariables.get('BaseUrl'), // BaseUrl es la variable que hemos definido con la URL de nuestra API
      method: 'GET'
   }, (e, r) => {
         pm.test("Tiempo de respuesta  " + r.responseTime, () => {
                   pm.expect(r).to.have.property('code', 200);
                   arrayRespuestas.push(r.responseTime);
       });
       if (i < ciclos - 1) {     
           i++;
           setTimeout(request, diff);
       } else {
           maxMinAvgResponse = maxMinAvg(arrayRespuestas);
           pm.test("El maximo, mímimo y media de la respuesta son " + maxMinAvgResponse );
       }
   })
  
}
 
function maxMinAvg(arr) {
   var max = arr[0];
   var min = arr[0];
   var sum = arr[0];
   for (var i = 1; i < arr.length; i++) {
       if (arr[i] > max) {
           max = arr[i];
       }
       if (arr[i] < min) {
           min = arr[i];
       }
       sum = sum + arr[i];
   }
   return [max, min, sum/arr.length];
}
request();
```


