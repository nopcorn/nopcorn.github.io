---
layout: post
title: Using DuckDuckGo's Image Proxy as a C2 Channel
author: nopcorn
description: DuckDuckGo's image preview APIs put way too much trust in the user and don't validate that the requests came from authentic browsing behavior.
summary: DuckDuckGo's image preview APIs put way too much trust in the user and don't validate that the requests came from authentic browsing behavior.
tags: [infrastructure, c2, proxy]
---

> - DuckDuckGo's image proxy doesn't allow-list trusted URLs, so anyone can request any image through it.
> - The proxy doesn't sanitize the URI, allowing for any number of request parameters to be relayed to the server hosting the image.
> - This can be leveraged by a malicious actor to covertly communicate with a compromised server using an image-based protocol.
> - Small proof of concept provided: [DuckDuckC2](https://github.com/nopcorn/DuckDuckC2)


[DuckDuckGo](https://duckduckgo.com) (DDG) is a search engine known for its emphasis on user privacy. Unlike many other search engines, it doesn't track users' search history, ensuring a more private online search experience. With its user-friendly interface and commitment to not collecting or sharing personal information, it has garnered a dedicated user base among privacy-conscious individuals.

# Expected Behavior

Conducting an image search via DuckDuckGo results in many requests to the `external-content` subdomain:

![](/assets/img/ddg-image-search-network-loads.png)

This endpoint seems to accept a url as parameter `u` and returns and image. From some of the research I've been conducting on image proxies, I know this domain as Bing's own image proxy (`tse.mm.bing.net` and `tse.explicit.bing.net`). It's not surprising that DuckDuckGo would be leveraging Bing's services as they [source traditional links and images in their search results from the company](https://duckduckgo.com/duckduckgo-help-pages/results/sources/). Scrolling down through multiple pages of image search always shows the same domains being supplied in the `u` parameter. 

I assumed the `external-content` proxy would only be allow-listing `*.bing.net` domains and decided it would be interesting to try and confuse the regular expression that was underpinning this function by registering custom domains with `bing` substrings. Keeping things simple though, I first tested for the presence of the allow-listing by generating a webhook on my testing server and providing it to the proxy...

..and it happily tried to pull the image 🤦

# Probing the API

To my surprise, DuckDuckGo happily accepted the webhook and requested it to receive content. I guess they aren't allow-listing domains. The request my server received looked like this - curiously, the IP it came from doesn't appear in [DuckDuckGo's list of DuckDuckBot IPs](https://duckduckgo.com/duckduckgo-help-pages/results/duckduckbot/), despite providing the `DuckDuckBot/1.1` user-agent:

```
GET /notanimage HTTP/1.1
Host: <MYSERVER>
Accept: */*
connection: close
accept-encoding: gzip
user-agent: DuckDuckBot/1.1; (+http://duckduckgo.com/duckduckbot.html)
```

Probing the `/iu` endpoint a bit more, I discovered that DuckDuckGo returns a `HTTP/2 400` error if the content isn't understood by the proxy (aka it isn't an image in a valid format - likely based on `Content-Type` headers). Alternative HTTP verbs like `POST` and `PUT` get automatically converted by DuckDuckGo's servers to a `GET` for the same endpoint. We know the server software is likely `nginx` based on the banner, so it's possible the location block for the `/iu` endpoint looks something like this:

```
location @force_get {
    proxy_method GET;
    proxy_pass_request_headers off;
    proxy_pass http://ourupstream;
}
```

The proxy *does* pass request parameters to the server hosting the image. I fuzzed the maximum parameter size and it doesn't seem like DuckDuckGo is performing any filtering on content or length. This means that the only limit is from the web servers and clients doing the proxying. I was able to relay 8192 bytes of data without issue and suspect that number isn't the ceiling. Additionally, it seems like multiple DuckDuckGo subdomains support the `/iu` proxy service.

# Using as a C2 Channel

Assuming we control both sides of proxy connection, we can communicate using request parameters and return output via encoded images using the proper `Content-Type` headers. While this can be leveraged in many ways, here's how my proof of concept [DuckDuckC2](https://github.com/nopcorn/DuckDuckC2) works:

![](/assets/img/duckduckc2-flow.png)

It assumes that `server.py` is running somewhere where a shell is desired and that `client.py` needs to communicate with it from reputable IP space (DuckDuckGo). The client can relay tasking for the server via GET parameters and the output the commands are encoded into the image returned as a result. DuckDuckGo happily forwards on the modified image and the client is able to extract the output from it. All of this functionality is wrapped is a pseudo-shell to give the client the impression they are interacting with an actual shell.

# Remediation

Without internal knowledge of DuckDuckGo's strategy for image search, it's difficult to say if this open proxy is a bug or pre-positioning code for integration with future image providers. I would suggest applying an allow-list of domains to the API underpinning this feature - likely something like `*.bing.net` would suffice as it seems all legitimate use of the proxy simply points to Bing. DuckDuckGo does have many other products though, so it's possible there are other uses in other codebases.

Defending against the use of this technique in a network isn't too difficult seeing as most web servers will log request parameters. Regardless of the directionality (whether the client or server is in your network), attackers will have to go to great lengths to obfuscate commands and responses from network-aware security solutions. 




