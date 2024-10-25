---
layout: post
title:  "Home Assistant y el bebé"
description: "Cositas que han ayudado en casa"
date:   2024-09-04 13:00:00 +0200
render_with_liquid: false
---

Tenemos un nuevo miembro de la familia en casa y han surgido algunas necesidades que si no se resuelven de manera friki, no vale la pena resolverlas.
A saber:

	- Recordatorios de cuando a dormido o comido la bebé.
	- Recordatorios de medicamentos.
	- Reproducir ruidos blancos en las habitaciónes.

Ya he hablado [en alguna ocasión](https://blog.morettigiuseppe.com/2023/01/27/control-calefaccion-homeassistant.html) que tengo Home Assistant en casa para todo lo de domotica, asi que vamos al grano con que automatizaciónes nuevas tenemos. Home assistant es gratuito y super flexible para cualquier tarea. Puede hacer lo que le pidas (algunas cosas mas o menos dificil de programar).

## Recordatorios

Para los recordatorios, primero hace falta recolectar datos. Lo mejor que hay en el mundillo open source domótica para instalar en casa es [BabyBuddy](https://docs.baby-buddy.net/). Ofrece una interfaz para marcar comidas y siestas, etc, se puede instalar en Home Assistant como un Add-on, asi que es relativamente facil, y se puede integrar con Home Assistant con sensores que indican cuando y cuanto fueron los últimos eventos. Esto ultimo es lo que nos interesa para los recordatorios.

Necesitamos un recordatorio para cuando ha pasado mucho rato de la última comida y un recordatorio para última siesta.

## Ultima comida

Para esto usamos una Automation con lo siguiente

```yaml

alias: Send feed alert when treshold
description: ""
trigger:
  - platform: state
    entity_id:
      - sensor.baby_last_feeding
    attribute: start
condition: []
action:
  - variables:
      variableDelay: |
        {% if  now() > today_at("06:00") and now() < today_at("23:00")  %}
          {{timedelta(hours=2, minutes=00)}} 
        {% else %}
          {{timedelta(hours=3, minutes=30)}} 
        {% endif %}
  - delay: >-
      {{  variableDelay | as_timedelta - (now() -
      state_attr("sensor.baby_last_feeding", "start") | as_datetime) }}
  - variables:
      criticalLoudNotificationTime: >
        {% if  now() > today_at("06:00") and now() < today_at("23:00")  %}1{%
        else %}0{% endif %}
      message: >-
        El bebe comió {{states.sensor.baby_last_feeding.state}}ml hace
        {{variableDelay}}h.
      title: Baby feed updates
  - action: notify.mobile_app_iphone
    metadata: {}
    data:
      title: "{{title}}"
      data:
        push:
          sound:
            name: default
            critical: |
              {{ criticalLoudNotificationTime }}
            volume: |
              {{ criticalLoudNotificationTime }}
      message: "{{ message }}"
  - action: notify.mobile_app_iphone
    data:
      data:
        push:
          sound:
            name: default
            critical: |
              {{ criticalLoudNotificationTime }}
            volume: |
              {{ criticalLoudNotificationTime }}
      title: "{{title}}"
      message: "{{ message }}"
mode: restart


```

Lo interesante aqui es **variableDelay** que nos indica cuanto vamos a esperar de la ultima comida dependiendo de noche o día 2 o 3 horas. Despues en **delay** seteamos cuando esperará la automatización antes de enviar la notificación al telefono. Esto el delay de 2h menos el tiempo que ya ha pasado desde el comienzo de la comida (teniendo en cuenta que el registro se hace normalmente al final de la comida).

Luego lo interesante de las noticaciónes (para configurar los moviles se instala la aplicación de home assistant en los teléfonos) es ponerlas en CRITICO. Con esto aunque el teléfono este en silencio suena a todo volumen igual. Por eso también hay un control para no poner en critico durante la noche.

Y con esto tenemos un aviso en el movil cuando ha pasado mucho rato desde la útlima comida.

## Ultima siesta

En control de sueño es más sencillo en tanto que nos interesa el tiempo pasado desde que terminó la ultima siesta, que es el dato ya como viene desde babyBuddy, asi que solo hay que contar desde que se actualiza el último sueño y ya.

```yaml

alias:  sleep window alert
description: ""
trigger:
  - platform: state
    entity_id:
      - sensor.baby_last_sleep
    attribute: end
condition:
  - condition: time
    after: "06:00:00"
    before: "21:00:00"
action:
  - delay:
      hours: 1
      minutes: 15
      seconds: 0
      milliseconds: 0
  - action: notify.mobile_app_iphone_giuseppe
    metadata: {}
    data:
      message: Hace 1h 15m de su ultima siesta
      title: Baby's sleep updates
      data:
        push:
          sound:
            name: default
            critical: 1
            volume: 1
  - action: notify.mobile_app_iphone
    data:
      message: Hace 1h 15m de su ultima siesta
      title: Baby's sleep updates
      data:
        push:
          sound:
            name: default
            critical: 1
            volume: 1
mode: restart


```

## Maquina de ruido blanco

Pensé en hacerlo como un dispositivo independiente, pero luego pensé, tengo los altavoces de Google Home Mini, ya conectado en las habitaciónes para poner musica. Y estos estan conectados a Home Assistant, osea puedo mandar audio a los altavoces en atomatizaciónes.

Entonces, la solución fue, crear un dashboard en HA con una botonera que mande los diferentes sonidos a los altavoces. Muy simple. Aunque con un detalle. No hay una funcionalidad de repetir automaticamente un audio. Entonces para reproducir un ruido blanco toda noche es un problema. Si son cortos se pueden subir a la sección de MEDIA de home assistant sin problemas (o desde WINSCP porque la interfaz tiene problemas con ficheros grandes). 

Para ficheros más grandes lo que hice fue duplicar el audio hasta hacerlos de 8 horas (500MB :'D ) y subirlo a mi servidor multimedia. Se puede a cualquier sitio que quede un link publico que home assistant pueda acceder para mandar.

Vamos a ver la configuración en el dashboard.


```yaml

square: false
type: grid
cards:
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: media_player.play_media
      target:
        device_id: device_iddfdsfdsfdsfdsf
      data:
        media_content_id: >-
          http://192.168.86.32:8096/Items/519b4933158279d75b48c58b702f74e3/Download?api_key=64cc4e7783d94186b857064dcc292d6b
        media_content_type: audio/mp3
    name: White Noise
    entity: media_player.living_room_speaker
    icon: mdi:sine-wave
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: media_player.play_media
      target:
        device_id: device_id324324324324
      data:
        media_content_id: http://192.168.22.33:8123/media/local/ocean-sounds-1.mp3
        media_content_type: music
    name: Ocean
    entity: media_player.living_room_speaker
    icon: mdi:waves
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: media_player.play_media
      target:
        device_id: device_iddfsfsfdsfdsfd
      data:
        media_content_type: music
        media_content_id: http://192.168.22.33:8123/media/local/rain-sounds-1.mp3
    entity: media_player.living_room_speaker
    icon: mdi:grain
    name: Rain
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: media_player.play_media
      target:
        device_id: device_idvdsvfdvfdvfd
      data:
        media_content_id: http://192.168.22.33:8123/media/local/hypno-baby-1.mp3
        media_content_type: music
    name: Lullaby
    entity: media_player.living_room_speaker
    icon: mdi:cradle
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: media_player.media_stop
      target:
        device_id: device_idvfsdvfsvfsvfd
        entity_id: media_player.living_room_speaker
      data: {}
    name: Stop
    entity: media_player.living_room_speaker
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: light.turn_on
      target:
        device_id: device_iddfdsfdsfds
      data:
        brightness_pct: 2
    entity: number.wled_intensity_2
    name: Dormir
    icon: mdi:weather-night
    hold_action:
      action: toggle
  - show_name: true
    show_icon: true
    type: button
    tap_action:
      action: perform-action
      perform_action: light.turn_on
      target:
        device_id: device_idfdsfdsvfdsvs
      data:
        brightness_pct: 24
    entity: number.wled_intensity_2
    icon: mdi:human-baby-changing-table
    hold_action:
      action: toggle
    name: Cambiar
  - type: entity
    entity: sensor.atc_dc39_temperature
    name: T
columns: 3
title: TV Room

```

Y la botonera quedaría asi, accesible desde el movil.

![ha1.png]({{ site.baseurl }}/assets/images/ha1.png) 

Hay botones extras para control de luces y un visor de temperatura para la habitación.

Destacable la primera url es el white noise largo de 8h que viene de jellyfin y las otras URL son locales de HA de ficheros subidos a MEDIA.

Es más facil configurar en UI que como se ve aqui en YAML.

Eso es todo por ahora, pero creu que se vienen más automaticaciónes pronto. Estaría bien integrar el asistente de Voz de HA a ver, aunque no se puede activar desde los Home Minis.


**Cualquier comentario o sugerencia se puede dejar en [este thread del fediverso]()**