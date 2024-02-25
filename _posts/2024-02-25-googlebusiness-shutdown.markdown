---
layout: post_with_comments
title:  "Migrating before Google Business websites shutdown"
description: Another service that goes down and takes you work with it
date:   2024-02-25 16:20:00 +0100
---

<img class="d11Ssd" src="https://lagoyluz.com/lh3.googleusercontent.com/p/AF1QipMUK2ahO4YOlEz0l_vQSrg-IZVRpxFyhGF2YEgl=w768-h768-n-o-v1">

So, once again, Google is saying goodbye to another service. Websites attached to the businesses profile you can create from Google Maps map points

My parents have [a rental cabin by the lake](https://lagoyluz.com), and my mom took advantage of this easy service of google, where you can quickly create a website so it gets linked in the Google Maps and well, it works as a standalone for all purposes. These pasts weeks I've been reading a lot about small/indie web and how they do take everything with them when they shut down, and I came across the google graveyard site https://killedbygoogle.com/ 
My parents receive the notice from google that they are shuting down this pages and gave no other option than just creating a new one form services like Wix. Which is not at all good nor an easy solution. https://support.google.com/business/answer/14368911?hl=en

![googles_options]({{ site.baseurl }}/assets/images/jellyfin-esp-power-up/Captura.PNG "Wix, Shopify and others offered as alternative places to create a website")

Since my mom put effort on creating this website, I want it to save it somehow, so I taught, since it's basically static website with some dynamic comments, to create a local copy with wget and host it on my VPS under a new domain name I bought for the occasion: lagoyluz.com

```bash
wget --adjust-extension -H -k -K -p https://lagoyluz.negocio.site 
```
I found these parameters to work best on downloading also some resources that were used like CSS and Fonts, and some images. You can explored in the wget docs
https://blog.morettigiuseppe.com/articles.html
The index itself was absolutely bloated with JS tracking code and completely unfriendly code. I made a first pass of removing some scripting I thought not necessary for my clone, and manage to cut it down to a still not very nice 3500 lines of a lot of garbage. Anyway, everything was displayed and working except for the navigation menu and its links, That I did not investigate why in the cloned version did not worked so I just added my own Javascript to cover links and anchors

I copied to a static folder behind my nginx reverse proxy and created a certificate for it with Let's encrypt (which gave me some trouble cause I forgot again how the challenges worked). And after a few hours I had my own version of the website working under a new domain with SSL

https://lagoyluz.com

Regarding the dynamic content, I will have to think in some backend or static build for my parent to be able to push new reviews they want to share and change the photographs

##  notes

Now more than ever, I am feeling strong about walled gardens and the impact of trusting them with your information, work and time, just to be let down once they decide to shutdown x or y service for no reason other than not making billions a year.
It is true though that an effort is required to make this process of self hosting or domain name driven identity more technically affordable, or at least portable. Like the technologies like Small Tech Foundation https://small-tech.org/ is developing, and even more as more high level services where spawning you digital identity with your own domain at home is a matter of a few clicks, just as easy as getting a new phone number.

## related links
https://boffosocko.com/2017/07/28/an-introduction-to-the-indieweb/

https://small-tech.org/

https://blog.morettigiuseppe.com/articles.html From when I helped to render the cabin's blueprints