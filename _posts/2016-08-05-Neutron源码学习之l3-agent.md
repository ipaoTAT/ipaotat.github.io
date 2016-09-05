---
layout: post
title: Neutron学习之l3-agent
comments: true
tags: [OpenStack,Neutron]
header-img: img/post-bg-os-metro.jpg
---

### Overview
l3-agent为用户提供三层路由服务。包括vrouter管理、floatingip管理、网关管理、firewall管理等。vrouter使用linux route完成路由转发；使用iptables完成floatingip、网关地址转换和firewall规则。
从结构上，l3-agent与neutron-server通过RPC交互，从controller上数据库中拉取配置，并使用上述技术在网络节点上管理三层服务资源；同时neutron-server及时通知（notify）l3-agent主动进行资源配置调整。

<!-- more -->

### 1. agent启动
和openstack其他使用基于MQ的rpc的服务相同，重点在于传入的manager类。真正的rpc调用将由该类中的同名函数提供。
以l3-agent为例：
```python
# neutron/agent/l3_agent.py
def main(manager='neutron.agent.l3.agent.L3NATAgentWithStateReport'):
    register_opts(cfg.CONF)
    common_config.init(sys.argv[1:])
    config.setup_logging()
    server = neutron_service.Service.create(
        binary='neutron-l3-agent',
        topic=topics.L3_AGENT,
        report_interval=cfg.CONF.AGENT.report_interval,
        manager=manager)
    service.launch(cfg.CONF, server).wait()
```
传入的manager类为neutron.agent.l3.agent:L3NATAgentWithStateReport，该类继承了L3NATAgent，L3NATAgent类实现了由neutron-server触发的操作。neutron-server对l3-agent的操作在neutron.api.rpc.agentnotifiers.l3_rpc_agent_api:L3AgentNotifyAPI类中定义。如下：
RPC|描述
---|---
agent_updated|
router_deleted|delete时
routers_updated|更新路由时触发（包括gateway）*
add_arp_entry|DVR使用*
del_arp_entry|DVR使用*
router_removed_from_agent|l3-agent-router-remove和rescheduler router时
router_added_to_agent|l3-agent-router-add和rescheduler router时

