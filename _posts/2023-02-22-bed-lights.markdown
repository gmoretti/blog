---
layout: post_with_comments
title:  "Luces de lectura con Home Assistant, ESP32 y tiras de LEDs"
description: Control leds con un módulo ESP32, Home Assistant y una tira de LED direccionables
date:   2023-02-22 21:00:00 +0200
---

![myroom]({{ site.baseurl }}/assets/images/bed-lights/bedroom.jpg "my room")

Vamos a esconder las luces de noche de la habitación detrás de la cama y controlarlas por Home Assistant. Aprovecharemos la integración con home assistant para sincronizar el encendido con la alarma del iPhone y para tener un botón rápido de encendido y apagado

Este es un proyecto muy sencillo ya que todo el software y configuraciones se hacen en via interfaz y se utilizan los mismos 5V por USB del ESP32 para alimentar las luces.

# Componentes
Primero partes y componentes!

## Raspberry Pi 4 
Mi instalación de Home Assistant es en una raspberry, pero cualquier instalación serviría.
Se pueden encontrar instrucciones en la web de Home Assistant para la instalación. (Esto probablemente sería lo más complicado de conseguir, hay falta de stock, aunque se supone debería volver en 2023)

## ESP32
Cualquier ESP32 debería funcionar. Se puede comprobar en [la página de ESPHome](https://esphome.io/index.html) Los míos son de AliExpress.
También se puede encontrar en amazon: [https://amzn.to/3jewNUA](https://amzn.to/3jewNUA)

## Tiras de led de 1M direccionables
En mi caso, estoy usando [estas tiras de led](https://es.aliexpress.com/item/1005004014556428.html). Lo importante es que sean tiras de leds WS2812B. Estas utilizan el tipo de direccionamiento compatible con el software que instalaremos en el ESP32

# Configuración del ESP32
Vamos a instalar un WLED. Un firmware especial para controlar luces de este tipo y que tiene compatibilidad con home assistant integrada.
Para esto basta con 
  - Tener bien instalado el driver CH340[el driver CH340](https://sparks.gogo.co.nz/ch340.html) para que el USB de ESP32 sea reconocido como un puerto serie mas.
  - usar Chrome
  - seguir los pasos en https://install.wled.me/ (seleccionar el COM del ESP y dependiendo de la placa ESP tener presionado el botón de BOOT mientras se hace click en instalar)

Para conectar la placa a la tira de led necesitaremos 3 puntos. Un GPIO para controlar los leds, un pin de 5V y un GND. Por defecto el firmware admite conectar las luces en el puerto GPIO1 sin ninguna otra configuración.

![myroom]({{ site.baseurl }}/assets/images/bed-lights/circuito.jpg "my room")


<div style="width:100%;height:480px;background-color:black;text-align:center;">
  <video style="height:50%;" controls>
    <source src="https://lh3.googleusercontent.com/uVnrVMIjCbXwZ93-Ak9pHeZQY5ibRPNfmnoLjp3xtY2Uu0AaVTFw3K05LMRB51bG3MkEKgkzmsZXGsU2mmUEG6UHjzIi5kNcNOkjdToaG36utpq4PKdOl9T7VTrpGi22ZiNv15TONkg=m18" type="video/mp4">
  </video>
</div>


De este paso hay muchos videos, por ejemplo: https://www.youtube.com/watch?v=TOEnFKLm9Sw

# Configuración en Home Assistant
Una vez instalado el firmware y el ESP conectado, Home Assistant debería reconocer un dispositivo nuevo WLED que se puede configurar sin problemas y que dejara accesibles todas las entidades de la luces.
Las que vamos a utilizar aquí, serán color y intensidad.

![controls]({{ site.baseurl }}/assets/images/bed-lights/light_controls.PNG "controls")

# Crear botón de acceso rápido para encendido en iOS
La primera automatización que se necesita es la de apagado y encendido. Me decanté por hacerlo con el móvil, asi no tenía que diseñar y implementar el botón físico.
La idea es la siguiente: La aplicación para Home Assistant permite implementar acciones en el teléfono que se traducen en EVENTOS que se pueden leer desde home assistant. Utilizaremos uno de estos eventos para indicar encendido/apagado. 
Luego, a traves del widget para iOS de HA se puede acceder el botón fácilmente en cualquier lugar.

Vamos paso a paso:
En la Aplicación vamos a Settings -> Companion App -> Actions -> Add
Aquí solo tenemos que poner un nombre y guardar para después el EXAMPLE TRIGGER al final

![action]({{ site.baseurl }}/assets/images/bed-lights/action.PNG "action")

Luego nos vamos a Home Assistant a Settings -> Automations & Scenes y creamos una nueva automatización. Dentro de la automatización hacemos click en los tres puntos de la derecha para abrir el editor YAML y usamos lo siguiente:

```yaml
alias: Bedside light 
description: Bedside light description
trigger:
  - platform: event
    event_type: ios.action_fired
    event_data:
      actionID: your_action_id
      actionName: Read Light
condition: []
action:
  - if:
      - condition: device
        type: is_on
        device_id: your_device_id
        entity_id: light.wled
        domain: light
    then:
      - type: turn_off
        device_id: your_device_id
        entity_id: light.wled
        domain: light
    else:
      - type: turn_on
        device_id: your_device_id
        entity_id: light.wled
        domain: light
        enabled: true
        brightness_pct: 20
mode: single
```

Remplazamos el actionID con el valor del trigger example en la action la companion app que vimos antes, y también el device y entity del dispositivo añadido en Home Assistant. 
Esta misma configuración se puede conseguir igualmente solo con las opciones de la UI.
Después solo queda añadir el widget de Home Assistant con las acciones y deberíamos tener algo como esto:

![widget]({{ site.baseurl }}/assets/images/bed-lights/widget.PNG "widget")

# Sincronizar encendido con la alarma de iOS
Para esto utilizaremos los Atajos de iOS, que nos permiten ejecutar una acción cuando se activa la alarma del móvil.
La acción será enviar un evento a Home Assistant. Y de la misma manera que antes la utilizaremos para encender la luz a brillo máximo.
El nombre de la acción no es relevante, mientras sea el mismo que leamos desde HA

![widget]({{ site.baseurl }}/assets/images/bed-lights/automatizacion.PNG "widget")

# Caja impresa en 3D
Para encapsular el circuito diseñé una caja simple con cierre por imanes.
![3DPrintedBox]({{ site.baseurl }}/assets/images/bed-lights/box.PNG "3D printed box")


# Siguiente pasos
Como mejora, me gustaría añadir un botón manual de encendido en la cama. De este modo es mas fácil encender y apagar durante la noche si ver el móvil.
EL firmware de WLED deja automáticamente el PIN GPIO0 como switch, si se conecta con tierra se apaga y enciende la luz, osea sería cuestión solo de diseñar el botón, sin requerir nada de esto software.

Otra mejora seria introducir un sensor de movimiento para en la noche al levantarse active dos o tres sensores de abajo para iluminar un poco el camino.