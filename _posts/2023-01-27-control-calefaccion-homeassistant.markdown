---
layout: post_with_comments
title:  "Control de caldera con Home Assistant y ESP32"
description: Control de la calefacción utilizando un módulo relé, ESP32 y Home Assistant
date:   2023-01-27 21:00:00 +0200
---

La solución más fácil hubiera sido un Nest, pero bueno, que hay de divertido en eso.

<strong> Advertencia: Algunas conexiones trabajan a 220V, si no estás seguro de lo que haces mejor consultar un profesional. </strong> 

# Instalación original
La instalación de mi caldera actual tenía un termostato honeywell inalámbrico, que después de inspeccionarlo un poco
solo tenía un par de relés. Parecía que accionaba la caldera con solo un interruptor.

Es un módulo termostato que se posiciona dentro de casa y manda la señal RF para activar el relé.
![Termostato original]({{ site.baseurl }}/assets/images/boiler/termo.jpg "Termostato original")

## Identificando el cableado
De la caldera bajaban dos pares de cables, uno de alimentación de 220v para alimentar el módulo inalámbrico y otro a modo de interruptor para activar y desactivar la caldera.
[foto cableado y caldera]

A partir de videos y información de tipos de cableado y caldera se puede identificar que el A B es el cableado para poner en marcha la caldera. O sea, los cables A B los tenemos que poner en nuestro propio relé.
Los cables NL son 220V de línea que alimentan el módulo inalámbrico, que en mi caso no he usado porque la ESP32 se alimenta por USB a un enchufe. Se podría haber puesto una placa transformadora, pero la verdad prefería no meterme a transformar esos voltajes.

![ComponenteTermostato]({{ site.baseurl }}/assets/images/boiler/oldrfmodule.png "Componente Termostato")

# Instalación nueva
## Descripción
La idea es utilizar una placa ESP32 con ESPHome conectado a Home Assistant, la placa acciona un módulo relé que activa y desactiva la caldera desde la interfaz de termostato de Home Assistant.

## Componentes
Vamos primero a listar todos los componentes necesarios.

### Raspberry Pi 4 
Mi instalación de Home Assistant es en una raspberry, pero cualquier instalación serviría, siempre y cuando tenga Bluetooth. Los termómetros de Xiaomi que utilizo necesitan bluetooth.
Se pueden encontrar instrucciones en la web de Home Assistant para la instalación. (Esto probablemente sería lo más complicado de conseguir, hay falta de stock, aunque se supone debería volver en 2023)

