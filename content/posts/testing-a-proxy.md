---
title: "Testing a reverse http proxy"
date: 2020-08-03T14:05:25-03:00
draft: false
tags: ["proxy","development"]
---

Did you eve had trouble with a proxy config? Found that is not simple to test a proxy locally?. Why this? simple, I found many times deploying a proxy configuration to production, even a small change like adding a new path, a redirection or just updating ssl certs, can lead to a disruption or an outage because sometimes it's hard to properly test it before pushing the configuration. So I found a couple od tricks that can help test locally in our laptops or machine before even pushing the code or config giving a safer path to deploy a proxy config.

# What should I know beforehand?

This can be used to test any kind of reverse proxy so I won't go into details of what proxy can be used, what proxy is running is irrelevant of what I want to test here. Also, normally a reverse proxy has backends and sometimes those backends depend on service discovery like consul or kubernetes, this part is out of the scope of the post so I will focus on the testing part not configuring the proxy.  
When testing a proxy we need to gather the configuration details or at least what should be the behavior of it, specially around routing the requests to the right backends.  
We should know the behavior of the proxy beforehand as testing might be a bit different for each use case.

- Is the routing path based, hostname based, or both?
- Is it plain http or https?
- Is the proxy terminating tls or passing through ssl to the backends?

This is not an extensive list but the will give a pretty good understanding how to test it locally on our development machine. The primary issue when testing a proxy locally is that the dns address of the proxy is not the same as production, but we can simmulate it very easy and debug locally before pushing to staging or production.  
One thing to have in mind is that it's much easier to test a proxy running plain http, I will explain that a bit, but with todays requirements running something in plain http without redirecting to https is a major design flaw as https is pretty much a standard way of communicationg today. I will explain http, but If you need to test plain http then means that something needs to be reviewed in you proxy config.  
ALL following tests asume we are running the proxy locally in our machine and that is bound to localhost or 127.0.0.x and using standard http (80) and https (443) ports. Also I am asuming all http methods are `GET` requests, but this can be implemented with all valid http methods, like `POST`, `PUT`, `DELETE`, etc.

### Testing plain http

As metioned the easiest way is to test is plain http, and the reason is that with plain http there is no tls handshake (will explain more later) and just by simple doing the request against the proxy and setting some header will help the routing. For testing we will use `curl` which is a command line tool easy to use and available in most operating systems.  
There are 2 approaches for http, if the routing is path based like `http://proxy/service1` or `http://proxy/service2`, this one is easy as the routing doesn't depend on the hostname so we can use `localhost` or `127.0.0.1` as the proxy address.

```bash
# PATH BASED ROUTING

# To test routing to service 1
$ curl -v http://localhost/service1

# To test routing to service 2
$ curl -v http://localhost/service2
```
But if the routing at the proxy is based on hostname either then `localhost` or `127.0.0.1` won't work, because in the http request the `Host` header will be set to localhost or 127.0.0.1 then the proxy won't know where to route the request to, or will route the request to the default backend, but that depends on how the proxy configuration was set up. To actually help routing based on host we will need to tell curl to override the host header.  
Now lets imagine I have service1.example.com and service2.example.com configured in the reverse proxy that need to be routed to the proper service, this time we would still use the localhost address for the proxy but overriding the `Host` header.
```bash
# HOST BASED ROUTING

# To test routing to service 1
$ curl -v -H "Host: service1.example.com" http://localhost/

# To test routing to service 2
$ curl -v -H "Host: service2.example.com" http://localhost/
```
Both path and host based routing can be combined, but if there is a host based config we will always have to set the `Host` header so the proxy knows what is the host part of the request.
```bash
# HOST AND PATH BASED ROUTING

# To test routing to service 1 on path1
$ curl -v -H "Host: service1.example.com" http://localhost/path1

# To test routing to service 1 on path2
$ curl -v -H "Host: service1.example.com" http://localhost/path2


# To test routing to service 2 on path1
$ curl -v -H "Host: service2.example.com" http://localhost/path1

# To test routing to service 2 on path2
$ curl -v -H "Host: service2.example.com" http://localhost/path2
```
This last one both service1 and service2 can implement the same paths as the proxy will know how to route them based on the `Host` header of the http request.

