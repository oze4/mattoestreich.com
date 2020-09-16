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

<div style="max-height:35rem;overflow:scroll;">
<div class="language-golang highlighter-rouge"><div class="highlight"><pre class="highlight"><code><span class="k">package</span> <span class="n">main</span>

<span class="k">import</span> <span class="p">(</span>
	<span class="s">"errors"</span>
	<span class="s">"fmt"</span>
	<span class="s">"io/ioutil"</span>
	<span class="s">"log"</span>
	<span class="s">"net/http"</span>
	<span class="s">"os"</span>
	<span class="s">"strings"</span>

	<span class="s">"github.com/joho/godotenv"</span>
	<span class="s">"github.com/oze4/godaddygo"</span>
	<span class="s">"github.com/oze4/godaddygo/pkg/endpoints"</span>
<span class="p">)</span>

<span class="k">func</span> <span class="n">main</span><span class="p">()</span> <span class="p">{</span>
	<span class="n">godotenv</span><span class="o">.</span><span class="n">Load</span><span class="p">()</span>

	<span class="n">api</span> <span class="o">:=</span> <span class="n">newIcanhazip</span><span class="p">()</span>

	<span class="n">apires</span><span class="p">,</span> <span class="n">err</span> <span class="o">:=</span> <span class="n">api</span><span class="o">.</span><span class="n">get</span><span class="p">()</span>
	<span class="k">if</span> <span class="n">err</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="n">log</span><span class="o">.</span><span class="n">Fatalf</span><span class="p">(</span><span class="s">"Error getting public IP from %s %s</span><span class="se">\n</span><span class="s">"</span><span class="p">,</span> <span class="s">"https://icanhazip.com"</span><span class="p">,</span> <span class="n">err</span><span class="o">.</span><span class="n">Error</span><span class="p">())</span>
		<span class="n">os</span><span class="o">.</span><span class="n">Exit</span><span class="p">(</span><span class="m">1</span><span class="p">)</span>
	<span class="p">}</span>

	<span class="n">k</span> <span class="o">:=</span> <span class="n">os</span><span class="o">.</span><span class="n">Getenv</span><span class="p">(</span><span class="s">"GODADDY_APIKEY"</span><span class="p">)</span>
	<span class="n">s</span> <span class="o">:=</span> <span class="n">os</span><span class="o">.</span><span class="n">Getenv</span><span class="p">(</span><span class="s">"GODADDY_APISECRET"</span><span class="p">)</span>
	<span class="n">d</span> <span class="o">:=</span> <span class="n">os</span><span class="o">.</span><span class="n">Getenv</span><span class="p">(</span><span class="s">"GODADDY_DOMAIN"</span><span class="p">)</span>

	<span class="n">gd</span> <span class="o">:=</span> <span class="n">newGoDaddy</span><span class="p">(</span><span class="n">k</span><span class="p">,</span> <span class="n">s</span><span class="p">,</span> <span class="n">d</span><span class="p">)</span>

	<span class="n">gdres</span><span class="p">,</span> <span class="n">err</span> <span class="o">:=</span> <span class="n">gd</span><span class="o">.</span><span class="n">get</span><span class="p">()</span>
	<span class="k">if</span> <span class="n">err</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="n">explainStrExit</span><span class="p">(</span><span class="s">"Error getting IP from GoDaddy: "</span><span class="o">+</span><span class="n">err</span><span class="o">.</span><span class="n">Error</span><span class="p">(),</span> <span class="m">1</span><span class="p">)</span>
	<span class="p">}</span>

	<span class="k">if</span> <span class="n">apires</span> <span class="o">==</span> <span class="n">gdres</span> <span class="p">{</span>
		<span class="n">explainStrExit</span><span class="p">(</span><span class="s">"Public IP has not changed: "</span><span class="o">+</span><span class="n">apires</span><span class="p">,</span> <span class="m">0</span><span class="p">)</span>
	<span class="p">}</span>

	<span class="n">explainStr</span><span class="p">(</span><span class="s">"Public IP has changed, updating GoDaddy now. Old: "</span> <span class="o">+</span> <span class="n">gdres</span> <span class="o">+</span> <span class="s">" New: "</span> <span class="o">+</span> <span class="n">apires</span><span class="p">)</span>

	<span class="k">if</span> <span class="n">err</span> <span class="o">:=</span> <span class="n">gd</span><span class="o">.</span><span class="n">update</span><span class="p">(</span><span class="n">apires</span><span class="p">);</span> <span class="n">err</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="n">explainStrExit</span><span class="p">(</span><span class="s">"Error updating GoDaddy DNS: "</span><span class="o">+</span><span class="n">err</span><span class="o">.</span><span class="n">Error</span><span class="p">(),</span> <span class="m">1</span><span class="p">)</span>
	<span class="p">}</span>

	<span class="n">explainStrExit</span><span class="p">(</span><span class="s">"</span><span class="se">\n</span><span class="s">Done</span><span class="se">\n</span><span class="s">"</span><span class="p">,</span> <span class="m">0</span><span class="p">)</span>
