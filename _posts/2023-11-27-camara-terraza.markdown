---
layout: post_with_comments
title:  "Módulo ESP32 con camara en Home Assistant"
description: Añadiendo una placa ESP32 con cámara y sensor de movimiento
date:   2023-11-27 23:07:00 +0200
---

![placa]({{ site.baseurl }}/assets/images/camara-terraza/placa.png "placa")

Vamos a integrar un módulo ESP32 con cámara, sensor de movimiento y pantalla en mi instalación de Home Assistant. La clave para este proyecto fue la implementación de ESPHome, una plataforma que simplifica la integración de dispositivos ESP32 en Home Assistant.

El modulo que tenía guardado hace un tiempo es el TTGO: LILYGO® TTGO T-Camera ESP32 WROVER cámara 0,96 OLED. Importante es que la versión de mi placa es 1.6.2. Eso sera relevante a la hora de identificar los diferentes pines de los periféricos.

## Problemas Encuentros y Soluciones

### 1. Problema con el Flasheo y Windows
Al principio, me encontré con un problema al intentar cargar el programa en el ESP32. Por algún motivo en MacOS era imposible hace la carga del programa en el ESP32. Quizás los drivers USB/Serial. La solución fue cambiar a Windows. Desde Windows pude cargar los binario desde ESPHome.

### 2. Cambios en los Pines de la Cámara
Experimenté dificultades con los pines de la cámara, ya que estaban ligeramente diferentes a los estándar. Afortunadamente, encontré ejemplos útiles que me ayudaron a solucionar este problema. El la sección de links "Configuración Funcional para TTGO ESP32 Camera Board" Uno de los comentarios propone una configuración de PINs que sirve para la placa 1.6.2

## Configuración de ESPHome
La configuración la hice toda a través de las herramientas de ESPHome. ESPHome es una plataforma de código abierto diseñada para simplificar la integración de dispositivos basados en ESP8266 y ESP32 con el ecosistema Home Assistant. Permite la configuración fácil y la programación de dispositivos IoT utilizando un formato YAML sencillo, lo que facilita la creación de firmware personalizado para una variedad de sensores, actuadores y otros componentes electrónicos.

Me base en la configuración de los links de recursos, pero finalmente adapté la configuración inicial de espHome con la de los posts de referencia. 
También añadí la configuración para poder enviar mensajes a la pantalla a través de una cola de MQTT. Esto último no tiene mucha utilidad pero era mi primera vez interactuando con MQTT.

```yaml
esphome:
  name: camera-1

esp32:
  board: esp32dev
  framework:
    type: arduino

# Example configuration entry
mqtt:
  broker: homeassistant
  username: mosquitto
  password: 432fr354g543r

# Enable logging
logger:

# Enable Home Assistant API
api:
  encryption:
    key: "fdsfdsfds+cKCuURB6GbFf4="

ota:
  password: "fdsfdsfdsfds"

wifi:
  ssid: !secret wifi_ssid
  password: !secret wifi_password

  # Enable fallback hotspot (captive portal) in case wifi connection fails
  ap:
    ssid: "Camera-1 Fallback Hotspot"
    password: "23243242d"

captive_portal:
  
# ttgo_camearv16 configuration
esp32_camera:
  external_clock:
    pin: GPIO4
    frequency: 20MHz
  i2c_pins:
    sda: GPIO18
    scl: GPIO23
  data_pins: [GPIO34, GPIO13, GPIO14, GPIO35, GPIO39, GPIO38, GPIO37, GPIO36]
  vsync_pin: GPIO5
  href_pin: GPIO27
  pixel_clock_pin: GPIO25
  resolution: 640x480
  jpeg_quality: 10

  # Image settings
  name: Camera 1

binary_sensor:
  - platform: gpio
    pin: 
      number: GPIO19
      mode: INPUT
      inverted: False
    name: Camera 1 Motion
    device_class: motion
    filters:
      - delayed_off: 1s
  - platform: gpio
    pin:
      number: GPIO15
      mode: INPUT_PULLUP
      inverted: True
    name: Camera 1 Button
    filters:
      - delayed_off: 50ms
  - platform: status
    name: Camera 1 Status

sensor:
  - platform: wifi_signal
    name: Camera 1 WiFi Signal
    update_interval: 10s
  - platform: uptime
    name: Camera 1 Uptime

i2c:
  sda: GPIO21
  scl: GPIO22
# This file has to exists when compiling
font:
  - file: "fonts/Roboto-Regular.ttf"
    id: tnr1
    size: 17

# Configuración para Mosquitto MQTT
text_sensor:
  - platform: mqtt_subscribe
    name: "Info for screen"
    id: infocamera
    topic: home/camera1_screen_text

display:
  - platform: ssd1306_i2c
    model: "SSD1306 128x64"
    address: 0x3C
    rotation: 180
    lambda: |-
      it.printf(0, 0, id(tnr1), "%s", id(infocamera).state.c_str());
```