l3-agent启动之初，会启动一个循环loop，等待处理queue中的路由。
```python
# neutron/agent/l3/agent.py
    def after_start(self):
        eventlet.spawn_n(self._process_routers_loop)
        #...
```
```python
# neutron/agent/l3/agent.py
    def _process_routers_loop(self):
        LOG.debug("Starting _process_routers_loop")
        pool = eventlet.GreenPool(size=8)
        while True:
            pool.spawn_n(self._process_router_update)
```
_process_router_update函数从queue中读出数据并处理的。紧接着，l3-agent会从neutron-server获取路由配置，并将需要更新的路由（本地与数据库中的差值）放入queue。
```
graph TB
after_start --> periodic_sync_routers_task
periodic_sync_routers_task --> fetch_and_sync_all_routers
fetch_and_sync_all_routers --> get_routers
fetch_and_sync_all_routers --> queue.add
```
**Note：** 如何触发_process_router_update执行需要进一步确认。
#### agent启动时的ns管理
l3 agent 启动之后，会使用periodic_sync_routers_task函数从neutron server拉一次本agent上的全部router的数据，并对本地的namespace进行管理。该函数的代码如下：
```python
# neutron/agent/l3/agent.py
    def periodic_sync_routers_task(self, context):
        self.process_services_sync(context)
        if not self.fullsync:
            return
        try:
            with self.namespaces_manager as ns_manager:
                self.fetch_and_sync_all_routers(context, ns_manager)
        except n_exc.AbortSyncRouters:
            self.fullsync = True
```
其中“with self.namespaces_manager as ns_manager”会调用namespaces_manager的__enter__函数，该函数查询本机上所有的qrouter namespace，并将其保存在namespaces_manager对象的_all_namespaces中。
```python
# neutron/agent/l3/namespace_manager.py
    def __enter__(self):
        self._all_namespaces = set()
        self._ids_to_keep = set()
        if self._clean_stale:
            self._all_namespaces = self.list_all()
        return self
```
接着fetch_and_sync_all_routers会从neutron server抓取所有属于该agent的router信息，并将他们加入update queue中（queue中router的更新下文将详细描述）。同时很重啊的一点是，该函数还会把所有router记录在namespaces_manager的_ids_to_keep中，表示该namespace需要保留。
```python
# neutron/agent/l3/agent.py
    def fetch_and_sync_all_routers(self, context, ns_manager):
        #...
        for r in routers:
            ns_manager.keep_router(r['id'])
            #...
            update = queue.RouterUpdate(r['id'],
                                        queue.PRIORITY_SYNC_ROUTERS_TASK,
                                        router=r,
                                        timestamp=timestamp)
            self._queue.add(update)
        #....
```
```python
# neutron/agent/l3/namespace_manager.py
    def keep_router(self, router_id):
        self._ids_to_keep.add(router_id)
```
在periodic_sync_routers_task函数中语句“with self.namespaces_manager as ns_manager”执行块之后，会自动调用self.namespaces_manager的__exit__函数（python的with关键字），在该函数中namespaces_manager会将_all_namespaces中不在_ids_to_keep的namespace删除。
```python
# neutron/agent/l3/namespace_manager.py
    def __exit__(self, exc_type, value, traceback):
        if not self._clean_stale
            #不需要删除
            return True
        即使需要删除，也就这一次（agent刚启动时）
        self._clean_stale = False

        for ns in self._all_namespaces:
            _ns_prefix, ns_id = self.get_prefix_and_id(ns)
            if ns_id in self._ids_to_keep:
                continue
            self._cleanup(_ns_prefix, ns_id)
        return True
```
未知namespace的删除流程与正常router的删除流程不同。正常的路由删除会按照数据库中的配置删除ip、port、route之后再删除ns；而为止ns的删除比较简单，只是将该ns中的port删除即可进行ns的删除动作。
```python
# neutron/agent/l3/namespace_manager.py
    def _cleanup(self, ns_prefix, ns_id):
        ns_class = self.ns_prefix_to_class_map[ns_prefix]
        ns = ns_class(ns_id, self.agent_conf, self.driver, use_ipv6=False)
        try:
            if self.metadata_driver:
                # 删除metadata proxy进程（如果有的话）
                self.metadata_driver.destroy_monitored_metadata_proxy(
                    self.process_monitor, ns_id, self.agent_conf)
            # 删除namespace
            ns.delete()
        except RuntimeError:
            LOG.exception(_LE('Failed to destroy stale namespace %s'), ns)
```
```python
# neutron/agent/l3/namespaces.py:RouterNamespace
    def delete(self):
        ns_ip = ip_lib.IPWrapper(namespace=self.name)
        for d in ns_ip.get_devices(exclude_loopback=True):
            if d.name.startswith(INTERNAL_DEV_PREFIX):
                # device is on default bridge
                self.driver.unplug(d.name, namespace=self.name,
                                   prefix=INTERNAL_DEV_PREFIX)
            elif d.name.startswith(ROUTER_2_FIP_DEV_PREFIX):
                ns_ip.del_veth(d.name)
            elif d.name.startswith(EXTERNAL_DEV_PREFIX):
                self.driver.unplug(
                    d.name,
                    bridge=self.agent_conf.external_network_bridge,
                    namespace=self.name,
                    prefix=EXTERNAL_DEV_PREFIX)

        super(RouterNamespace, self).delete()
```
可以看到，这里的delete函数只是删除devices，包括qg设备、qr设备以及veth设备，然后就调用超类的delete函数进行ip netns delete动作。所以清楚一个qrouter的ns残留的时候，只需要将device删除，就可以删除ns了。
#### _process_router_update函数
_process_router_update函数是l3-agent具体操作的入口函数。
```python
# neutron/agent/l3/agent.py
    def _process_router_update(self):
        # 从queue中取出每个update任务
        for rp, update in self._queue.each_update_to_next_router():
            # rp: RouterProcesser
            # update.id：更新的router id
            # update.router: 具体router信息
            # update.action： 操作
            # update.priority： 操作优先级
            # IPv6前缀更新[?]
            if update.action == queue.PD_UPDATE:
                self.pd.process_prefix_update()
                continue
            router = update.router
            # 只有在删除时update.router才会为None，如果其他操作下未传入router，需要通过rpc请求neutron server获取router信息。
            if update.action != queue.DELETE_ROUTER and not router:
                try:
                    update.timestamp = timeutils.utcnow()
                    routers = self.plugin_rpc.get_routers(self.context,
                                                          [update.id])
                except Exception:
                    msg = _LE("Failed to fetch router information for '%s'")
                    LOG.exception(msg, update.id)
                    self.fullsync = True
                    continue

                if routers:
                    router = routers[0]
            
            # action为删除路由，或者根据id找不到router信息（数据库中已经删除）时，进行删除路由操作。
            if not router:
                removed = self._safe_router_removed(update.id)
                if not removed:
                    # 当执行periodic_sync_routers_task的时候重试该操作[并没有执行?]
                    self.fullsync = True
                else:
                    # 更新updatetime域
                    rp.fetched_and_processed(update.timestamp)
                continue

            try:
                # 更新路由，包括在创建操作以及更新操作
                self._process_router_if_compatible(router)
            except n_exc.RouterNotCompatibleWithAgent as e:
                # 更新失败（RouterNotCompatibleWithAgent ？），删除路由
                    self._safe_router_removed(router['id'])
            except Exception:
                # 更新失败，伺机重试
                self.fullsync = True
                continue

            LOG.debug("Finished a router update for %s", update.id)
            rp.fetched_and_processed(update.timestamp)  
```
可见该函数主要做的操作是从queue读取任务，并将其分为删除路由和更新路由操作，并交由_safe_router_removed和_process_router_if_compatible函数分别处理。这两个函数在下文介绍。
### 2. router_deleted
#### 调用触发
1. delete_router <-- DELETE /routers/{router_id}
```python
# neutron/db/l3_db.py
    def delete_router(self, context, id):
        super(L3_NAT_db_mixin, self).delete_router(context, id)
        self.notify_router_deleted(context, id)
#...
    def notify_router_deleted(self, context, router_id):
        self.l3_rpc_notifier.router_deleted(context, router_id)
```
以广播方式通知所有l3-agent。
```python
# neutron/api/rpc/agentnotifiers/l3_rpc_agent_api.py
    def router_deleted(self, context, router_id):
        self._notification_fanout(context, 'router_deleted', router_id)
```
#### 处理流程
l3-agent的RPC服务器接收到上述通知后，会调用L3NATAgentWithStateReport类的router_deleted函数。该函数在queue中放入一个update事件，action为DELETE_ROUTER，参数为要删除的router_id。（无需router详细信息）
```python
# neutron/agent/l3/agent.py
    def router_deleted(self, context, router_id):
        """Deal with router deletion RPC message."""
        LOG.debug('Got router deleted notification for %s', router_id)
        update = queue.RouterUpdate(router_id,
                                    queue.PRIORITY_RPC,
                                    action=queue.DELETE_ROUTER)
        self._queue.add(update)
```
该操作最终由_router_removed具体执行。
```
graph TB
_process_router_update --> _safe_router_removed
_safe_router_removed --> _router_removed
_router_removed --> namespaces_manager.ensure_router_cleanup
_router_removed --> router_info.delete
namespaces_manager.ensure_router_cleanup --> _cleanup
_cleanup --> ns.delete/删除namespace
router_info.delete --> process/清空router的所有配置
```

