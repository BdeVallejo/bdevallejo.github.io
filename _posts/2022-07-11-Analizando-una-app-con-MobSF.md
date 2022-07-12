---
layout: post
title: Analizando una App cualquiera con MobSF
description: Análisis de de una App con MobSF y descubriendo cómo se comparten nuestros datos.
tags: [ Android , App, MobSF, seguridad, testing, analisis ]
---

# Indice

1. [Introducción](#introduction)

2. [Instalación de MobSF y ApkExtractor](#instalacion)
     1. [MobSF](#mobsf)
     2. [ApkExtractor](#apkextractor)

3. [Análisis estático](#estatico)
     1. [Permisos](#permisos)
     2. [Trackers](#trackers)
     3. [Vulnerabilidades](#vulnerabilidades)

4. [Análisis dinámico](#dinamico)
     1. [URLs](#URLs)
     2. [Cookies](#cookies)
     3. [Continuara](#continuara)

# Introducción<a id="introduction"></a>

En las últimas semanas me ha dado por investigar el uso de [MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF). Para quien no lo conozca, MobSF (Mobile Security Framework) es un framework de análisis de aplicaciones móviles increíblemente completo desarrollado por [Ajin Abraham](https://github.com/ajinabraham). 

En este tiempo estoy asombrado por varios motivos. En primer lugar, encuentro lo que parecen vulnerabilidades y fallos de programación en muchas aplicaciones a priori desarrolladas por un gran equipo y con un buen presupuesto detrás. Digo a priori porque MobSF cataloga algunos fallos como "High risk", pero a decir verdad no he tenido mucho tiempo de explorar todo bien a fondo.

En segundo lugar, porque MobSF evidencia lo que ya sabía: en internet nada es gratis. La cantidad de datos que se comparten mediante aplicaciones móviles es incluso mayor que la que compartimos mediante apliaciones web. Geolocalización, información de otras aplicaciones, acceso a almacenamiento... 

Como muestra, un botón. En el siguiente post voy a analizar de forma superficial un juego. No diré el nombre de la app, pero sí que diré que es un juego bastante común y popular, con más de 1 millón de descargas.

# Instalación de MobSF y ApkExtractor<a id="instalacion"></a>

## Instalación de MobSF<a id="mobsf"></a>

[MobSF](https://github.com/MobSF/Mobile-Security-Framework-MobSF) es, como he dicho antes, un framework de análisis de apliaciones móviles y de código abierto. Para instalarlo, primeramente clonaremos el repositorio de Github:

````
git clone https://github.com/MobSF/Mobile-Security-Framework-MobSF
````
A continuación, y dado que su instalación dependerá de nuestro sistema operativo, os dejo el enlace a la [documentación oficial](https://mobsf.github.io/docs/#/installation). 

IMPORTANTE: comprueba que instalas antes todos los [paquetes requeridos](https://mobsf.github.io/docs/#/requirements).

## Instalación de ApkExtractor<a id="apkextractor"></a>

Por otra parte, es necesario instalar ApkExtractor en el móvil. Para este ejemplo utilizaré `Android`, ya que es más sencillo desempaquetar sus aplicaciones y en principio cuenta con menos restricciones que `ios` .

Así que tan sólo hace falta ir a la Play Store y descargar [ApkExtractor](https://play.google.com/store/apps/details?id=com.ext.ui&hl=es&gl=US).

# Análisis estático<a id="estatico"></a>

Una vez instalado todo lo necesario, abrimos APK Extractor y seleccionamos la aplicación que nos interesa. Se creará un archivo `.apk` que debemos copiar a nuestro ordenador. La manera más sencilla es subirlo a Google Drive o similar, ya que si queremos pasárnoslo mediante Email o WhatApp nos dará problemas.

Una vez tengamos el archivo .apk en nuestro ordenador, vamos a la carpeta "Mobile-Security-Framework-MobSF" que hemos clonado y arrancamos con 

````
./run.sh
````

En nuestro navegador iremos a `localhost:8000` donde tendremos ya MobSF listo para analizar. Tan sólo hay que arrastrar el archivo .apk y se ejecutará un análisis de forma automática.

El primer análisis es estático. En él se detallan las vulnerabilidades y los riesgos asociados a la instalación de la aplicación. Primera sorpesa: el juego que analizo (insisto, descargado más de 1 millón de veces) tiene un `Security Score` de 32/100. Es decir, MobSF la cataloga como  `High Risk`. 

## Permisos<a id="permisos"></a>

A continuación, hecho un vistazo a los permisos. 9 de los 30 que se requieren están catalogados como `dangerous`. Lo más curioso es que muchos son absolutamente innecesarios para la ejecución del juego. Por ejemplo, la localización, el acceso a la cámara, al calendario, o al estado del teléfono ( tal y como se describe, esto implica el acceso al número de serie, a si hay una llamada activa, etcétera...) no están ni remotamente relacionados con el desarrollo del juego. Por tanto, la opción que se me ocurre es que estos datos se recopilan únicamente para ser vendidos.

<img src="../../../../assets/images/Analizando-app-MobSF/Permissions.png"
     alt="Una captura de los permisos catalogados como dangerous"
     style="float: center; margin-right: 10px; " />

## Trackers<a id="trackers"></a>

Otro de los aspectos que me llaman la atención es la cantidad de trackers (11 en total), que se utilizan en una aplicación de este tipo. La mayoría son de Analytics, aunque este término puede resultar algo ambiguo. ¿Analytics de qué y para qué?

También hay 5 dedicados a Advertisement. De algo tienen que vivir las aplicaciones, por supuesto. Pero cuanto menos me resulta curioso la cantidad de servicios que están "atentos" a todo lo que hacemos con la aplicación.

<img src="../../../../assets/images/Analizando-app-MobSF/Trackers.png"
     alt="Trackers de analisis y advertisement en su mayoria"
     style="float: center; margin-right: 10px; " />


## Vulnerabilidades <a id="vulnerabilidades"></a>

Paso ahora a echar un vistazo al apartado de `Security Analysis`. En primer lugar, en el análisis del `Manifest.xml`, MobSF encuentra 10 vulnerabilidades catalogadas como `high`. 

Como desarrollador he hecho mis pinitos con aplicaciones Android y Android estudio, y sé que es una verdadera pesadilla el actualizar una aplicación con la montaña de actualizaciones y de librerias que hoy Google da por buenas y mañana por `deprecated`. Por otra parte, no es esta una app cualquiera, sino una muy popular, y que (espero) cuenta con un equipo detrás. Y por tanto algunos fallos como que se envíe el tráfico en `Clear Text` si bien no son demasiado alarmantes ( en la aplicación no se comparten datos comprometedores ni se realizan transferencias de dinero ), son un poco sorprendentes.

<img src="../../../../assets/images/Analizando-app-MobSF/Cleartext.png"
     alt="Clear text traffic is enabled"
     style="float: center; margin-right: 10px; " />

Me dirijo ahora a la parte de análisis del código. Tan sólo 3 problemas catalogados como `high`. Uno de ello me la atención: "Insecure WebView Implementation. WebView ignores SSL Certificate errors and accept any SSL Certificate. This application is vulnerable to MITM attacks". Investigo un poco y descubro que hay un [hilo en StackOverflow al respecto](https://stackoverflow.com/questions/36050741/webview-avoid-security-alert-from-google-play-upon-implementation-of-onreceiveds). El hilo es de hace 6 años y ya dicen que Google te alerta si aceptas cualquier certificado en tu app. Viendo el código, parece que la aplicacón sí que solventa ( al menos aparentenemente, no soy un gran experto ) esta vulnerabilidad avisando al usuario. 

````
@Override // android.webkit.WebViewClient
        public void onReceivedSslError(WebView view, final SslErrorHandler handler, SslError error) {
            AdvancedWebView advancedWebView;
            int primaryError = error.getPrimaryError();
            String str = primaryError != 0 ? primaryError != 1 ? primaryError != 2 ? primaryError != 3 ? "SSL Certificate error." : "The certificate authority is not trusted." : "The certificate Hostname mismatch." : "The certificate has expired." : "The certificate is not yet valid.";
            String url = error.getUrl();
            if (TextUtils.isEmpty(url) || ((advancedWebView = this.a) != null && advancedWebView.e(url))) {
                i a = i.a();
                a.b("SSL error at " + url + "\n" + str);
                AlertDialog.Builder builder = new AlertDialog.Builder(AdvancedWebView.this.a.get());
                builder.setTitle("SSL Certificate Error");
                builder.setMessage(url + "\n" + str + " Do you want to continue anyway?");
                builder.setPositiveButton("Continue", new DialogInterface$OnClickListenerC0118a(this, handler));
                builder.setNegativeButton("Cancel", new b(this, handler));
                builder.create().show();
                return;
            }
            handler.cancel();
        }
````

Lo cual me lleva a pensar que MobSF es un buen punto de inicio. Fantástico diría yo. Pero es posible que a veces se equivoque. Insisto: no soy un gran experto, por tanto ¡igual el que se equivoca soy yo!

Hay otra vulnerabilidad que sí que puedo comprobar: se utiliza AES-CBC con PKCS7, que como es bien sabido, es vulnerable a los ataques de tipo `Padding Oracle`. [Aquí se puede leer más](https://pulsesecurity.co.nz/articles/dotnet-padding-oracles). En el código veo varias veces esto:

````
Cipher cipher = Cipher.getInstance("AES/CBC/PKCS7Padding")
````
Supongo que tirando del hilo se podría llegar a saber para qué se utiliza este cifrado, pero creo que queda fuera del enfoque de este artículo así que decido pasar al análisis dinámico de la aplicación.

# Análisis dinámico<a id="dinamico"></a>

Paso ahora a realizar un análisis dinámico. Para ello utilizaré el Emulador de Android Studio, aunque también se puede utilizar Genymotion Android y Genymotion Cloud Android [tal y como detallan en la documentación](https://mobsf.github.io/docs/#/dynamic_analyzer). Para ello:

```
cd /Users/USER/Library/Android/sdk/emulator
 sudo ./emulator -avd MobSF2 -writable-system -no-snapshot

```
Y ya tengo el emulador en marcha. En mi localhost:8000 voy a `Dynamic Analyzer` y pincho en `Start Dynamic Analysis`. Este tipo de análisis da para un libro, así que voy a hacer lo más básico. Si algún día aprendo más y me resulta interesante, escribiré una segunda parte del post.

Esta parte también es algo más tediosa, ya que a veces la aplicación se detiene, o bien algunas pestañas tardan un poco en cargar. No todo puede ser perfecto.


Primeramente hago click en `Start Instrumentalization`. Juego y navego un poco por las opciones de la app en el emulador. Después hago click en `API Monitor` pero resulta bastante confuso. 

Así que pincho en `Generate Report`. Este proceso tarda unos minutos pero después se genera un reporte bastante interesante.

## URLs <a id="urls"></a>

En el apartado de URLs veo un sinfín de enlaces. Filtro por mi nombre de usuario y veo que varios enlaces contienen mi nombre y un número, de lo cual deduzco que es mi ID de usuario. Algunas referencias a Google Analytics y páginas similares con mi nombre de usuario, pero es tal la cantidad de URLs que dejo su estudio para más adelante. 

## Cookies <a id="cookies"></a>

Esta creo que es una de las parte más interesantes. Dentro del apartado `SQLITE DATABASE` encuentro una carpeta Cookies. Entro y me encuentro con un torrente de cookies. Muchas de ellas son JWT así que con jwt.io abro algunas. Y aquí aparece bien claro con quien se comparten nuestros datos. ¡

<img src="../../../../assets/images/Analizando-app-MobSF/jwt.png"
     alt="Captura de pantalla de JWT.io"
     style="float: center; margin-right: 10px; " />

## Continuará <a id="continuara"></a>

El siguiente paso en mi "investigación" es redirigir todo el tráfico a BurpSuite y analizar las peticiones que se realizan. Pero creo que eso da para otro post, así que sin duda habrá una segunda parte. Hasta entonces.
