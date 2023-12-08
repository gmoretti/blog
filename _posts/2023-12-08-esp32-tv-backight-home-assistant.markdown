---
layout: post_with_comments
title:  "TV backlight with LED strip and ESPHOME"
description: Home Assistant integrated TV backlight with ESP32 and WLED
date:   2023-12-03 15:24:00 +0200
---

<video width="315" height="516" controls>
    <source src="https://morettigiuseppe-blog-files.s3.eu-west-3.amazonaws.com/tv-backlight-reel-vertical.mov" type="video/mp4">
</video>

This project essentially follows the same approach as my previous Bed Lights project, utilizing the ESP32 and ESPHome, along with the same LED strip.

However, in this iteration, I introduced some automations. Apart from controlling the light in Home Assistant, I also integrated Chromecast with the TV. This integration allows me to determine whether the TV is on and which application is currently in use.

With this information, I implemented a couple of automations to adjust the light when the TV is in playback mode. Specifically, I dim the light when the TV is in playback, except for Spotify where the light effects differ.

Here are the details of the automations in YAML format:

```yaml

alias: TV Light reacts to playback in TV
description: ""
trigger:
  - platform: device
    device_id: 43254314315431
    domain: media_player
    entity_id: 543265435432
    type: playing
    enabled: false
  - platform: state
    entity_id:
      - media_player.livingroom_tv
    attribute: app_name
    from: com.google.android.apps.tv.launcherx
condition: []
action:
  - if:
      - condition: state
        entity_id: media_player.livingroom_tv
        attribute: app_name
        state: com.spotify.tv.android
    then:
      - service: scene.turn_on
        target:
          entity_id: scene.chill
        metadata: {}
    else:
      - service: scene.turn_on
        target:
          entity_id: scene.watching_tv
        metadata: {}
mode: single

```

This YAML configuration was generated using the GUI to set up the automation. Two scenes with predefined light configurations were created, specifying color and brightness options. This allows for easy modification of the scenes without affecting the automation.

That's all for today!