### 3.routers_updated
#### 调用触发
1. create_floatingip <-- POST /floatingips
2. delete_floatingip <-- DELETE /floatingips/{floatingip_id}
3. update_router <-- PUT /routers/{router_id}
4. udpate_floatingip <-- PUT /floatingips/{floatingip_id}
5. update_bw_bind
6. subscribe* <-- 启动时

其中external gateway的设置调用的neutron api是PUT /routers/{router_id}，从而触发了update_router，进而触发l3-agent调用routers_updated。
#### 处理流程
l3-agent收到上述请求后，会调用L3NATAgentWithStateReport类的routers_updated函数,该函数将路由列表中的每个路由id封装为一个update对象，action为None（缺省操作为更新），最后放入queue中。
```python
# neutron/agent/l3/agent.py
   def routers_updated(self, context, routers):
        """Deal with routers modification and creation RPC message
        if routers:
            # This is needed for backward compatibility
            if isinstance(routers[0], dict):
                routers = [router['id'] for router in routers]
            for id in routers:
                update = queue.RouterUpdate(id, queue.PRIORITY_RPC)
                self._queue.add(update)
```
最终该操作由_process_router_if_compatible函数执行。该函数主要进行路由ns的创建或配置更新。
```python
# neutron/agent/l3/agent.py
    def _process_router_if_compatible(self, router):
        #...
        if router['id'] not in self.router_info:
            self._process_added_router(router)
        else:
            self._process_updated_router(router)
```
l3-agent维护的路由资源由namespace来体现，由router_info来记录。因此路由的创建和更新都对应到router_info的创建和更新，以及namespace的创建。_process_updated_router主要调用了路由的router_info对象的process方法，该方法会根据router_info对象的router属性（router属性的记录）在router的namespace中进行iptables和route table的操作;_process_added_router函数做的工作，就是先创建一个router_info对象（同时创建namespace），将其加入到routers列表中，然后再执行router_info对象的process方法。
```python
# neutron/agent/l3/router_info.py
    def process(self, agent):
        self._process_internal_ports(agent.pd)
        agent.pd.sync_router(self.router['id'])
        self.process_external(agent)
        # 更新静态路由配置
        self.routes_updated()

        # 更新external gateway port
        self.ex_gw_port = self.get_ex_gw_port()
        # 更新enable_snat，如果为False，则不会配置iptables snat规则
        self.enable_snat = self.router.get('enable_snat')
```
#### _process_internal_ports
函数_process_internal_ports进行了路由器上的ports更新。由current_ports和existing_ports对比得到新增的、删除的和更新的ports，分别进行处理。
```python
# neutron/agent/l3/router_info.py
    def _process_internal_ports(self, pd):
        existing_port_ids = set(p['id'] for p in self.internal_ports)

        internal_ports = self.router.get(l3_constants.INTERFACE_KEY, [])
        current_port_ids = set(p['id'] for p in internal_ports
                               if p['admin_state_up'])

        new_port_ids = current_port_ids - existing_port_ids
        # 得到新建的，删除的，以及更新的ports
        new_ports = [p for p in internal_ports if p['id'] in new_port_ids]
        old_ports = [p for p in self.internal_ports
                     if p['id'] not in current_port_ids]
        updated_ports = self._get_updated_ports(self.internal_ports,
                                                internal_ports)

        enable_ra = False
        # 新建ports
        for p in new_ports:
            self.internal_network_added(p)
            self.internal_ports.append(p)
            enable_ra = enable_ra or self._port_has_ipv6_subnet(p)
            for subnet in p['subnets']:
                if ipv6_utils.is_ipv6_pd_enabled(subnet):
                    interface_name = self.get_internal_device_name(p['id'])
                    pd.enable_subnet(self.router_id, subnet['id'],
                                     subnet['cidr'],
                                     interface_name, p['mac_address'])
        # 删除ports
        for p in old_ports:
            self.internal_network_removed(p)
            self.internal_ports.remove(p)
            enable_ra = enable_ra or self._port_has_ipv6_subnet(p)
            for subnet in p['subnets']:
                if ipv6_utils.is_ipv6_pd_enabled(subnet):
                    pd.disable_subnet(self.router_id, subnet['id'])
        # 更新ports
        updated_cidrs = []
        if updated_ports:
            for index, p in enumerate(internal_ports):
                if not updated_ports.get(p['id']):
                    continue
                self.internal_ports[index] = updated_ports[p['id']]
                interface_name = self.get_internal_device_name(p['id'])
                ip_cidrs = common_utils.fixed_ip_cidrs(p['fixed_ips'])
                LOG.debug("updating internal network for port %s", p)
                updated_cidrs += ip_cidrs
                self.internal_network_updated(interface_name, ip_cidrs)
                enable_ra = enable_ra or self._port_has_ipv6_subnet(p)

        # Check if there is any pd prefix update
        for p in internal_ports:
            if p['id'] in (set(current_port_ids) & set(existing_port_ids)):
                for subnet in p.get('subnets', []):
                    if ipv6_utils.is_ipv6_pd_enabled(subnet):
                        old_prefix = pd.update_subnet(self.router_id,
                                                      subnet['id'],
                                                      subnet['cidr'])
                        if old_prefix:
                            self._internal_network_updated(p, subnet['id'],
                                                           subnet['cidr'],
                                                           old_prefix,
                                                           updated_cidrs)
                            enable_ra = True

        # Enable RA[?]
        if enable_ra:
            self.enable_radvd(internal_ports)
        # 删掉不需要的qr-设备（理论不需要，因为之前删除port已经把设备移除了。）
        existing_devices = self._get_existing_devices()
        # 从ns中的/sys/class/net中查找虚拟接口设备
        current_internal_devs = set(n for n in existing_devices
                                    if n.startswith(INTERNAL_DEV_PREFIX))
        # 获得当前虚拟接口设备名字列表（qr-XXX）
        current_port_devs = set(self.get_internal_device_name(port_id)
                                for port_id in current_port_ids)
        stale_devs = current_internal_devs - current_port_devs
        # 拔掉不需要的设备
        for stale_dev in stale_devs:
            LOG.debug('Deleting stale internal router device: %s',
                      stale_dev)
            pd.remove_stale_ri_ifname(self.router_id, stle_dev)
            self.driver.unplug(stale_dev,
                               namespace=self.ns_name,
                               prefix=INTERNAL_DEV_PREFIX)

```
其中最主要的三个函数internal_network_added, internal_network_removed, internal_network_updated。

