### Overviews
Neutron中使用metadata agent来为虚机提供元数据服务，包括用户名密码，磁盘分区信息等。然而虚机并不是直接与metadata agent交互，而是通过metadata proxy。metadata proxy与metadata agent通过本地Socket通信，所以metadata proxy必须与metadata agent启动在用一个节点上。

在Neutron中，metadata proxy启动在每个vpc内，该vpc内的虚机通过一个虚拟的ip地址（169.254.169.254）访问proxy。proxy的启动位置默认有如下三个规则：
1. 所有的router的namespace中都会启动proxy，监听其9697端口。同时在router的iptables配置将169.254.169.254:80的请求重定向到本地的9697端口。
2. 没有绑定网关的network的namespace中会启动proxy，监听其80端口。
3. 配置了（force_metadata=True）的dhcp agent将启动proxy，监听其80端口。
作为配合，绑定了网关的network会为其虚机注入目标地址169.254.169.254，下一条为网关ip的路由表，而未绑定网关和force_metadata的network注入该路由下一条是dhcp地址。

### Router启动proxy

在l3 agent初始化的时候，会初始化一个metadata_driver实例：
```python
# neutron/agent/l3/agent.py:L3NATAgent
    def __init__(self, host, conf=None):
        #...
        if self.conf.enable_metadata_proxy:
            self.metadata_driver = metadata_driver.MetadataDriver(self)
```
该实例在init方法中，注册了两个回调函数
```python
# neutron/agent/metadata/driver.py
    def __init__(self, l3_agent):
        self.metadata_port = l3_agent.conf.metadata_port
        self.metadata_access_mark = l3_agent.conf.metadata_access_mark
        registry.subscribe(
            after_router_added, resources.ROUTER, events.AFTER_CREATE)
        registry.subscribe(
            before_router_removed, resources.ROUTER, events.BEFORE_DELETE)
```
可以看出，这两个函数分别对应于router资源的创建后和删除前。
```python
# neutron/agent/metadata/driver.py
def after_router_added(resource, event, l3_agent, **kwargs):
    router = kwargs['router']
    proxy = l3_agent.metadata_driver
    for c, r in proxy.metadata_filter_rules(proxy.metadata_port,
                                           proxy.metadata_access_mark):
        router.iptables_manager.ipv4['filter'].add_rule(c, r)
    for c, r in proxy.metadata_mangle_rules(proxy.metadata_access_mark):
        router.iptables_manager.ipv4['mangle'].add_rule(c, r)
    for c, r in proxy.metadata_nat_rules(proxy.metadata_port):
        router.iptables_manager.ipv4['nat'].add_rule(c, r)
    router.iptables_manager.apply()

    if not router.is_ha:
        proxy.spawn_monitored_metadata_proxy(
            l3_agent.process_monitor,
            router.ns_name,
            proxy.metadata_port,
            l3_agent.conf,
            router_id=router.router_id)
```