<span class="p">}</span>

<span class="c">/**
 * helper functions
 */</span>

<span class="k">func</span> <span class="n">explainStr</span><span class="p">(</span><span class="n">message</span> <span class="kt">string</span><span class="p">)</span> <span class="p">{</span>
	<span class="n">fmt</span><span class="o">.</span><span class="n">Printf</span><span class="p">(</span><span class="s">"%s</span><span class="se">\n</span><span class="s">"</span><span class="p">,</span> <span class="n">message</span><span class="p">)</span>
<span class="p">}</span>

<span class="k">func</span> <span class="n">explainStrExit</span><span class="p">(</span><span class="n">message</span> <span class="kt">string</span><span class="p">,</span> <span class="n">exitCode</span> <span class="kt">int</span><span class="p">)</span> <span class="p">{</span>
	<span class="n">fmt</span><span class="o">.</span><span class="n">Printf</span><span class="p">(</span><span class="s">"%s</span><span class="se">\n</span><span class="s">"</span><span class="p">,</span> <span class="n">message</span><span class="p">)</span>
	<span class="n">os</span><span class="o">.</span><span class="n">Exit</span><span class="p">(</span><span class="n">exitCode</span><span class="p">)</span>
<span class="p">}</span>

<span class="c">/**
 * icanhazip
 */</span>

<span class="k">func</span> <span class="n">newIcanhazip</span><span class="p">()</span> <span class="n">icanhazip</span> <span class="p">{</span>
	<span class="k">return</span> <span class="n">icanhazip</span><span class="p">{}</span>
<span class="p">}</span>

<span class="k">type</span> <span class="n">icanhazip</span> <span class="k">struct</span><span class="p">{}</span>

<span class="k">func</span> <span class="p">(</span><span class="n">i</span> <span class="n">icanhazip</span><span class="p">)</span> <span class="n">get</span><span class="p">()</span> <span class="p">(</span><span class="kt">string</span><span class="p">,</span> <span class="kt">error</span><span class="p">)</span> <span class="p">{</span>
	<span class="n">resp</span><span class="p">,</span> <span class="n">err</span> <span class="o">:=</span> <span class="n">http</span><span class="o">.</span><span class="n">Get</span><span class="p">(</span><span class="s">"https://icanhazip.com"</span><span class="p">)</span>
	<span class="k">if</span> <span class="n">err</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="k">return</span> <span class="s">""</span><span class="p">,</span> <span class="n">err</span>
	<span class="p">}</span>
	<span class="k">defer</span> <span class="n">resp</span><span class="o">.</span><span class="n">Body</span><span class="o">.</span><span class="n">Close</span><span class="p">()</span>
	<span class="n">body</span><span class="p">,</span> <span class="n">err</span> <span class="o">:=</span> <span class="n">ioutil</span><span class="o">.</span><span class="n">ReadAll</span><span class="p">(</span><span class="n">resp</span><span class="o">.</span><span class="n">Body</span><span class="p">)</span>
	<span class="k">if</span> <span class="n">err</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="k">return</span> <span class="s">""</span><span class="p">,</span> <span class="n">err</span>
	<span class="p">}</span>
	<span class="k">return</span> <span class="n">strings</span><span class="o">.</span><span class="n">TrimSpace</span><span class="p">(</span><span class="kt">string</span><span class="p">(</span><span class="n">body</span><span class="p">)),</span> <span class="no">nil</span>
<span class="p">}</span>

<span class="c">/**
 * godaddy
 */</span>

