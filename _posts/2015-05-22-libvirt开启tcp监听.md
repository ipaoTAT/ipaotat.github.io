---
layout: post
title: libvirt开启监听tcp
comments: true
category: linux
tags: [libvirt]
---

## 1.设置libvirtd

编辑`/etc/libvirt/libvirtd.conf`

<!-- more -->

```
listen_tls = 0
listen_tcp = 1
auth_tcp="none"
tcp_port = "16509"
```

## 2.设置服务为监听状态

即使设置了`listen_tcp`也不会开启监听服务, 重启`libvirt-bin`服务,验证:

```bash
sudo netstat -nlpt 

# 可见16509端口并没有开启

ps aux | grep libvirt

# 可见libvirtd没有-l参数

```

需要开启监听服务，设置`/etc/init/libvirt-bin.conf`文件，设置`exec /usr/sbin/libvirtd $libvirtd_opts -l`，注意后面的`-l`选项
不能直接写在`libvirtd_opts`上，不生晓，原因不明

## 3. 验证

重启`libvirt-bin`服务, 使用`netstat`是否开启了tcp端口和`ps`查看libvirtd是否有`-l`选项,都没有问题后，运行:

```bash
virsh --connect qemu+tcp://node1/system list
```

其中`node1`为主机名，如果无错误，则表示正常开启tcp监听服务.
