---
layout: post
title: Haproxy配置HTTPReferer为Host
comments: true
category: 技术
tags: [haproxy,httpserver]
---


```python

frontend http-in
    bind *:80
    default_backend servers
backend servers
    mod http

    http-request set-heaser Referer http://%[req.hdr(host)]

    server servername ip:port weight 1 max conn 100000 check inter 1s

```
