---
layout: post
title:  "Homegrown Dynamic DNS for GoDaddy"
description: "this container helps make you flexible"
date:   2020-09-14
categories: kubernetes godaddygo dynamic-dns dns godaddy
---

This container helps make you flexible.

<div style="text-align:center;">
<img title="kubernetes" style="max-width:80%;" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/kubernetes.png" alt="kubernetes">
</div>

### [Check out the GitHub repo here](https://github.com/oze4/service.godaddy-dynamic-dns)

I recently stood up an at home (lab) Kubernetes cluster to help with studying for my CKA/CKAD as well as started to learn `Go`. While standing up some services, I realized I would need some sort of dynamic DNS client due to the fact I do not have a static public IP.

So, I figured I would marry the two and write a "microservice" to run as a CronJob that updates my public IP (hosted with GoDaddy) when it changes (you can also compile the `Go` code and run on demand if you would like).

In order to properly create this "microservice", or dynamic DNS client, I knew I would need an SDK for GoDaddy's API. I took this as an opportunity to write my own GoDaddy SDK in `Go`, called `godaddygo`, which you can [read more about here](https://mattoestreich.com/golang/sdk/godaddy/godaddygo/2020/09/11/godaddygo.html). Once I was through writing the bare minimum I needed (the SDK does not support ALL of the API's features), I wrote the actual microservice.

<div style="text-align:center;">
<img title="godaddy" style="max-width:5rem;" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/godaddy.jpeg" alt="godaddy">
<p><small>godaddy</small></p>
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
You can either build the container yourself or use the one [on my Docker Hub](https://hub.docker.com/repository/docker/oze4/godaddy-dynamic-dns). The code for the full "microservice" can be found below.


<div style="text-align:center;">
<img style="max-height:20rem;" class="modal-image" src="https://raw.githubusercontent.com/oze4/mattoestreich.com/master/assets/godaddy-dynamic-dns-cronjob-running.png" alt="godaddy-dynamic-dns container running">
<p style="margin:0;">cronjob running</p>
</div>

Hopefully, this can be of use to someone!

```golang
package main

import (
	"errors"
	"fmt"
	"io/ioutil"
	"log"
	"net/http"
	"os"
	"strings"

	"github.com/joho/godotenv"
	"github.com/oze4/godaddygo"
	"github.com/oze4/godaddygo/pkg/endpoints"
)

const icanhazipURL = "https://icanhazip.com"

func main() {
	godotenv.Load()

	api := newIcanhazip()

	apires, err := api.get()
	if err != nil {
		log.Fatalf("Error getting public IP from %s %s\n", icanhazipURL, err.Error())
		os.Exit(1)
	}

	k := os.Getenv("GODADDY_APIKEY")
	s := os.Getenv("GODADDY_APISECRET")
	d := os.Getenv("GODADDY_DOMAIN")

	gd := newGoDaddy(k, s, d)

	gdres, err := gd.get()
	if err != nil {
		explainStrExit("Error getting IP from GoDaddy: "+err.Error(), 1)
	}

	if apires == gdres {
		explainStrExit("Public IP has not changed: "+apires, 0)
	}

	explainStr("Public IP has changed, updating GoDaddy now. Old: " + gdres + " New: " + apires)

	if err := gd.update(apires); err != nil {
		explainStrExit("Error updating GoDaddy DNS: "+err.Error(), 1)
	}

	explainStrExit("\nDone\n", 0)
}

/**
 * helper functions
 */

func explainStr(message string) {
	fmt.Printf("%s\n", message)
}

func explainStrExit(message string, exitCode int) {
	fmt.Printf("%s\n", message)
	os.Exit(exitCode)
}

/**
 * icanhazip
 */

func newIcanhazip() icanhazip {
	return icanhazip{}
}

type icanhazip struct{}

func (i icanhazip) get() (string, error) {
	resp, err := http.Get(icanhazipURL)
	if err != nil {
		return "", err
	}
	defer resp.Body.Close()
	body, err := ioutil.ReadAll(resp.Body)
	if err != nil {
		return "", err
	}
	return strings.TrimSpace(string(body)), nil
}

/**
 * godaddy
 */

func newGoDaddy(k, s, d string) goDaddy {
	baseline := os.Getenv("BASELINE_RECORD")
	api := godaddygo.ConnectProduction(k, s).V1().Domain(d).Records()
	return goDaddy{baseline, api}
}

type goDaddy struct {
	baseline string // BASELINE_RECORD
	recs     endpoints.Records
}

// get returns the *value* of your `BASELINE_RECORD`
func (g *goDaddy) get() (string, error) {
	r, e := g.recs.GetByTypeName("A", g.baseline)
	if e != nil {
		return "", e
	}
	return strings.TrimSpace((*r)[0].Data), nil
}

// update sets all A records to your new public IP
func (g goDaddy) update(newIP string) error {
	r, e := g.recs.GetByType("A")
	if e != nil {
		return e
	}
	for _, d := range *r {
		if e := g.recs.SetValue(d.Type, d.Name, newIP); e != nil {
			return errors.New("Error setting record: " + d.Name)
		}
	}
	return nil
}
```

<div style="max-height:35rem;overflow:scroll;">

</div>
