## How to bypass CloudFlare bot protection ?

Several months ago I submitted what appeared to be a security flaw to CloudFalre’s bugbounty program. According to them, this is not a problem, it’s up to you to make up your own mind.
Cloudflare offers a system of JavaScript workers that can be used to execute code on the server side (at Cloudfalre therefore). This feature can be useful for static sites, maintenance pages etc … But it is also a great tool for pentest (serverless C&C, easy phishing proxy etc …). In this post we will explore Cloudflare bot protection bypass.
If you’ve ever tried accessing a site like shodan.io from Tor, you know how annoying these captchas are.
First, we will register a domain (a free .tk domain will be sufficient) and create a Cloudfare account. Once the domain is validated by Cloudflare we need to add at least one valid DNS entry that uses proxy mode.

![This is an image](https://miro.medium.com/max/1400/1*hD8nazwq5c63NIYvyc9STQ.jpeg)

Now we are going to create a JavaScript worker that will fulfill the role of reverse proxy (full code is available on GitHub: https://github.com/trewisscotch/How-to-bypass-CloudFlare-bot-protection-). Create a new worker and copy/paste worker.js content into it. You can customize TOKEN_HEADER, TOKEN_VALUE, HOST_HEADER and IP_HEADER values.
Then add a route to you worker: proxy.domain.com/*

![This is an image](https://miro.medium.com/max/700/1*dnEXzePMsUFmQr0ZnZHaIw.jpeg)

Now, if you try to reach proxy.domain.com, you will see “Welcome to NGINX.”. The JavaScript code is pretty easy to understand, it will look for a specific header (acting as a magic) and will forward your request to the given domain.
To easily use this proxy, a python wrapper is available in my GitHub repository, let’s play with it.

```
>>> from cfproxy import CFProxy
>>> proxy = CFProxy('proxy.domain.com', 'A random User-Agent', '1.2.3.4')
>>> req = proxy.get('https://icanhazip.com')
>>> print(req.status_code)
200
>>> print(req.text)
108.162.229.50

```

You can try to perform WHOIS request on the result, it’s a Cloudflare IP (probably the server running the worker).
At this point if you try to request your proxy through Tor, you will be blocked. We are therefore going to add a rule to our Cloudflare firewall (Firewall / Firewall Rules). There are several ways to get a rule to work, here is one:

![This is an image](https://miro.medium.com/max/700/1*jwunahDnhPoM7czb0F-CDw.jpeg)

Now you are able to request your proxy using tor without any annoying reCaptcha.
At this point you can request any site using Cloudflare (except if they are using “Under Attack Mode” (eg: shodan.io). But there is more with this script.
Try to request a site who display your headers, and you will see:
ACCEPT: */*
ACCEPT-ENCODING: gzip
CDN-LOOP: cloudflare; subreqs=1
CF-CONNECTING-IP: 2a06:98c0:3600::103 (could be any Cloudflare IP)
CF-EW-VIA: 15
CF-RAY: [REDACTED]
CF-REQUEST-ID: [REDACTED]
CF-VISITOR: {&quot;scheme&quot;:&quot;https&quot;}
CF-WORKER: yourdomain.com (OPSEC Warning !)
CONNECTION: Keep-Alive
HOST: www.whatismybrowser.com
USER-AGENT: My Random User-Agent
X-FORWARDED-FOR: 1.2.3.4 (yes, we can override this header with whatever we want !)
X-FORWARDED-PROTO: https
As you can see, X-FORWARDED-FOR can be set to an arbitrary value, so you can bypass IP limitation requests during scrapping or more dangerous IP verification during a login procedure … The origin IP is not forwarded to the website, so the only way to block this kind of request on your server is to filter on CF-WORKER header.
This bypass is not a bug according to Cloudflare:

![This is an image](https://miro.medium.com/max/700/1*GQ_nKk9Lp6WRa73j-wFBRA.jpeg)

So let’s enjoy the 100 000 request/day for your free Cloudflare account and go scrape the world !


