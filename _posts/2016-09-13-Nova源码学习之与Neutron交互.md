---
layout: post
title: Nova源码学习之与Neutron交互
comments: true
tags: [Openstack,Neutron,Nova]
header-img: img/post-bg-os-metro.jpg
---

[TOC]

### Case 1. 定时任务_heal_instance_info_cache

```
graph TB
_heal_instance_info_cache --> get_instance_nw_info
get_instance_nw_info --> _get_instance_nw_info
_get_instance_nw_info --> _build_network_info_model
_build_network_info_model --> list_ports
_build_network_info_model --> _gather_port_ids_and_networks
_build_network_info_model --> _nw_info_get_ips
_build_network_info_model --> _nw_info_get_subnets
_gather_port_ids_and_networks --> _get_available_networks
_get_available_networks --> list_networks
_nw_info_get_ips --> _get_floating_ips_by_fixed_and_port
_get_floating_ips_by_fixed_and_port --> list_floatingips
_nw_info_get_subnets --> _get_subnets_from_port
_get_subnets_from_port --> list_subnets
_get_subnets_from_port --> list_ports
```
```shell
GET /ports.json?tenant_id=13076c4c52c844c6a63dcf3708552c96&device_id=42b7c3f3-867a-4979-9121-0fd0a4815223
GET /networks.json?id=78dca4a5-3650-4f3b-ad6c-82a0e565f21b
GET /floatingips.json?fixed_ip_address=192.168.100.104&port_id=fa71b682-0c33-4d14-8daa-bb7679f3b66c
GET /subnets.json?id=323fa67a-61df-4ef4-ad51-084aa9d2670c
GET /ports.json?network_id=78dca4a5-3650-4f3b-ad6c-82a0e565f21b&device_owner=network%3Adhcp
```

### Case 2. 创建虚机
#### Nova-api
```
graph TB
create --> _create_instance
_create_instance --> _validate_and_build_base_options
_create_instance --> build_instances
_validate_and_build_base_options --> _check_requested_secgroups
_validate_and_build_base_options --> _check_requested_networks
_validate_and_build_base_options --> create_pci_requests_for_sriov_ports
_check_requested_secgroups --> SecurityGroupAPI.get
SecurityGroupAPI.get --> show_security_group
_check_requested_networks --> validate_networks
validate_networks --> _ports_needed_per_instance
validate_networks --> list_ports
validate_networks --> show_quota
_ports_needed_per_instance --> _get_available_networks
_get_available_networks --> list_networks
_ports_needed_per_instance --> _show_port
_show_port --> show_port
create_pci_requests_for_sriov_ports --> _get_port_vnic_info
_get_port_vnic_info --> _show_port
_get_port_vnic_info --> show_network

build_instances --> nova-conductor:build_instances
nova-conductor:build_instances --> build_and_run_instance
build_and_run_instance --> nova-compute:build_and_run_instance
```

```shell
GET /security-groups.json?id=26318222-10cc-49fb-a455-4bb212b8d110
GET /networks.json?tenant_id=78dca4a5-3650-4f3b-ad6c-82a0e565f21b
GET /networks.json?id=78dca4a5-3650-4f3b-ad6c-82a0e565f21b
GET /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
GET /ports.json?tenant_id=13076c4c52c844c6a63dcf3708552c96
GET /quotas/13076c4c52c844c6a63dcf3708552c96.json
GET /networks/78dca4a5-3650-4f3b-ad6c-82a0e565f21b.json
```

***nova-compute:build_and_run_instance 见下***

#### nova compute

```
graph TB
nova-compute:build_and_run_instance --> _do_build_and_run_instance
_do_build_and_run_instance --> _build_and_run_instance
_do_build_and_run_instance --> _cleanup_allocated_networks
_do_build_and_run_instance --> cleanup_instance_network_on_host
_build_and_run_instance --> _build_resources
_build_resources --> _build_networks_for_instance
_build_resources --> _shutdown_instance
_build_networks_for_instance --> setup_instance_network_on_host
_build_networks_for_instance --> get_instance_nw_info
_build_networks_for_instance --> _allocate_network
setup_instance_network_on_host --> _update_port_binding_for_instance
_update_port_binding_for_instance --> _has_port_binding_extension
_update_port_binding_for_instance --> list_ports
_update_port_binding_for_instance --> update_port
_has_port_binding_extension --> _refresh_neutron_extensions_cache
_refresh_neutron_extensions_cache --> list_extensions
_cleanup_allocated_networks --> _deallocate_network
_deallocate_network --> deallocate_for_instance
cleanup_instance_network_on_host --> pass
_allocate_network --> _allocate_network_async
_allocate_network_async --> allocate_for_instance
_shutdown_instance --> _try_deallocate_network
_try_deallocate_network --> _deallocate_network

```

```shell
GET /extensions.json
GET /ports.json?tenant_id=13076c4c52c844c6a63dcf3708552c96&device_id=42b7c3f3-867a-4979-9121-0fd0a4815223
PUT /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
```