<span class="k">func</span> <span class="n">newGoDaddy</span><span class="p">(</span><span class="n">k</span><span class="p">,</span> <span class="n">s</span><span class="p">,</span> <span class="n">d</span> <span class="kt">string</span><span class="p">)</span> <span class="n">goDaddy</span> <span class="p">{</span>
	<span class="n">baseline</span> <span class="o">:=</span> <span class="n">os</span><span class="o">.</span><span class="n">Getenv</span><span class="p">(</span><span class="s">"BASELINE_RECORD"</span><span class="p">)</span>
	<span class="n">api</span> <span class="o">:=</span> <span class="n">godaddygo</span><span class="o">.</span><span class="n">ConnectProduction</span><span class="p">(</span><span class="n">k</span><span class="p">,</span> <span class="n">s</span><span class="p">)</span><span class="o">.</span><span class="n">V1</span><span class="p">()</span><span class="o">.</span><span class="n">Domain</span><span class="p">(</span><span class="n">d</span><span class="p">)</span><span class="o">.</span><span class="n">Records</span><span class="p">()</span>
	<span class="k">return</span> <span class="n">goDaddy</span><span class="p">{</span><span class="n">baseline</span><span class="p">,</span> <span class="n">api</span><span class="p">}</span>
<span class="p">}</span>

<span class="k">type</span> <span class="n">goDaddy</span> <span class="k">struct</span> <span class="p">{</span>
	<span class="n">baseline</span> <span class="kt">string</span> <span class="c">// BASELINE_RECORD</span>
	<span class="n">recs</span>     <span class="n">endpoints</span><span class="o">.</span><span class="n">Records</span>
<span class="p">}</span>

<span class="c">// get returns the *value* of your `BASELINE_RECORD`</span>
<span class="k">func</span> <span class="p">(</span><span class="n">g</span> <span class="o">*</span><span class="n">goDaddy</span><span class="p">)</span> <span class="n">get</span><span class="p">()</span> <span class="p">(</span><span class="kt">string</span><span class="p">,</span> <span class="kt">error</span><span class="p">)</span> <span class="p">{</span>
	<span class="n">r</span><span class="p">,</span> <span class="n">e</span> <span class="o">:=</span> <span class="n">g</span><span class="o">.</span><span class="n">recs</span><span class="o">.</span><span class="n">GetByTypeName</span><span class="p">(</span><span class="s">"A"</span><span class="p">,</span> <span class="n">g</span><span class="o">.</span><span class="n">baseline</span><span class="p">)</span>
	<span class="k">if</span> <span class="n">e</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="k">return</span> <span class="s">""</span><span class="p">,</span> <span class="n">e</span>
	<span class="p">}</span>
	<span class="k">return</span> <span class="n">strings</span><span class="o">.</span><span class="n">TrimSpace</span><span class="p">((</span><span class="o">*</span><span class="n">r</span><span class="p">)[</span><span class="m">0</span><span class="p">]</span><span class="o">.</span><span class="n">Data</span><span class="p">),</span> <span class="no">nil</span>
<span class="p">}</span>

<span class="c">// update sets all A records to your new public IP</span>
<span class="k">func</span> <span class="p">(</span><span class="n">g</span> <span class="n">goDaddy</span><span class="p">)</span> <span class="n">update</span><span class="p">(</span><span class="n">newIP</span> <span class="kt">string</span><span class="p">)</span> <span class="kt">error</span> <span class="p">{</span>
	<span class="n">r</span><span class="p">,</span> <span class="n">e</span> <span class="o">:=</span> <span class="n">g</span><span class="o">.</span><span class="n">recs</span><span class="o">.</span><span class="n">GetByType</span><span class="p">(</span><span class="s">"A"</span><span class="p">)</span>
	<span class="k">if</span> <span class="n">e</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
		<span class="k">return</span> <span class="n">e</span>
	<span class="p">}</span>
	<span class="k">for</span> <span class="n">_</span><span class="p">,</span> <span class="n">d</span> <span class="o">:=</span> <span class="k">range</span> <span class="o">*</span><span class="n">r</span> <span class="p">{</span>
		<span class="k">if</span> <span class="n">e</span> <span class="o">:=</span> <span class="n">g</span><span class="o">.</span><span class="n">recs</span><span class="o">.</span><span class="n">SetValue</span><span class="p">(</span><span class="n">d</span><span class="o">.</span><span class="n">Type</span><span class="p">,</span> <span class="n">d</span><span class="o">.</span><span class="n">Name</span><span class="p">,</span> <span class="n">newIP</span><span class="p">);</span> <span class="n">e</span> <span class="o">!=</span> <span class="no">nil</span> <span class="p">{</span>
			<span class="k">return</span> <span class="n">errors</span><span class="o">.</span><span class="n">New</span><span class="p">(</span><span class="s">"Error setting record: "</span> <span class="o">+</span> <span class="n">d</span><span class="o">.</span><span class="n">Name</span><span class="p">)</span>
		<span class="p">}</span>
	<span class="p">}</span>
	<span class="k">return</span> <span class="no">nil</span>
<span class="p">}</span>

</code></pre></div></div>
</div>
