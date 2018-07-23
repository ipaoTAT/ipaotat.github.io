---
layout: post
title: Ryu控制器源码-Openflow控制器启动
comments: true
tags: [SDN,Ryu,openflow]
header-img: img/post-bg-os-metro.jpg
---

### 主要类：
#### 1. RyuApp
该类其实是个基类，其他的app都要继承自该类。该类的主要成员是event_handlers和observers。event_handlers里存放event和handler的映射关系，在openflow里，event和ofp消息一一对应。因此当ofp消息到达时意味着触发一个event，而app会执行event_handlers对应的handler们。
observer是用来通知其他app的，observers本质也是一个个app，这些注册的app都会收到该event，并且使用其event_handlers对应的handler们处理该event。
#### 2. OpenFlowController
该类实现了openflow控制器的行为。该类实例化并被调用（__call__被执行）后会启动一个socket监听（ssl_cert, ssl 或tcp），然后为accep的t连接实例化一个DataPath对象（datapath_connection_factory函数）并调用它的server方法启动这个DataPath的处理流程。
#### 3. Datapath
该类实现了openflow连接行为。该类负责读写openflow消息，并将其对应为一个event，分发给一个name为ofp_event的类，该类其实是OFPHandler，它也是一个RyuApp的子类。
#### 4. OFPHandler
它是一个RyuApp，它是openflow控制器app树的根，所有的openflow消息都通过改类进行第一手处理，然后才以obervers的方式通知其他app。该类同时还持有一个OpenFlowController对象，意味着通过启动OFPHandler就可以启动openflow控制器。
### 主要流程
#### 1. controller启动
运行命令行ryu run会默认加载OFPHandler，并调用该实例的start方法。
```python
# ryu/cmd/manager.py: main
if not app_lists:
        app_lists = ['ryu.controller.ofp_handler']  //如未指定app，则默认启动openflow controller

    app_mgr = AppManager.get_instance() // 实例化appmanager，该类为app的管理类
    app_mgr.load_apps(app_lists)        // 加载app类
    contexts = app_mgr.create_contexts() // 实例化app
    services = []
    services.extend(app_mgr.instantiate_apps(**contexts)) // 启动app，调用app.start方法
```
针对ofp_handler这样一个app，其start方法除了启动其app，用于处理event外，还实例化并启动了一个OpenFlowController类（通过调用__call__方法）。
```python
# ryu/controller/ofp_handler.py
    def start(self):
        super(OFPHandler, self).start()
        self.controller = OpenFlowController()
        return hub.spawn(self.controller)
```
OpenFlowController类的__call__方法实际就是启动了一个listen，即openflow控制器server的监听，用于等待和处理openflow交换机发起的连接。
```python
# ryu/controller/controller.py
    def __call__(self):
        # LOG.debug('call')
        for address in CONF.ofp_switch_address_list:
            addr = tuple(_split_addr(address))
            self.spawn_client_loop(addr)

        self.server_loop(self.ofp_tcp_listen_port,
                         self.ofp_ssl_listen_port)

    def server_loop(self, ofp_tcp_listen_port, ofp_ssl_listen_port):
        if CONF.ctl_privkey is not None and CONF.ctl_cert is not None:
            if CONF.ca_certs is not None:
                // ssl_cert类型监听
                server = StreamServer((CONF.ofp_listen_host,
                                       ofp_ssl_listen_port),
                                      datapath_connection_factory,
                                      keyfile=CONF.ctl_privkey,
                                      certfile=CONF.ctl_cert,
                                      cert_reqs=ssl.CERT_REQUIRED,
                                      ca_certs=CONF.ca_certs,
                                      ssl_version=ssl.PROTOCOL_TLSv1)   
            else:
                // ssl 类型监听
                server = StreamServer((CONF.ofp_listen_host,
                                       ofp_ssl_listen_port),
                                      datapath_connection_factory,
                                      keyfile=CONF.ctl_privkey,
                                      certfile=CONF.ctl_cert,
                                      ssl_version=ssl.PROTOCOL_TLSv1)
        else:
            // tcp类型监听
            server = StreamServer((CONF.ofp_listen_host,
                                   ofp_tcp_listen_port),
                                  datapath_connection_factory)

        # LOG.debug('loop')
        server.serve_forever()  // 开始循环accept连接, 并调用handle处理新连接。

#hub.py
        def serve_forever(self):
            while True:
                sock, addr = self.server.accept()
                spawn(self.handle, sock, addr)
```
#### 2. 连接建立
在OpenFlowController类中，当新连接来临，上述accept会被唤醒，接着调用OpenFlowController类的handle方法成员。该方法实际就是创建StreamServer时传入的datapath_connection_factory方法。该方法实际为新连接创建了一个Datapath对象，用于处理该openflow连接。
```python
# ryu/controller/controller.py
# 参数为新连接的socker和对端地址
def datapath_connection_factory(socket, address):
    LOG.debug('connected socket:%s address:%s', socket, address)
    # 实例化Datapath对象
    with contextlib.closing(Datapath(socket, address)) as datapath:
        try:
            # 调用serve方法处理新连接的消息
            datapath.serve()
        except:
            # Something went wrong.
            # Especially malicious switch can send malformed packet,
            # the parser raise exception.
            # Can we do anything more graceful?
            if datapath.id is None:
                dpid_str = "%s" % datapath.id
            else:
                dpid_str = dpid_to_str(datapath.id)
            LOG.error("Error in the datapath %s from %s", dpid_str, address)
            raise

```
#### 3.消息分发和处理
serve方法启动了两个协程， _send_loop用于从队列里读取要发送的消息，并实际write到socket；_echo_request_loop用于定时发送echo消息，为openflow连接保活。然后serve方法进入_recv_loop循环读取消息。
```python
# ryu/controller/controller.py
    def serve(self):
        send_thr = hub.spawn(self._send_loop)

        # send hello message immediately
        hello = self.ofproto_parser.OFPHello(self)
        self.send_msg(hello)

        echo_thr = hub.spawn(self._echo_request_loop)

        try:
            self._recv_loop()
        finally:
            hub.kill(send_thr)
            hub.kill(echo_thr)
            hub.joinall([send_thr, echo_thr])
            self.is_active = False
```
_recv_loop循环从socket里readopenflow消息，并将改消息对应成一个event分发给其成员ofp_brick（实际为OFPHandler对象）的obervers，然后调用ofp_brick的handlers处理该消息。（为什么是先send给observers再自己处理？）
```python
# ryu/controller/controller.py
 msg = ofproto_parser.msg(
                    self, version, msg_type, msg_len, xid, buf[:msg_len])
                # LOG.debug('queue msg %s cls %s', msg, msg.__class__)
                if msg:
                    ev = ofp_event.ofp_msg_to_ev(msg) # 消息到event
                    self.ofp_brick.send_event_to_observers(ev, self.state)

                    dispatchers = lambda x: x.callers[ev.__class__].dispatchers
                    handlers = [handler for handler in
                                self.ofp_brick.get_handlers(ev) if
                                self.state in dispatchers(handler)]
                    for handler in handlers:
                        handler(ev)
```
