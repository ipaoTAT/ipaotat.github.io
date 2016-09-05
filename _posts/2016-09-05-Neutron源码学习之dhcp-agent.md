### Overview

dhcp agentΪʵ���ṩ����dhcp����ÿ�����������Ӧ��һ��dhcp������neutron dhcp agent�б�ʾΪһ��������dhcp����dnsmasq����namespace�������ڸ�namespace�л�������dhcp����ӿ��豸��tap�豸����
��l3 agentһ����dhcp agent��neutron server�Ľ���ͨ������message queue��RPC���С�dhcp agent��neutron server����ȡ�Լ���network���ã��Ա�����ڵ��namespace�������ã�ͬʱneutron server����֪ͨdhcp agent������network���õı仯��
**ע�⣺**ֻ�п�����dhcp���ܵ�network�Żᴴ��ʵ�ʵ�dhcp namespace��

### 1.agent����
dhcp agent������Ҳ�Ǵ���һ��oslo_messaging��service����������������֪������service�Ĺؼ����ڴ����manager��manager��ʵ�ֵķ�����ֱ�ӱ�¶��rpc client�����ˡ�__����ͷ�ķ���������dhcp agent����ʱ�������neutron.agent.dhcp.agent.DhcpAgentWithStateReport�࣬
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
DhcpAgentWithStateReport��̳���DhcpAgent��ķ�����ͬʱ���ᶨʱ��neutron server�ϱ���״̬����״̬��ǰֻ�����ж�agent�Ĵ��״����
��l3 agentһ����dhcp agent����֮����Ա����Ѿ��д��dhcp���г�ʼ��������ͬ���ǣ�l3 agent��vrouter��namespace����׽�д�vrouter����dhcp agentʹ��state path�е�network״̬Ŀ¼����¼����Ϊnetwork���Ǳ���namespace����l3 agent��¼��ǰvrouter��״̬��Ϣ����RouterInfo�����ֵ����洢��dhcp agentͨ��NetworkCache�������洢����DhcpAgentWithStateReport�ĸ���DhcpAgent��__init__���������networkchache�ĳ�ʼ����
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

Dhcp Agent����ʱ����neutron server����rpc�����Ի�ȡ��ǰ������ڵ��ϵ�network��Ϣ������ȫ�����¡���������ᱻִ�����Σ���һ����create��ʱ��service����manager������DhcpAgentWithStateReport��__init__�����н��С�
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
sync_state����networks��Ϊ�����������ò���ΪNoneʱ��dhcp agent����ȡ���б��ڵ��network���и��¡���һ��network��ȫ�����·�����agent��rpc��������ʱ��������service.launch(cfg.CONF, server)�У��õ������ջ����neutron_service.Service��start������
```python
# neutron/service.py
    def start(self):
        self.manager.init_host()
        #...

# neutron/agent/dhcp_agent.py
    def init_host(self):
        self.sync_state()
```
��dhcp agent�����ڼ�Ҳ�����sync_state�������������϶���networks������Ϊ�գ���ֻ����һ����network���������������ĳЩ�����׳�Exceptionʱ��

Neutron Server��dhcp agent��RPC������������notify��������neutron/api/rpc/agentnotifiers/dhcp_rpc_agent_api.py�С�����network_removed_from_agent��network_added_to_agent��agent_updated�����ӿ��⣬DhcpAgentNotifyAPI�໹������һ��notify method�б��Լ�һ��ͨ�õ�notify������notify��������method�б��е�'.'�滻Ϊ'_'�����notify��
RPC|����
---|---
agent_updated|
network.create.end|����network
network.update.end|����network
network.delete.end|ɾ��network
subnet.create.end|
subnet.update.end|
subnet.delete.end|
port.create.end|
port.update.end|
port.delete.end|
network_removed_from_agent|����network.delete.end���
network_added_to_agent|����network.create.end���

����Ӧ��RPC��������DhcpAgent�ж���ʵ�֡