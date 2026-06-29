# BunkerWeb Tips and Tricks
This repo is to document and show ways of manipulating bunkerweb to provide you the best reverse proxy experience.

My bunkerweb is installed using the basic setup for linux. (Not docker, K8s or Swarm)

I'm quite new to BunkerWeb, but I can see there's a lot you can do in the UI. or you can even skip the UI and do everything in the configs sections. Like adding additional locations to services and stuff. Or managing various headers in terms of passthrough, overwriting and setting. Below are just some things I did to make it work for me and how i want it to.

I found it best to use the Basic Setup for linux so you can read the code, see configs and change and test accordingly. 

## Table of Contents

<!--ts-->
   * [Intro](#BunkerWeb-Tips-and-Tricks)
   * [Table of Contents](#Table-of-Contents)

   * [nginx testing config, applying temporary custom config. (A faster way of testing)](#nginx-testing-config,-applying-temporary-custom-config.-(A-faster-way-of-testing))
   * [444 Default Server Response](#444-Default-Server-Response)
   * [444 Default Server Response shows a certificate error](#444-Default-Server-Response-shows-a-certificate-error)
   * [Vaultwarden (and maybe Bitwarden) Login/2fa issues when HTTP errors are intercepted](#Vaultwarden-(and-maybe-Bitwarden)-Login/2fa-issues-when-HTTP-errors-are-intercepted)
   * [Home Assistant Add On Web UI's not working. 400 responses and "URL / does not match base url"](#Home-Assistant-Add-On-Web-UI's-not-working.-400-responses-and-"URL-/-does-not-match-base-url")


<!--te-->


### nginx testing config, applying temporary custom config. (A faster way of testing)

You can make changes to the nginx config (Everything in `/etc/nginx/` essentially) and reload nginx with `nginx -s reload` and parse configs with `nginx -t`. This allows you to test stuff behind BunkerWeb's back. However, once you change something, the scheduler will recreate everything. so no fear of losing original configs. unless you forgot what you change in BunkerWeb's UI.

This process also works for configs on the "configs" tab on the WebUI. These configs are in `/etc/bunkerweb/configs/`

This is really beneficial as you can quickly make config changes by changing files. Test and restart the webserver and test if it works how you want. without waiting for BunkerWeb to tell the scheduler, the scheduler to schedule and then recreate config, then reload the service, etc, etc. The WebUI can be slow for that sort of stuff. so doing stuff this way and then figuring out how to set that in the UI felt a lot quicker to me.

I also found that when the config files didn't work, the notifications on the webpage were unclear if new changes were applied or if the old notification about nginx config test failed.


### 444 Default Server Response
I migrated from nginx to bunkerweb, which uses nginx under the hood. I had set my original reverse proxy up to server a 444 for anything that was not listed as a server name.

When I migrated to bunkerweb it was finicky getting it exactly how I wanted it. This may seem superfluous, but I believe in technology being our tools, and we should make it do what we want if possible.

Below are the global rules. I want 403 for anyone doing anything naughty according to bunkerweb on services/server names specified. I don't want bunkerweb showing any webpages for default server situations and preferably respond with a 444.

<img width="1628" height="792" alt="image" src="https://github.com/user-attachments/assets/20d0a08c-c2e0-40a6-a34c-62fbceac5ace" />

Having HTTP2/3 enabled shows a response I didn't like in the browser indicating an http2/3 error due to abruptly closed connection. So disabled those to get it closer to my original configuration with a "no response" page.

I could have bunkerweb regard every security violation and default server responses as a 403 or a 444 from this global settings page. It's an either or. And includes how protected services. I want 403 for protected services and 444 for default server.

To get my desired setup, I changed this file:
`/usr/share/bunkerweb/confs/default-server-http.conf`

In this file on line 90. I added `return 444;`

<img width="1156" height="437" alt="image" src="https://github.com/user-attachments/assets/c7372d6a-cd3e-4951-b66a-751db1fe3f78" />


And that's all there is for this.

### 444 Default Server Response shows a certificate error

New issue. When default server loads. Even though I've specified my certificates in the global settings. When loading default server, It would show a warning for www.example.com cert's not aligning to my domain.

After further investigation. The certs are hard coded in the file `/usr/share/bunkerweb/confs/default-server-http.conf` @ line 61

Here is the default cert's commented and my certs listed:
<img width="1525" height="677" alt="image" src="https://github.com/user-attachments/assets/687f30bc-2aa3-4217-b237-3aaa9d4fd4de" />

### Vaultwarden (and maybe Bitwarden) Login/2fa issues when HTTP errors are intercepted

When you log into Vaultwarden/Bitwarden and 2fa is set. You get a document with json body response but the http code on that result is a 400. So if you have intercept HTTP errors enabled. This document gets intercepted and you get bunkerweb's 403 page instead of a json bodied response.

This breaks the ability to log in.

Bunkerity's discord support advise the best solution is to disable the intercept specifically for error 400. I really like the intercept web pages though. They look quite professional like a cloudflare WAF page and would miss it even if it was for a single HTTP code.

So the next best thing I found to do was disable intercept for only specific URI's.

<img width="1292" height="293" alt="image" src="https://github.com/user-attachments/assets/02e94b10-83a5-4cd3-a401-a55954a34928" />

As you can see above, you can set this in the config section. Just make sure you include a the trailing slash if required. As that can also cause issues and keep the log in from working.

### Home Assistant Add On Web UI's not working. 400 responses and "URL / does not match base url"

If you've got home assistant and you're running addon's that have their own UI. In my case i had ESPHome and Home-Assistant-Matter-Hub. But if you try and load these through bunkerweb, they'd fail to load. I was getting error 400's.

After some digging, and changes. it indicated that something wasn't quite right with the headers as after fiddling with the headers. I got an error saying `URL / does not match base url /api/hassio_ingress/4hDFCZAiHsJOAVP5bo4xu655Dqw-l0ntq6arXhyckLE`

So I looked at the nginx config: `/etc/nginx/<SERVICE>/server-http/reverse-proxy.conf`

Inside there was this header:`proxy_set_header X-Forwarded-Prefix "/";`

When I removed that it would work. Great. but... this config is generated by BunkerWeb and can be updated regularly. So... how do we fix this?

Let's look at the code.
In file `/usr/share/bunkerweb/core/reverseproxy/confs/server-http/reverse-proxy.conf` @ row 89 it says this:
```
                        {% if url.startswith("/") %}
        proxy_set_header X-Forwarded-Prefix "{{ url }}";
                        {% endif %}
```
so if "Reverse proxy url" is "/" then this header is set. Huh. How to bypass this.

My solution, Simple but elegant. Encapsulate the URL with quotes.

<img width="1389" height="420" alt="image" src="https://github.com/user-attachments/assets/83cbd359-5a8e-4a49-96ba-c0a4f5e0ed1b" />

