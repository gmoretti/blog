---
layout: post
title:  "Video Monitor on the corner of my TV"
description: "Frigate, Home Assistant and WebRTC magic"
date:   2024-12-04 14:00:00 +0200
---

![Notification on TV]({{ site.baseurl }}/assets/images/tv.JPG) 

After installing Frigate and connecting it with Home Assistant, the video feeds are available to do whatever I you want. Meaning, it's time to try to open the baby monitor everywhere I can. This includes the TV. 

The best article I came across to appreach this is this one. https://seanblanchfield.com/2022/03/realtime-pip-cameras-on-tv-with-home-assistant
Thanks to Sean!
And then this HA thread: https://community.home-assistant.io/t/a-short-guide-for-setting-up-tv-pip-notifications-with-pipup/537084/41

The idea is the following. Pipup App allows to send Notification to a corner of the TV with REST requests. This request after a few patches from forks of Pipup allow them to be with JavaScript and from a non secured HTTP source. (This is explained in the HA thread above.)
We configure Home Assistant to send such REST request, so that we have a service in HA we can use on demand.

The main problem I faced was that when I played the stream on top of Netflix, the sound gets muted and there's no way to put it back unless you restart the App. This is too inconvenient for an alarm notification type of scenario. But starting the notification before opening the Netflix stream works fine. So it would work for having a fixed monitor of the feed in the corner.

The page which contains the Stream I took it from go2rtc installation, I had to open the port on my docker config 1984, to be able to access from outside. But there's plenty of features in this control panel. Lots of different streams to use nad configurable pages.

To do the automations in Home Assistant, I created two Scripts in HA, one for sending a notification for, say 3h. (to keep it open), and another empty notification of 1s. This last one replaces the former and helps closing the window. The scripts use the Rest Services configured from the tutorial mentioned above.

### Turn ON Script

```yaml
alias: Baby Monitor on TV
sequence:
  - action: rest_command.pipup_url_on_tv
    data:
      duration: 10800
      url: http://yourFrigateIntallationIP:1984/stream.html?src=bebe&width=150px
description: turn the notification for 3h
icon: mdi:baby
```

The width controls the size in the screen in the notification, I found this work to have a tiny monitor in the corner, provided you configure the size of the notification to 320px, when following the tutorials mention above.

### Turn OFF Script

```yaml
alias: "CLEAR Baby Monitor on TV "
sequence:
  - action: rest_command.pipup_url_on_tv
    data:
      duration: 1
description: ""
icon: mdi:baby
```

Then I created a Toggle template in Devices->Helpers that uses the two scripts as ON OFF so they can be easily triggered from dashboard.

I would have liked Netflix to remain with sound to have the alerts. But this solution works for a general use case.

If I had it working as alerts I would also liked to implement the audio detection for criying in Frigate and have the alert jumping when that happened.

**Leave any comments in this Fediverse thread! [este thread del fediverso]()**