internal_network_added函数调用了_internal_network_added函数，该函数完成虚拟设备qr-xxx的就插入（plug），并通过init_router_port 调用init_l3完成虚拟设备cidr的配置。
```
graph TB
internal_network_added --> _internal_network_added
_internal_network_added --> driver.plug/调用具体驱动的plug_new进行插入
_internal_network_added --> init_router_port
init_router_port --> init_l3
```
init-l3经过一系列调用，最终执行shell命令: 
```shell
ip netns exec qrouter-xxx ip addr add {cidrs} scope {scopy} dev {qr-xxx}
```
完成地址添加。

internal_network_removed函数直接将虚拟设备qr-xxx拔出（unplug），不需要删除cidr配置。

internal_network_updated函数直接调用init_l3对虚拟设备qr-xxx进行cidr的配置。
```python
# neutron/agent/l3/router_info.py
    def internal_network_updated(self, interface_name, ip_cidrs):
        self.driver.init_l3(interface_name, ip_cidrs=ip_cidrs,
                            namespace=self.ns_name)
```
#### process_external
process_external负责配置router的external gateway接口配置和floating ip配置，主要包括ip配置和iptables规则配置两个动作。
```python
    def process_external(self, agent):
        fip_statuses = {}
        existing_floating_ips = self.floating_ips
        try:
            with self.iptables_manager.defer_apply():
                ex_gw_port = self.get_ex_gw_port()
                self._process_external_gateway(ex_gw_port, agent.pd)
                if not ex_gw_port:
                    return

                # Process SNAT/DNAT rules and addresses for floating IPs
                self.process_snat_dnat_for_fip()

            # Once NAT rules for floating IPs are safely in place
            # configure their addresses on the external gateway port
            interface_name = self.get_external_device_interface_name(
                ex_gw_port)
            fip_statuses = self.configure_fip_addresses(interface_name)

        except (n_exc.FloatingIpSetupException,
                n_exc.IpTablesApplyException) as e:
                # All floating IPs must be put in error state
                LOG.exception(e)
                fip_statuses = self.put_fips_in_error_state()
        finally:
            agent.update_fip_statuses(
                self, existing_floating_ips, fip_statuses)
```
##### _process_external_gateway函数用于配置external网关。
```python
# neutron/agent/l3/router_info.py
    def _process_external_gateway(self, ex_gw_port, pd):
        # TODO(Carl) Refactor to clarify roles of ex_gw_port vs self.ex_gw_port
        ex_gw_port_id = (ex_gw_port and ex_gw_port['id'] or
                         self.ex_gw_port and self.ex_gw_port['id'])

        interface_name = None
        if ex_gw_port_id:
            interface_name = self.get_external_device_name(ex_gw_port_id)
        if ex_gw_port:
            if not self.ex_gw_port:
                self.external_gateway_added(ex_gw_port, interface_name)
                pd.add_gw_interface(self.router['id'], interface_name)
            elif not self._gateway_ports_equal(ex_gw_port, self.ex_gw_port):
                self.external_gateway_updated(ex_gw_port, interface_name)
        elif not ex_gw_port and self.ex_gw_port:
            self.external_gateway_removed(self.ex_gw_port, interface_name)
            pd.remove_gw_interface(self.router['id'])
        # 删掉不需要的qr-设备（理论不需要，保险起见）
        existing_devices = self._get_existing_devices()
        stale_devs = [dev for dev in existing_devices
                      if dev.startswith(EXTERNAL_DEV_PREFIX)
                      and dev != interface_name]
        for stale_dev in stale_devs:
            LOG.debug('Deleting stale external router device: %s', stale_dev)
            pd.remove_gw_interface(self.router['id'])
            self.driver.unplug(stale_dev,
                               bridge=self.agent_conf.external_network_bridge,
                               namespace=self.ns_name,
                               prefix=EXTERNAL_DEV_PREFIX)

        # Process SNAT rules for external gateway
        # iptables设置
        gw_port = self._router.get('gw_port')
        self._handle_router_snat_rules(gw_port, interface_name)
```
_process_external_gateway主要有三个函数：external_gateway_added、external_gateway_removed和external_gateway_updated。_process_external_gateway流程与_process_internal_ports类似，可互相参考。不同的是，_process_external_gateway在最后通过_handle_router_snat_rules设置iptables规则，而internal-port不需要。

