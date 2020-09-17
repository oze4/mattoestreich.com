---
layout: article
title:  "Post Testing Please Ignore"
date:   1920-01-01
description: "testing something you can ignore this"
headerImage: https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/kubernetes.png
headerImageHeight: 80%
headerImageWidth: auto
categories: kubernetes microservice golang
---

Solution commonly found online only fixes half of the issue.

<img class="modal-image" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/ns-stuck-term.png" alt="namespace stuck in terminating state">

If you have ever had a namespace stuck in terminating state, you have most likely Googled this issue...and I bet you found the same article I did. Which, as it turns out, is **wrong**!

[This](https://medium.com/@craignewtondev/how-to-fix-kubernetes-namespace-deleting-stuck-in-terminating-state-5ed75792647e) is the article you most likely stumbled upon, which provides an incorrect solution. The solution they provide will leave remnants of those "fixed" namespaces around - just overall not very clean.

Thankfully, I read through the comments and [found the correct solution](https://medium.com/@cristi.posoiu/this-is-not-the-right-way-especially-in-a-production-environment-190ff670bc62).

## Jumping the gun, a microservice

Unfortunately, before reading through the comments, I wrote a microservice to automate the steps outlined in the incorrect solution. 

### [Feel free to check out the GitHub repo here](https://github.com/oze4/service.remove-terminating-namespaces).

Overall, I'm glad I did this as it helped me begin to see what is "under the hood" in Kubernetes, and maybe others can learn from this. Whether learning how to use an in-cluster-client-config with custom service accounts, or learning how to use an out-of-cluster-client-config to write applications that interact with Kubernetes, there is still value in this microservice that half solves your issue. ;)

I wrote this microservice using [the official Kubernetes Golang SDK, `client-go`](https://github.com/kubernetes/client-go) and it is designed to run as a cron job once per hour (which you can modify). 

<div style="text-align:center">
<img class="modal-image" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/ns-stuck-term-cronjob.png" alt="microservice in action" style="max-height:60rem;">
</div>

## The Correct Solution

[Taken from here](https://medium.com/@cristi.posoiu/this-is-not-the-right-way-especially-in-a-production-environment-190ff670bc62)

>This is not the right way, especially in a production environment.
>
>Today I got into the same problem. By removing the finalizer you’ll end up with leftovers in various states. You should actually find what is keeping the deletion from complete.
>
>See https://github.com/kubernetes/kubernetes/issues/60807#issuecomment-524772920
>
>(also, unfortunately, ‘kubetctl get all’ does not report all things, you need to use similar commands like in the link)
>
>My case — deleting ‘cert-manager’ namespace. In the output of ‘kubectl get apiservice -o yaml’ I found APIService ‘v1beta1.admission.certmanager.k8s.io’ with status=False . This apiservice was part of cert-manager, which I just deleted. So, in 10 seconds after I ‘kubectl delete apiservice v1beta1.admission.certmanager.k8s.io’ , the namespace disappeared.
>
>Hope that helps.
