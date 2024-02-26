---
layout: post_with_comments
title:  "Migrating before Google Business websites shutdown"
description: Another service that goes down and takes your work with it
date:   2024-02-25 16:20:00 +0100
image: "https://lagoyluz.com/lh3.googleusercontent.com/p/AF1QipMUK2ahO4YOlEz0l_vQSrg-IZVRpxFyhGF2YEgl=w768-h768-n-o-v1"
---

<img class="d11Ssd" src="https://lagoyluz.com/lh3.googleusercontent.com/p/AF1QipMUK2ahO4YOlEz0l_vQSrg-IZVRpxFyhGF2YEgl=w768-h768-n-o-v1">

So, once again, Google is saying goodbye to another service. Websites attached to the businesses profile you can create from Google Maps map points

My parents have [a rental cabin by the lake](https://lagoyluz.com), and my mom took advantage of this easy service of google, where you can quickly create a website so it gets linked in the Google Maps and well, and it works as a standalone website for all purposes. These pasts few weeks I've been reading a lot about small/indie web and how companies do take most of oue information with them when they shut down. An specially sad website I came across it's the [google graveyard](https://killedbygoogle.com/) which lists all companies Google has killed.

They received a notice from google that they are shuting down these pages soon and gave no other option than just creating a new one form services like Wix. Which is not at all good nor an easy solution. <https://support.google.com/business/answer/14368911?hl=en>

![googles_options]({{ site.baseurl }}/assets/images/options.png "Wix, Shopify and others offered as alternative places to create a website")

Since my mom put effort on creating this website, I wanted to save it somehow, so I thought, since it's basically static website with some dynamic comments, to create a local copy with wget and host it on my VPS under a new domain name I bought for the occasion: <https://lagoyluz.com>

```bash
wget --adjust-extension -H -k -K -p https://lagoyluz.negocio.site 
```
I found these parameters to work best to download also some resources that were needed like CSS and Fonts, and some images. You can check the wget docs for the details on each parameter.

The cloned index.html itself was absolutely bloated with google's javascript tracking code and it was completely unfriendly. I made a first pass of removing some scripting I thought not necessary for my clone, and manage to cut it down to a still not very nice 3500 lines of a lot of garbage. Anyway, everything was displayed and working except for the navigation menu and its links, which I did not investigate why in the cloned version did not worked so I just added my own Javascript to cover links and anchor functionality

I copied to a static folder behind my nginx reverse proxy and created a certificate for it with Let's encrypt (which gave me some trouble cause I forgot again how the acme challenges worked). And after a few hours I had my own version of the website working under a new domain with SSL

<https://lagoyluz.com>

Regarding the dynamic content, I will have to think in some backend or static build for my parent to be able to push new reviews they want to share and change the photographs

##  notes

Now more than ever, I am feeling strong about walled gardens and the impact of trusting them with your information, work and time, just to be let down once they decide to shutdown x or y service for no reason other than not making billions a year.
It is true, though, that an effort is required to make this process of self hosting or domain name driven identity more technically affordable. 
We need more services like the ones being developed by the Small Tech Foundation <https://small-tech.org/>, and go even further with high level services where spawning you digital identity with your own domain at home is a matter of a few clicks, just as easy as getting a new phone number.

## related links
<https://boffosocko.com/2017/07/28/an-introduction-to-the-indieweb/>

<https://small-tech.org/>

<https://blog.morettigiuseppe.com/articles.html> From when I helped to render the cabin's blueprints