external_gateway_added函数调用_external_gateway_added函数，该函数完成外部网关的插入、cidr配置等。
```
graph TB
external_gateway_added --> _external_gateway_added
_external_gateway_added --> _plug_external_gateway
_external_gateway_added --> driver.init_router_port
_external_gateway_added --> send_ip_addr_adv_notif/通知ip地址变更
_plug_external_gateway -->driver.plug/插入qg端口
driver.init_router_port --> init_l3/配置cidr
```

external_gateway_removed函数首先移除qg设备上的ip，之后拔掉设备。
```python
# neutron/agent/l3/router_info.py
    def external_gateway_removed(self, ex_gw_port, interface_name):
        LOG.debug("External gateway removed: port(%s), interface(%s)",
                  ex_gw_port, interface_name)
        device = ip_lib.IPDevice(interface_name, namespace=self.ns_name)
        for ip_addr in ex_gw_port['fixed_ips']:
            self.remove_external_gateway_ip(device,
                                            common_utils.ip_to_cidr(
                                                ip_addr['ip_address'],
                                                ip_addr['prefixlen']))
        self.driver.unplug(interface_name,
                           bridge=self.agent_conf.external_network_bridge,
                           namespace=self.ns_name,
                           prefix=EXTERNAL_DEV_PREFIX)
```

external_gateway_updated只是简单的调用_external_gateway_added函数对网关ip进行更新。

***configure_fip_addresses用于配置floating ip, 通过调用process_floating_ip_addresses来实现。该函数通过add_floating_ip和remove_floating_ip两个函数实现floating ip的管理，其中add_floating_ip函数被不同的Router类重写。***

***update_fip_statuses函数用于向neutron server报告floating ip的状态。当前floating ip的状态并不能准确指示其可用性。***

#### routes_updated
routes_updated函数用于更新静态路由配置。该函数通过一系列调用之后，最终使用 ip route命令进行操作。当然，该操作也在router的namespace下。

### 4.router_removed_from_agent
#### 触发调用
1. remove_router_from_l3_agent <-- DELETE /agents/{agent_id}/l3-routers/{router_id}
2. reschedule_router 
#### 调用流程
router_removed_from_agent函数的处理流程与router_deleted相同，在queue中添加一个删除路由的任务。

### 5.router_added_to_agent
#### 触发调用
1. add_router_to_l3_agent <-- POST /agents/{agent_id}/l3-routers
2. reschedule_router
#### 调用流程
router_added_to_agent函数的处理流程与router_updated相同。

