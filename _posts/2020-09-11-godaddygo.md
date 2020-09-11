---
layout: post
title:  "My Latest Adventure, godaddygo"
date:   2020-09-11
description: "A Golang SDK for GoDaddy API"
categories: golang sdk godaddy godaddygo
---

I have been learning `Go` for a couple of weeks now, this is what I built while learning the language!

[godaddygo](https://github.com/oze4/godaddygo)

![texture theme preview](https://raw.githubusercontent.com/oze4/mattoestreich.com/master/golang.jpg)

After getting laid off I decided to spend some time learning new things, as to better myself and my career. About a year ago or so, I fiddled with Kubernetes on DigitalOcean - this was, of course, after failing to run it on some hardware I had laying around. The support for bare-metal load balancers wasn't like it is today (or I just wasn't as competent).

I have also been extremely interested in `Go`. So, I thought, "`Go` is used heavily in the Kubernetes ecosystem, I should give it another shot" (and possibly pick up a new cert along the way).

After setting up my home Kubernetes cluster and standing up services, some which required public access, I quickly realized I would need some sort of dynamic DNS. I do not have a static IP at home, so I figured I could resolve this issue myself. Just stand up a service that checks my public IP and if it is different that what was recorded "last run", update my public records via the GoDaddy API.

I attempted to write this service in `TypeScript` at first, but for some reason I had issues with the existing SDK. So, I figured it was a sign to give `Go` a shot...and here we are!
