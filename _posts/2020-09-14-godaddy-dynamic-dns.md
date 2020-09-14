---
layout: post
title:  "Homegrown Dynamic DNS for GoDaddy"
date:   2020-09-14
categories: kubernetes godaddygo dynamic-dns dns godaddy
---

This container helps make you flexible.

<div style="text-align:center;">
<img title="kubernetes" style="max-width:25rem;" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/kubernetes.png" alt="kubernetes">
</div>

### [Check out the GitHub repo here](https://github.com/oze4/service.godaddy-dynamic-dns)

I recently stood up an at home (lab) Kubernetes cluster to help with studying for my CKA/CKAD as well as started to learn `Go`. While standing up some services, I realized I would need some sort of dynamic DNS client due to the fact I do not have a static public IP.

So, I figured I would marry the two and write a "microservice" to run as a CronJob that updates my public IP (hosted with GoDaddy) when it changes (you can also compile the `Go` code and run on demand if you would like).

In order to properly create this "microservice", or dynamic DNS client, I knew I would need an SDK for GoDaddy's API. I took this as an opportunity to write my own GoDaddy SDK in `Go`, called `godaddygo`, which you can [read more about here](https://mattoestreich.com/golang/sdk/godaddy/godaddygo/2020/09/11/godaddygo.html). Once I was through writing the bare minimum I needed (the SDK does not support ALL of the API's features), I wrote the actual microservice.

<div style="text-align:center;">
<img title="godaddy" style="max-width:20rem;" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/godaddy.jpeg" alt="godaddy">
</div>

We pull your current public IP from [https://icanhazip.com](https://icanhazip.com) and match that to a `BASELINE_RECORD` that you will need to create within your zone. We use the `BASELINE_RECORD` as a database of sorts, it is the record we use to compare with what `icanhazip.com` reports. This is how we know your public IP has changed.

```bash
# You **have** to use these environmental variables
# Whether from `dotenv` or `docker run -e <...>`

# Your GoDaddy API key
GODADDY_APIKEY=<your_api_key>

# Your GoDaddy API secret
GODADDY_APISECRET=<your_api_secret>

# The domain you would like to target
GODADDY_DOMAIN=<domainyouown.com>

# The A record within that domains zone to use as our
# baseline
# I generated a GUID and just used that as my baseline
# record
BASELINE_RECORD=<someArecord>
```
You can either build the container yourself or use the one [on my Docker Hub](https://hub.docker.com/repository/docker/oze4/godaddy-dynamic-dns).


<div style="text-align:center;">
<img style="max-height:20rem;" class="modal-image" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/godaddy-dynamic-dns-cronjob-running.png" alt="godaddy-dynamic-dns container running">
<p style="margin:0;">cronjob running</p>
</div>

Hopefully, this can be of use to someone!