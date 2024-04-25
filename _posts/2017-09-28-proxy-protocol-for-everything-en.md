---
layout: post
title: "Docker + HAProxy = PROXY protocol for Everything"
date: 2017-09-28 00:00:00 +0800
categories: english
image: /assets/images/2017-09-28-proxy-protocol-for-everything/1.png
tags: docker haproxy devop proxy
---

For cantonese version, please check [here]({% post_url 2017-09-28-proxy-protocol-for-everything %})

Originally posted on [Lakoo's medium](https://m.lakoo.com/docker-haproxy-proxy-protocol-for-everyone-f0da0533d87b) on 2017-09-28, translated to english

MMORPG Experience Sharing / How to make any TCP network service in a container support PROXY protocol using Docker networking.

Background: Our mobile MMORPG Teon server is located in AWS Japan. Recently, some Taiwanese players have reported network issues and suspected problems with certain ISP routes. Relocating the server is too cumbersome, so we want to set up a proxy server in Taiwan to provide reliable and stable connections to all players.

Issue: The game server needs to record players' IP addresses and cannot lose player IP information through proxy NAT.

In general, HTTP proxies use headers like X-Forwarded-for to preserve the original connection's IP information, but this approach is not applicable in a pure TCP environment.

Traditional transparent proxy configurations require kernel support for TPROXY and modification of the default gateway. However, even when using a cross-AWS GCP VPN network, the EC2/VPC gateway settings do not support specifying a server outside of AWS as the internet gateway.

Solution:

![https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/](/assets/images/2017-09-28-proxy-protocol-for-everything/1.png)
uses HAProxy + PROXY protocol as a transparent proxy.

First, within the game server's docker network, we launch an HAProxy container to act as the default gateway for the entire docker network.

{% highlight yaml %}
haproxy:
image: tombull/haproxy
links: - game-server-container # game server's container name
ports: - "8000:8000"
cap_add: - ALL # ALL is for demo lazy purpose only
environment:
HAPROXY_PORTS=8000
networks:
teon-net:
ipv4_address: 172.20.0.10
volumes: - ./haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg
networks:
teon-net:
driver: bridge
ipam:
config: - subnet: 172.20.0.0/16
gateway: 172.20.0.1
{% endhighlight %}

The image uses https://hub.docker.com/r/tombull/haproxy/, which already includes the necessary iptable rules. You just need to configure net.ipv4.ip_nonlocal_bind and NET_ADMIN capabilities.

Since docker-compose currently doesn't have a way to specify individual container gateways, you need to enable NET_ADMIN within the game server's container and run an ip command during container entry to set the gateway.

{% highlight bash %}
ip route delete default
ip route add default via 172.20.0.10
{% endhighlight %}

In HAProxy, open a port that accepts the PROXY protocol and overwrite the original connection's IP when connecting to the game server, achieving the effect of a transparent proxy.

{% highlight yaml %}
frontend game-proxy
mode tcp
option tcplog
option clitcpka
bind 0.0.0.0:8000 accept-proxy
default_backend teon-servers

backend game-servers
option tcplog
mode tcp
source 0.0.0.0 usesrc clientip # overwrite src ip
server game game-server-container:8080
{% endhighlight %}

Since the game server's container specifies HAProxy as the default gateway, HAProxy will handle the subsequent NAT issues on inbound connections. As for outbound connections, additional iptable NAT rules are required:

{% highlight bash %}

# Not sure why this is needed for outbound

# `! -d` means do not apply for destination 172.20.0.0/16

iptables -t nat -A POSTROUTING ! -d 172.20.0.0/16 -o eth0 -j MASQUERADE
{% endhighlight %}

{% highlight yaml %}
backend reverse-proxy
mode tcp
server aws-haproxy aws-ip-address-here:8000 send-proxy
{% endhighlight %}

By connecting to the Taiwan HAProxy, anyone will be reverse-proxied to the AWS HAProxy and then connected to the game server. By controlling the number of HAProxy instances and the backend targets, you can preserve the original IP and have control over routing.

The above architecture, implemented using Docker networking, offers portability advantages:

It avoids the need to manage iptables and routing for each backend or proxy by creating separate VMs, reducing startup costs, and facilitating rapid deployment and modification.
Architecture can be managed on a per-docker-compose/stack basis. Conceptually, any TCP-based docker application can be treated as a single application supporting the PROXY protocol, without considering network and proxy architecture issuesThe article you mentioned discusses how to use Docker networking and HAProxy to support the PROXY protocol for TCP network services running in containers. The PROXY protocol allows preserving the original IP address information when using proxies in a TCP environment.

Conclusion:
The PROXY protocol, as a technology for preserving the source IP in proxies, has fewer limitations compared to the traditional TPROXY technique. However, it seems that support for PROXY protocol beyond the HTTP proxy level is still lacking, especially considering that AWS ELB already supports it (supported list).

The above solution using Docker with HAProxy to support the PROXY protocol aims to help everyone in environments other than web servers to easily benefit from the advantages of the PROXY protocol.

Additional read:

[https://www.haproxy.com/blog/haproxy/proxy-protocol/](https://www.haproxy.com/blog/haproxy/proxy-protocol/)

[https://www.haproxy.com/blog/preserve-source-ip-address-despite-reverse-proxies/](https://www.haproxy.com/blog/preserve-source-ip-address-despite-reverse-proxies/)

[https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/](https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/)
