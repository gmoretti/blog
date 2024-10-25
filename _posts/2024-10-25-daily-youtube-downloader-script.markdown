---
layout: post
title:  "Programar descarga de videos de una pagina diariamente"
description: "con un script de Python, yt-dlp y CRON"
date:   2024-09-04 13:00:00 +0200
---

El contexto es el siguiente. Tengo una pagina tipo blog que lista posts de una cierta categoría. Cada enlace de esa lista abre el post y dentro siempre hay lo mismo. Un video embebido. Necesito descargar este video. El video puede ser de diferentes plataformas. Y se van añadiendo nuevos posts diariamente a esta lista.

Aquí esta claro. YT-DLP puede descargar la mayoria de links de plataformas solo pasando el link. Perfecto.

Pero podemos ir un pasito más allá. Podríamos diariamente checkear esta lista de posts y descargarlos.

Para el script vamos a usar python con beautifulSoup, para poder extraer las URLs de la lista y pasarselas a YT-DLP-

```python

import requests
from bs4 import BeautifulSoup
import subprocess, sys

res = requests.get('https://La-pagina.queTieneElListado/de/URLS/')

soup = BeautifulSoup(res.content, 'html.parser')

posts = soup.find_all(class_="post-title") #ESTA ES LA LINEA IMPORTANTE Aqui tenemos que identificar donde estarían las URL


for post in posts:
	link = post.contents[0].attrs['href']
	subprocess.run('./yt-dlp ' + link, shell = True, executable="/bin/bash")
      
```
Se guarda como miScript.py

Este script descara en el mismo directorio del script todas las URLs encontradas, y ademas, cortesía de yt-dlp si ya se ha descargado, no repite la descarga.
La parte que se puede complicar es la de find_all, donde dependiendo de como es la página podríamos tener problemas para navegar a la URL, en mi paso los tags html con clase post-title ya incluían la url que quería en un atributo HREF, esto lo miré en la documentación de beautifulSoup y está bastante claro.

Ahora, para la parte de programar la ejecución, asumiento que estamos en un entorno Linux (mi servidor de casa donde lo ejecuto, es linux) podemos usar Crontab, que es el
programador incluido por defecto y va perfecto.

Primero accedemos al crobtab de root (quizas no sea necesario el de root, pero tengo que guardar los ficheros en una carpeta protegida)

```bash
sudo crontab -u root  -e

```

Y añadir una línea como esta al final

```
30 8 * * * python3 /home/lagoyluz/scripts/miScript.py 2>&1 | logger -t mi-logger

```

aqui dependiendo de cuan recurrente queremos el checkeo, para mi caso una vez al día a las 8:30AM. Se puede usar una calculadora como https://crontab.cronhub.io/

Se puede remplazar mi-logger con cualquier nombre, esto se puede usar para revisar la salida de la ejeción. Ira bien para cuando las cosas no funcionen por algun motivo.

Para revisar el log:

```bash
grep mi-logger /var/log/syslog

```

Y eso es todo, es muy facil hacer tareas periodicas asi, sin ningun programa extra a parte de herramientas que se puede encontrar en cualquier maquina.


**Cualquier comentario o sugerencia se puede dejar en [este thread del fediverso](https://social.morettigiuseppe.com/o/0942110fcb0544fab61ba461c23a26e6)**