---
layout: post
title: "魔術！乜叉嘢都支援 PROXY protocol！"
date: 2017-09-28 00:00:00 +0800
categories: cantonese
image: /assets/images/2017-09-28-proxy-protocol-for-everything/1.png
tags: docker haproxy devop proxy
---
For english version, please click [here]({% post_url 2017-09-28-proxy-protocol-for-everything-en %})

Docker + HAProxy = PROXY protocol for EVERYONE

Originally posted on [Lakoo's medium](https://m.lakoo.com/docker-haproxy-proxy-protocol-for-everyone-f0da0533d87b) on 2017-09-28

MMORPG 經驗分享 / How to make any TCP network service in container support PROXY protocol, using docker networking.

背景：
我哋隻 mobile MMORPG Teon 伺服器位於 AWS 日本 。最近有部分台灣玩家反映網絡唔順暢，懷疑係部份 ISP 線路有問題。搬遷伺服器又太麻煩，所以想係台灣起台 proxy 伺服器，提供可靠穩定線路俾所有玩家。

問題：
遊戲伺服器需要記錄玩家 IP，不能夠係 proxy NAT 後損失玩家 IP 資訊。

一般 HTTP Proxy 使用 X-Forwarded-for 之類嘅 Header 保留原連接嘅 IP 資訊，但係純 TCP 環境並不適用。

而傳統 Transparent proxy 的設定，需要 kernel 支援 TPROXY ，亦要修改 default gateway，但即使係用咗跨 AWS GCP 嘅 VPN network， EC2/VPC 的 gateway setting 都並不支援指定 AWS 以外的伺服器作為 Internet gateway。

解決方法：

![https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/](/assets/images/2017-09-28-proxy-protocol-for-everything/1.png)
圖來自 https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/
使用 HAProxy + PROXY protocol 作 transparent proxy。

首先係遊戲伺服器嘅 docker network 內起一個 HAProxy container ，作為整個 docker network 嘅 default gateway。

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

Image 使用 [https://hub.docker.com/r/tombull/haproxy/](https://hub.docker.com/r/tombull/haproxy/ ) ，已經有齊需要嘅 iptable rules，只要設定好 net.ipv4.ip_nonlocal_bind 同 NET_ADMIN 之類嘅 CAP 就可以。

另外因為 docker-compose 暫時未有方法指定個別 container 嘅 gateway ，需要在遊戲伺服器嘅 container 內開啟 NET_ADMIN，同係 container entry 時行 ip command ，設定 gateway。

{% highlight bash %}
ip route delete default
ip route add default via 172.20.0.10
{% endhighlight %}

係 HAProxy 開一個接受 PROXY protocol 嘅埠，連接到遊戲伺服器時覆寫返原連接嘅 IP ，就可以起到 transparent proxy 嘅效果。

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

同時因為遊戲伺服器嘅 container 指定咗 default gateway 係 HAProxy，HAProxy 會自行處理 inbound 後續嘅 NAT 問題。至於 outbound 連接，實測需要增加 iptable NAT rule 處理：

{% highlight bash %}

# Not sure why this is needed for outbound

# `! -d` means do not apply for destination 172.20.0.0/16

iptables -t nat -A POSTROUTING ! -d 172.20.0.0/16 -o eth0 -j MASQUERADE
{% endhighlight %}

最後係台灣 HAProxy 中繼設定

{% highlight yaml %}
backend reverse-proxy
mode tcp
server aws-haproxy aws-ip-address-here:8000 send-proxy
{% endhighlight %}

咁樣任何人只要連接台灣 HAProxy，就會被 reverse proxy 到 AWS 的 HAProxy，再被連接到 game server。 中間只要控制 HAProxy 嘅數量同 backend 目標，就可以做到保存原 ip，自由控制 routing 的效果。

上述架構使用 docker networking 處理嘅好處，主要在於 portability：

避免咗每一個 backend 或者 proxy 都要自行開個 vm 管理 iptable 同 routing ，減少開機成本，同時方便快速部署同修改。
可以以一個 docker-compose/stack 為單位管理 architecture，概念上即係任何使用 TCP 嘅 docker application ，只要使用上面設定方法，架構上都可以當係單個支援 PROXY protocol 嘅 application，而毋須考慮網絡和 proxy 架構問題。
結論：
PROXY protocol 作為 proxy 保存 source ip 嘅技術，比傳統 TPROXY 嘅技術限制少。但係在 aws ELB 早已支援 PROXY protocol 嘅當下， http proxy 層面以外嘅支援似乎還欠奉 (supported list)。

上述利用 docker 配合 HAproxy 支援 PROXY protocol 嘅方案，希望幫助到大家係 web server 以外嘅環境，簡單地利用到 PROXY protocol 嘅好處。

Additional read:

[https://www.haproxy.com/blog/haproxy/proxy-protocol/](https://www.haproxy.com/blog/haproxy/proxy-protocol/)

[https://www.haproxy.com/blog/preserve-source-ip-address-despite-reverse-proxies/](https://www.haproxy.com/blog/preserve-source-ip-address-despite-reverse-proxies/)

[https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/](https://www.haproxy.com/blog/using-haproxy-with-the-proxy-protocol-to-better-secure-your-database/)
