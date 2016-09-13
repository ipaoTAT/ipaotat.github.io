### Overview

dhcp agent为实例提供三层dhcp服务。每个三层网络对应了一个dhcp服务，在neutron dhcp agent中表示为一个启动了dhcp服务（dnsmasq）的namespace。并且在该namespace中还配置有dhcp服务接口设备（tap设备）。
与l3 agent一样，dhcp agent与neutron server的交互通过基于message queue的RPC进行。dhcp agent从neutron server中拉取自己的network配置，对本网络节点的namespace进行设置；同时neutron server主动通知dhcp agent关于其network配置的变化。
**注意：**只有开启了dhcp功能的network才会创建实际的dhcp namespace！

### 1.agent启动
dhcp agent的启动也是创建一个oslo_messaging的service对象并启动它。我们知道，该service的关键在于传入的manager，manager类实现的方法将直接暴露给rpc client（除了“__”开头的方法？）。dhcp agent启动时传入的是neutron.agent.dhcp.agent.DhcpAgentWithStateReport类，
```python
# neutron/agent/dhcp_agent.py
    def main():
        register_options(cfg.CONF)
        common_config.init(sys.argv[1:])
        config.setup_logging()
        server = neutron_service.Service.create(
            binary='neutron-dhcp-agent',
            topic=topics.DHCP_AGENT,
            report_interval=cfg.CONF.AGENT.report_interval,
            manager='neutron.agent.dhcp.agent.DhcpAgentWithStateReport')
        service.launch(cfg.CONF, server).wait()
```
DhcpAgentWithStateReport类继承了DhcpAgent类的方法的同时，会定时向neutron server上报其状态，该状态当前只用于判断agent的存活状况。
与l3 agent一样，dhcp agent启动之初会对本机已经残存的dhcp进行初始化管理。不同的是，l3 agent用vrouter的namespace来捕捉残存vrouter，而dhcp agent使用state path中的network状态目录来记录（因为network并非必需namespace）。l3 agent记录当前vrouter的状态信息是由RouterInfo对象字典来存储；dhcp agent通过NetworkCache对象来存储。在DhcpAgentWithStateReport的父类DhcpAgent的__init__方法中完成networkchache的初始化。
```python
#neutron/agent/dhcp/agent.py: DhcpAgent
    def __init__(self, host=None, conf=None):
        self.cache = NetworkCache()
        self._populate_networks_cache()
        
    def _populate_networks_cache(self):
        existing_networks = self.dhcp_driver_cls.existing_dhcp_networks(
            self.conf
        )
        for net_id in existing_networks:
            net = dhcp.NetModel(self.conf.use_namespaces,
                                {"id": net_id,
                                 "subnets": [],
                                 "ports": []})
            self.cache.put(net)
```

Dhcp Agent启动时会向neutron server发起rpc请求以获取当前该网络节点上的network信息，进行全量更新。这个动作会被执行两次：第一次是create的时候，service创建manager对象，在DhcpAgentWithStateReport的__init__方法中进行。
```python
# neutron/agent/dhcp_agent.py
    def __init__(self, host=None, conf=None):
        if report_interval:
            self.heartbeat = loopingcall.FixedIntervalLoopingCall(
                self._report_state)
            self.heartbeat.start(interval=report_interval) 
            
    def _report_state(self):
        #...
        if self.agent_state.pop('start_flag', None):
            self.run()
    
    def run(self):
        """Activate the DHCP agent."""
        self.sync_state()
        self.periodic_resync()
```
sync_state接受networks作为参数，但当该参数为None时，dhcp agent会拉取所有本节点的network进行更新。另一次network的全量更新发生在agent的rpc服务启动时，即调用service.launch(cfg.CONF, server)中，该调用最终会调用neutron_service.Service的start方法。
```python
# neutron/service.py
    def start(self):
        self.manager.init_host()
        #...

# neutron/agent/dhcp_agent.py
    def init_host(self):
        self.sync_state()
```
在dhcp agent运行期间也会调用sync_state函数，但基本上都是networks参数不为空，即只更新一部分network。这种情况发生在某些操作抛出Exception时。

Neutron Server对dhcp agent的RPC操作基本都是notify，定义在neutron/api/rpc/agentnotifiers/dhcp_rpc_agent_api.py中。除了network_removed_from_agent、network_added_to_agent和agent_updated三个接口外，DhcpAgentNotifyAPI类还定义了一个notify method列表以及一个通用的notify方法，notify方法负责将method列表中的'.'替换为'_'后进行notify。
RPC|描述
---|---
agent_updated|
network.create.end|创建network
network.update.end|更新network
network.delete.end|删除network
subnet.create.end|
subnet.update.end|
subnet.delete.end|
port.create.end|
port.update.end|
port.delete.end|
network_removed_from_agent|调用network.delete.end完成
network_added_to_agent|调用network.create.end完成

而相应的RPC方法在类DhcpAgent中都有实现。