***allocate_for_instance 和 deallocate_for_instance 见下***

#### allocate_for_instance

```
graph TB
allocate_for_instance --> _has_port_binding_extension
allocate_for_instance --> _process_requested_networks
allocate_for_instance --> _get_available_networks
allocate_for_instance --> _process_security_groups
allocate_for_instance --> _populate_neutron_extension_values
allocate_for_instance --> update_port
allocate_for_instance --> _create_port
allocate_for_instance --> _unbind_ports
allocate_for_instance --> _delete_ports
_process_requested_networks --> _show_port
_show_port --> show_port
_get_available_networks --> list_networks
_process_security_groups --> list_security_groups
_populate_neutron_extension_values --> _refresh_neutron_extensions_cache
_populate_neutron_extension_values --> _has_port_binding_extension
_has_port_binding_extension --> _refresh_neutron_extensions_cache
_refresh_neutron_extensions_cache --> list_extensions
_create_port --> create_port
_create_port --> delete_port
_unbind_ports --> _has_port_binding_extension
_unbind_ports --> update_port

```
```shell
GET /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
PUT /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
GET /extensions.json
GET /networks.json?tenant_id=78dca4a5-3650-4f3b-ad6c-82a0e565f21b
GET /security-groups.json?id=26318222-10cc-49fb-a455-4bb212b8d110
POST /ports.json
DELETE /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
```

#### deallocate_for_instance

```
graph TB
deallocate_for_instance --> list_ports
deallocate_for_instance --> _unbind_ports
deallocate_for_instance --> _delete_ports
deallocate_for_instance --> update_instance_cache_with_nw_info
_unbind_ports --> _has_port_binding_extension
_unbind_ports --> update_port
_has_port_binding_extension --> _refresh_neutron_extensions_cache
_refresh_neutron_extensions_cache --> list_extensions
_delete_ports --> delete_port
update_instance_cache_with_nw_info --> _get_instance_nw_info
_get_instance_nw_info --> _build_network_info_model
_build_network_info_model --> list_ports
_build_network_info_model --> _gather_port_ids_and_networks
_build_network_info_model --> _nw_info_get_ips
_build_network_info_model --> _nw_info_get_subnets
_gather_port_ids_and_networks --> _get_available_networks
_get_available_networks --> list_networks
_nw_info_get_ips --> _get_floating_ips_by_fixed_and_port
_get_floating_ips_by_fixed_and_port --> list_floatingips
_nw_info_get_subnets --> _get_subnets_from_port
_get_subnets_from_port --> list_subnets
_get_subnets_from_port --> list_ports

```

```shell
GET /extensions.json
PUT /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
DELETE /ports/52492456-0556-4130-abd7-7072ef9abd3b.json
GET /ports.json?tenant_id=13076c4c52c844c6a63dcf3708552c96&device_id=42b7c3f3-867a-4979-9121-0fd0a4815223
GET /networks.json?tenant_id=78dca4a5-3650-4f3b-ad6c-82a0e565f21b
GET /floatingips.json?fixed_ip_address=192.168.100.104&port_id=fa71b682-0c33-4d14-8daa-bb7679f3b66c
GET /subnets.json?id=323fa67a-61df-4ef4-ad51-084aa9d2670c
```

### Case 3. 删除虚机

```
graph TB
delete --> _delete_instance
_delete_instance --> _delete
_delete --> _local_delete
_local_delete --> deallocate_for_instance
```

***deallocate_for_instance 见上***

### Case 4. 未发现直接使用的

#### attach_interface
```
graph TB
attach_interface --> allocate_port_for_instance
allocate_port_for_instance --> allocate_for_instance
```

#### detach_interface
```
graph TB
detach_interface --> deallocate_port_for_instance
deallocate_port_for_instance --> _unbind_ports
deallocate_port_for_instance --> _delete_ports
deallocate_port_for_instance --> get_instance_nw_info
get_instance_nw_info --> _get_instance_nw_info
_get_instance_nw_info --> _build_network_info_model
_build_network_info_model --> list_ports
_build_network_info_model --> _gather_port_ids_and_networks
_build_network_info_model --> _nw_info_get_ips
_build_network_info_model --> _nw_info_get_subnets
_gather_port_ids_and_networks --> _get_available_networks
_get_available_networks --> list_networks
_nw_info_get_ips --> _get_floating_ips_by_fixed_and_port
_get_floating_ips_by_fixed_and_port --> list_floatingips
_nw_info_get_subnets --> _get_subnets_from_port
_get_subnets_from_port --> list_subnets
_get_subnets_from_port --> list_ports

```

#### list_ports

```
graph TB
MetadataRequestHandler.__call__ --> _handle_instance_id_request_from_lb
_handle_instance_id_request_from_lb --> _get_instance_id_from_lb
_get_instance_id_from_lb --> list_ports
```

#### ...