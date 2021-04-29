---
title: "GCP Google IPv6 Kubernetes support"
date: 2021-04-29T10:04:58-07:00
draft: false
tags: ["docker", "linux", "opensource", "kubernetes"]
---
- [Assumptions](#assumptions)
- [Implementation](#implementation)
  - [IPV6 Config](#ipv6-config)
  - [Load Balancer Backend Configuration](#load-balancer-backend-configuration)
  - [LoadBalancer FrontEnd configuration](#loadbalancer-frontend-configuration)
  - [Verification](#verification)
- [Closing Notes](#closing-notes)
  - [HTTPS](#https)

Most technical writes and writers in general tend to write in content in the hope that there will be value in their contribution for a good while.  Usually most of the content I write is mainly for my own purposes to preserve the directions because I struggled to find a solution online.  In other cases, I write because I have something to add to the conversation that I didn't see highlighted in the other technical writing / online chatter.

This is one of those few articles that I hope very much will become obsolete sooner rather then later.  It's getting close to 10 years since IPv6 has launched.  We were supposed to run out and have run out of IPV4 addresses ages ago. Somehow we still manage to squeeze some IPV4 addresses from here and there but we're really way past the time when we should be moving forward. 

There are far too many applications and libraries that still use IPV4 and don't even think about IPv6 networking just yet.  Docker is a great example where it feels like IPv6 was an afterthought.  I wrote an article a while back about the pains of [Docker IPv6](https://www.esamir.com/20/8/26/docker-ipv6-guide/).  Since then, it sounds like they've added a few fixes that make the experience easier to use.  

I do wish IPv6 adoption was a bit more prevalent.  This particular guide is how to get IPv6 working on Kubernetes on GCP.  There is native support for IPv6 on kubernetes but that hasn't made it out to GCP just yet.  Most of VPC's [networking](https://cloud.google.com/vpc/docs/vpc) has yet to add support for IPv6.

To quote the article: "VMs in the VPC network can only send to IPv4 destinations and only receive traffic from IPv4 sources. However, it is possible to create an IPv6 address for a global load balancer."  The latter part is what we're focusing on.  We have a IPV4 based kubernetes cluster where we're using a load balancer to direct traffic from an IPv6 stack to the application we have running.

Lastly although I'm writing this to solve a problem for kubernetes networking not having IPv6 support, this solution doesn't actually rely or event expect kubernetes.  We're simply mapping traffic from one IPv6 address to an IPv4 address.  As long as it's responding than it'll work fine.

## Assumptions

1. You have a running web app in kubernetes deployed using IPV4.
2. You have some familiarity with the GCP platform.

Disclaimer: This is VERY google specific. So it will not translate easily to other cloud providers.  There is likely a similar pattern you can use but you'll have to check with your own cloud provider to see how to make that work.

Enough of all this background, let's get on the the fun stuff.

## Implementation

### IPV6 Config
First step is to create a new external IPv6 [address](https://console.cloud.google.com/networking/addresses/list?).


![ipv6_create ](/images/k8s_ipv6_google/ipv6_create.png)

Once create you should see it in the listing.  You can add a DNS entry for your hostname if you'd like or we can test this by IP if preferred.

![ipv6_address](/images/k8s_ipv6_google/ipv6_address.png)


### Load Balancer Backend Configuration
Create a new HTTP Load Balancer.  Make sure to select HTTP and not TCP or UDP.

![http_lbl](/images/k8s_ipv6_google/http_lb.png)

Most load balancers in google defined a frontend (entrypoint), a backend (usually an internal GCP service to send the request to and routing rules (if needed) to control the flow of data.

First we setup a backend.  We'll create a backend service and select `Internet Network Endpoint Group` for the backend type.  I'm expecting HTTPS traffic so I'll select https as well for the protocol.

![backend_type](/images/k8s_ipv6_google/backend_type.png)

When you create a new instance group you'll have to put in the IP address you want traffic to be directed to.  Here's an example of a ghost-blog with an example IP.


![backend_config](/images/k8s_ipv6_google/backend_configuration.png)

At this point you have all the pieces you need to create your backend.  You can see what my final selection looks like here:


![backend_selection](/images/k8s_ipv6_google/backend_selection.png)

You can enable CDN, healthcheck, logging etc on here as well.  I didn't explore all these settings but feel free to enable/disable whatever you need.  

### LoadBalancer FrontEnd configuration

This part is much easier.  Select the protocol, since I'm routing IPv6 traffic, I chose https, Select the `IP Version` and choose IPv6 and then the drop down should give you all the options available.  Simple select the IP you created.  In my case that would be `blog-example-ip`


![frontend_selection](/images/k8s_ipv6_google/frontend_configuration.png)

### Verification

Finally you can confirm that everything is working with:

```sh
ping6 2600:1901:0:39bc::
```

That of course should respond and be reachable, and the real test is

```sh
curl -6 https://myhostname
```

should respond with your website's content.  

If you don't want to wait for DNS propagation you can add this entry to your hosts file.  Naturally replace this IP with your own IPv6 address.

```sh
2600:1901:0:39bc::	myhostname
```

Once everything is loaded I highly recommended this chrome extension called [IPvFoo](https://chrome.google.com/webstore/detail/ipvfoo/ecanpcehffngcegjmadlcijfolapggal?hl=en).  It'll show you all the IPv4 and IPv6 address associated with your website.

You can see an example for how [GeekBeacon](https://www.geekbeacon.org) looks like.  If you haven't checked out it's a really cool community I'm spending far too much time in.


![ipvfoo](/images/k8s_ipv6_google/ipvfoo.png)


## Closing Notes

### HTTPS
This pattern forwards https -> https which is not usually how a load balancer is setup.  We tend to want to offload the CPU cycles of encryption on the load balancer and the backend usually simply serves HTTP traffic.

My K8 application already handles HTTPS and I don't really want to turn that off.  My hope is that GCP would catch up and enable IPV6 in their GKE implementation sooner rather then later.

Also, I'm utilizing an K8s ingress control that manages https.  The big benefit of relying on K8s rather then GCP is that if at some point I need to migrate to another solution on another cloud provider or hosting internally, the only real requirement is to have an IP + SSL certificate allocated.  Everything else is simply a K8s manifest that needs to be deployed to one destination vs another.  The ability to be cloud agnostic is much preferred to over the savings a few CPU clock cycles.