### ESP32
Cualquier ESP32 debería funcionar. Se puede comprobar en [la página de ESPHome](https://esphome.io/index.html) Los míos son de AliExpress.
También se puede encontrar en amazon: https://amzn.to/3jewNUA

### Módulo Relé 
La mayoría funciona igual, los míos son de Amazon https://amzn.to/3HEgEBy

### Termómetro 
Este componente es el que usaremos para basar el control de temperatura. Tenía unos de Xiaomi que se pueden conectar a Home Assistant [con este tutorial](https://www.youtube.com/watch?v=5BEhAQwM0A0). Concretamente, el modelo es: https://es.aliexpress.com/item/1005005054100936.html



## Circuito ESP32 y Relé
El circuito de control es muy sencillo, necesitaremos soldar 3 pines de la placa ESP32 al módulo Relé, uno de GPIO, cualquiera sirve, yo usé TODO. Luego necesitaremos un GND y un VCC. Suele haber un par de cada en las placas. Los de GND y VCC van al GND y VCC del módulo relé, y el GPIO al Signal line (en el caso del módulo del link mencionado anteriormente).

![Circuito]({{ site.baseurl }}/assets/images/boiler/circuit.png "Circuito")

Cualquier tutorial de "controlar un rele con ESP32" mostrará una configuración similar, simplemente cambiarán que pines se usa.

### Configuración del ESP32
Necesitaremos subir la configuración al módulo ESP32 para esto necesitaremos tener ESPHome en Home Assistant. Esto hay otras personas que lo explican mucho mejor que yo, por ejemplo [aquí](https://www.youtube.com/watch?v=KmTdGaOQzyk).
El código que vamos a subir a la placa es el siguiente:

```yaml
esphome:
  name: boiler-control

esp32:
  board: lolin32
  framework:
    type: arduino

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "ZQZysfdsfdsfsvc2+3rEbHA84g7uj9g="

ota:
  password: "93eb9a3fdsfdsfds684d23fadee4a"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Boiler-Control Fallback Hotspot"
    password: "dHFVhxEDkT0I"

captive_portal:

switch:
  - platform: gpio
    pin: 1
    name: smartthermostatrelay 
```

Casi todo será autogenerado durante la instalación del ESPHome en la placa, por lo que no tenemos que cambiar nada de eso, solo tendremos que añadir la parte que configura nuestro interruptor:

```yaml
switch:
  - platform: gpio
    pin: 1
    name: smartthermostatrelay 
```

Esto hará que podamos referirnos a nuestra caldera como *smartthermostatrelay* dentro de Home Assistant

## Configuracion en Home Assistant
Una vez tenemos el termómetro configurado y nuestro interruptor con ESP32 también listo, solo necesitamos crear el componente de Termostato con el que podremos establecer la temperatura que queremos. Home Assistant hará el resto.

Esta sección se debe añadir al fichero */config/configuration.yaml* que se puede acceder de muchas maneras, yo lo modifiqué con el ADDON *File Editor*

```yaml
climate:
  - platform: generic_thermostat
    name: Termostato
    heater: switch.smartthermostatrelay
    target_sensor: sensor.termostato_temperature
```
Las dos variables importantes: heater es el relé que hemos instalado, debe coincidir con el nombre que le hayamos dado. Y luego, target_sensor, que sera la temperatura de referencia, este sensor saldrá del termómetro que se haya configurado.

Despues de un reinicio de Home Assistant el termostato debería ya ser visible y poder ajustarse a la temperatura deseada.

![ComponenteTermostato]({{ site.baseurl }}/assets/images/boiler/interface.png "Componente Termostato")

Para poder acceder desde fuera de casa y poder encender y apagar la calefacción antes de llegar a casa, yo utilizo el [servicio de cloud del mismo home assistant](https://www.nabucasa.com/config/), pero también es posible configurarlo manualmente, según algunos tutoriales.

## Caja impresa en 3D
Para proteger los componentes y poner todo debajo de la caldera, diseñé una caja, con los agujeros para atornillarla que coincidían con los que ya habían en la pared con el otro termostato, y para el cierre dejé unos pequeños huecos para pegar unos imanes y hacer un cierre magnético. Intente poner unas plataformas donde poner unos tornillos, pero no me salieron con las medidas correctas y simplemente pegue los componentes. No es ideal, pero funciona.

![3DPrintedBox]({{ site.baseurl }}/assets/images/boiler/box.png "3D printed box")

Todo instalado debajo de la caldera quedó así:
![Resultado]({{ site.baseurl }}/assets/images/boiler/fihnal.png "Resultado")

# Siguiente pasos
Lo primero que me gustaría mejorar es las temperaturas en las habitaciones, tengo varios termómetros más instalados en diferentes habitaciones, pero de momento solo tengo uno como referencia. Una mejora sería utilizar los sensores template de HA y dependiendo del momento del día, o una escena concreta, cambiar el termómetro de referencia. También podría ser basado en la media de todos los termómetros.

Ahora, el siguiente paso, sería integrar también las maquinas de aire acondicionado al sistema. HA con ESPHome también pueden controlar dispositivos con infrarrojos, como los mandos de los aires. Integrando los aires se podrían utilizar en modo caliente cuando la temperatura baje mucho y la caldera no sea suficiente. Pero también  estará listo para controlar los aires en el verano con el mismo termostato.