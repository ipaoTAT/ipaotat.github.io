### Overviews
Neutron��ʹ��metadata agent��Ϊ����ṩԪ���ݷ��񣬰����û������룬���̷�����Ϣ�ȡ�Ȼ�����������ֱ����metadata agent����������ͨ��metadata proxy��metadata proxy��metadata agentͨ������Socketͨ�ţ�����metadata proxy������metadata agent��������һ���ڵ��ϡ�

��Neutron�У�metadata proxy������ÿ��vpc�ڣ���vpc�ڵ����ͨ��һ�������ip��ַ��169.254.169.254������proxy��proxy������λ��Ĭ����������������
1. ���е�router��namespace�ж�������proxy��������9697�˿ڡ�ͬʱ��router��iptables���ý�169.254.169.254:80�������ض��򵽱��ص�9697�˿ڡ�
2. û�а����ص�network��namespace�л�����proxy��������80�˿ڡ�
3. �����ˣ�force_metadata=True����dhcp agent������proxy��������80�˿ڡ�
��Ϊ��ϣ��������ص�network��Ϊ�����ע��Ŀ���ַ169.254.169.254����һ��Ϊ����ip��·�ɱ���δ�����غ�force_metadata��networkע���·����һ����dhcp��ַ��

### Router����proxy

��l3 agent��ʼ����ʱ�򣬻��ʼ��һ��metadata_driverʵ����
```python
# neutron/agent/l3/agent.py:L3NATAgent
    def __init__(self, host, conf=None):
        #...
        if self.conf.enable_metadata_proxy:
            self.metadata_driver = metadata_driver.MetadataDriver(self)
```
��ʵ����init�����У�ע���������ص�����
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
���Կ����������������ֱ��Ӧ��router��Դ�Ĵ������ɾ��ǰ��
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