### Testing with https

When adding https into the equation things get a bit more complex, however they are easy to test using curl, but we need to know the differences to properly test https targets in a reverse proxy.  
The primary problem with https is that the tls/ssl handshake happens before any http packets are exhanged, and since the `Host` header comes in the http request, then when establishing the https connection the proxy won't know what is the host, this is when SNI (Server Name Indication) comes to play, it's an extension for the tls protocol to be able to serve multiple domains on tls under the same IP and still provide the correct certificates to the right domain names.  
In order to do so we need to "fake" the dns resolution to the localhost ip address a.k.a `127.0.0.1` in order for things to match.  
Other problem to test locally with https, is that certificates are bound to the dns name, even if using a wildcard certificate the request need to be done to the right domain otherwise it will not work and error saying a domain name missmatch, we can workaroud the issue by using the `-k` flag in curl which won't verify the domain cert and authenticity but, since we are testing end to end I don't recomend to use it in this particular case, specially if the proxy change was to modify the certs.  
This use case problem comes to play when the proxy is terminating https, there might be some other things to consider if the proxy is doing passthrough of https (normally a.k.a running in tcp mode), this last mode has some other things to consider when it comes to the proxy config and the routing configuration that are outside of the scope of this post. But testing the proxy behavior should be the same for both use cases as the client side will pretty much look the same.  
So basically the easiest way to test with curl is to fake the dns resolution which can be done in recent versions of curl with the `--resolve` flag. Again lets consider 2 services service1 and service2, I will show both path based routing and host based is pretty much the same way with https.

```bash
# PATH BASED ROUTING on HTTPS

# To test routing to service 1
$ curl -v --resolve example.com:443:127.0.0.1 https://example.com/service1

# To test routing to service 2
$ curl -v --resolve example.com:443:127.0.0.1 https://example.com/service2
```

The `--resolve` flag expects 3 args separated by colons, full hostname, port, and ipaddress (in this case the localhost address 127.0.0.1). Since we are faking the dns resolution to our localhost, now we can use the same address as we would use in staging or production, and the request will look similar and domains will match in the request making the ssl certificates and routing based on SNI work as expected in the proxy.  
Let's look at host based routing with https.

```bash
# HOST BASED ROUTING on HTTPS

# To test routing to service 1
$ curl -v --resolve service1.example.com:443:127.0.0.1 https://service1.example.com/

# To test routing to service 2
$ curl -v --resolve service2.example.com:443:127.0.0.1 https://service2.example.com/
```

And also the same approach can be used with https using host and path routing

```bash
# HOST and PATH BASED ROUTING on HTTPS

# To test routing to service 1 and path 1
$ curl -v --resolve service1.example.com:443:127.0.0.1 https://service1.example.com/path1

# To test routing to service 1 and path 2
$ curl -v --resolve service1.example.com:443:127.0.0.1 https://service1.example.com/path2


# To test routing to service 2 and path 1
$ curl -v --resolve service2.example.com:443:127.0.0.1 https://service2.example.com/path1

# To test routing to service 2 and path 2
$ curl -v --resolve service2.example.com:443:127.0.0.1 https://service2.example.com/path2
```

Nota as this last use case, for https and http path1 and path2 could be also different backends but that depends on the proxy configuration.  
Also another good point is that using `--resolve` flag can work also for http, and if we use this method there is no need to override the `Host` header as that header will be set to the right value when faking the resolution.  
There are other ways of faking the resolution to localhost like modifying the `/etc/hosts` file adding the domains we need to test, but using all flags available in curl seems much easier and cleaner.

# Conlusion

In this post I explained how to test a reverse proxy configuration locally in a development machine, this gives more flexibility to test and debug the proxy behavior without modifying staging ot production services and giving agility to the debug process without disrupting. This tips came handy to me testing proxies configs before pushing them as code or to the servers saving me some disruptions in some cases where I pushed the wrong config. But remember you need to be careful with this tips as there might be other moving parts in your setup that you might be missing, so always refer to the documentation of your setup and the documentation of the proxy you are testing.