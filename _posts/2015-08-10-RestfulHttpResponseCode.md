---
layout: post
title: HTTP Response Code in RESTful
comments: true
category: 
tags: [HTTP,Response Code,REST,RESTful]
---

1.若客户端发送的请求没有提供正确的认证信息，返回 401（“Unauthorized”），并告知正确的Authorization报头的格式；
2.若客户端访问一个不对应任何资源的URI，返回404（“Not Found”），**除非**：PUT请求表示在这个URI处放置资源；
3.若客户端使用URI不支持的HTTP方法，返回405（“Method Not Allowed”），比如删除（DELETE）只读资源时。（OPTION方法应返回URI支持的方法）；
4.