Antes de cargar espHome emite una key que habrá que introducir después para integrar el dispositivo, se tendrá que guardar hasta entonces.

## Integrando MQTT para Pantalla

Para aprovechar al máximo la pantalla, decidí utilizar MQTT para mostrar información relevante. Configuré manualmente la conexión con el servidor MQTT y agregué el componente necesario para la pantalla. MQTT tiene que estar instalado en Home Assistant. Se puede hacer a través del addon de Mosquitto. Se instala muy fácil. Luego solo queda definir un TOPIC donde se escriben y leen los mensajes de la cola. En mi caso home/camera1_screen_text. No es necesario que tenga esta estructura. Peru la estructura con / ayuda a organizar diferentes estancias.

Utilicé este pequeño script para probar mi configuración de Mosquitto que habia instalado en Home Assistant. Este script lo ejecuté desde otro ordenador conectado a la red.

```python
import paho.mqtt.client as mqtt

def on_connect(client, userdata, flags, rc):
    print("Connected with result code "+str(rc))

    # Subscribing in on_connect() means that if we lose the connection and
    # reconnect then subscriptions will be renewed.
    client.subscribe("home/camera1_screen_text")

# The callback for when a PUBLISH message is received from the server.
def on_message(client, userdata, msg):
    print(msg.topic+" "+str(msg.payload))

client = mqtt.Client()
client.on_connect = on_connect
client.on_message = on_message
client.username_pw_set("mosquitto", "6Y2TRQuNtjPHqEvC")

client.connect("homeassistant", 1883, 60)

# Blocking call that processes network traffic, dispatches callbacks and
# handles reconnecting.
# Other loop*() functions are available that give a threaded interface and a
# manual interface.
client.loop_forever()

```
## Carcasa impresa en 3D

He realizado una modificación en la carcasa para mi proyecto del termostato, inspirado en el artículo [Control de Calefacción con Home Assistant](https://blog.morettigiuseppe.com/2023/01/27/control-calefaccion-homeassistant.html). La nueva tapa tiene los agujeros para la cámara, pantalla, botones y sensor de movimiento. 

A continuación, puedes ver la foto de la nueva tapa:

![tapa]({{ site.baseurl }}/assets/images/camara-terraza/tapa.png "tapa")


El nuevo modelo no tenía las medidas adecuadas por tanto la placa no se cabia en el marco, pero como referencia, aquí están los enlaces a los archivos STL:

- [Archivo]({{ site.baseurl }}/assets/images/camara-terraza/tapa_boiler_with_camera.stl)

![final]({{ site.baseurl }}/assets/images/camara-terraza/final.png "final")
Asi se vé ya instalada

## Configuración en Home Assistant
Aquí está la imagen de la configuración en HA, utilice una de las tarjetas predefinidas que pueden mostrar el video en la pantalla principal y ademas los botones para el sensor de movimiento y otras cosas que puedan haber en esa estancia (luces por ejemplo)

![config]({{ site.baseurl }}/assets/images/camara-terraza/config.png "config")

## BONUS: Cámara autónoma
Es posible instalar un firmware que conecté la cámara a la red y asi poder acceder directamente a las imágenes. Se puede descargar desde "Documentación Original (YAML) - Importante para el Streaming" en los recursos de referencia.

## Recursos Útiles

- [Configuración Funcional para TTGO ESP32 Camera Board](https://community.home-assistant.io/t/working-esphome-config-for-ttgo-esp32-camera-board-with-microphone/126231/9)
- [Documentación Original (YAML) - Importante para el Streaming](https://github.com/lewisxhe/esp32-camera-series/blob/master/docs/yaml.md)
- [Configuración de MQTT en Home Assistant](https://github.com/home-assistant/addons/blob/master/mosquitto/DOCS.md)
- [Componente Text Sensor MQTT Subscribe](https://esphome.io/components/text_sensor/mqtt_subscribe)