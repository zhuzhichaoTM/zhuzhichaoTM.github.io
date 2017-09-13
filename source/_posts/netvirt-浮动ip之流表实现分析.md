---
title: netvirt-浮动ip之流表实现分析
date: 2017-09-13 13:54:49
tags: [netvirt, opendayligth, openstack, iptables, floatingip]
categories: [openstack, opendaylight, netvirt]
comments: true
---

## 1.基于iptables的浮动ip实现


openstack社区基于iptables规则来实现浮动ip功能,通过在相关路由命名空间中添加iptable规则对进出的包做NAT转换。当外网访问绑定浮动ip的虚机时，目的IP地址为虚机的浮动IP地址，因此必须配置iptables规则将其转化为虚拟机的私有固定IP地址，然后再将它路由到虚机；虚拟机回包的时候，也需要配置iptables规则将虚拟机的私有固定ip地址转化为浮动ip地址，然后路由回去。实现浮动ip功能的iptables规则主要设置在iptables nat表中。

下面是某个路由命名空间中的iptable规则：

	root@ubuntu:~# ip netns exec qrouter-05bf6a27-5901-4c67-bfef-89bde3d2bc67 iptables -t nat -S
	-P PREROUTING ACCEPT
	-P INPUT ACCEPT
	-P OUTPUT ACCEPT
	-P POSTROUTING ACCEPT                 #以上为默认的链规则
	-N neutron-postrouting-bottom
	-N neutron-vpn-agen-OUTPUT
	-N neutron-vpn-agen-POSTROUTING
	-N neutron-vpn-agen-PREROUTING
	-N neutron-vpn-agen-float-snat
	-N neutron-vpn-agen-snat             #以上为neutron增加的chain
	-A PREROUTING -j neutron-vpn-agen-PREROUTING
	-A OUTPUT -j neutron-vpn-agen-OUTPUT
	-A POSTROUTING -j neutron-vpn-agen-POSTROUTING
	-A POSTROUTING -j neutron-postrouting-bottom
	-A neutron-postrouting-bottom -m comment --comment "Perform source NAT on outgoing traffic." -j neutron-vpn-agen-snat
	-A neutron-vpn-agen-OUTPUT -d 192.168.30.34/32 -j DNAT --to-destination 192.168.10.7                #本机访问浮动IP（192.168.30.34）修改为固定IP（192.168.10.7）
	-A neutron-vpn-agen-POSTROUTING -s 192.168.10.0/24 -d 192.168.20.0/24 -m policy --dir out --pol ipsec -j ACCEPT
	-A neutron-vpn-agen-POSTROUTING ! -i qg-084adea8-05 ! -o qg-084adea8-05 -m conntrack ! --ctstate DNAT -j ACCEPT
	-A neutron-vpn-agen-PREROUTING -d 192.168.30.34/32 -j DNAT --to-destination 192.168.10.7           #外部访问浮动IP的traffic的目的IP转换成虚机的固定IP
	-A neutron-vpn-agen-PREROUTING -d 169.254.169.254/32 -i qr-+ -p tcp -m tcp --dport 80 -j REDIRECT --to-ports 9697
	-A neutron-vpn-agen-float-snat -s 192.168.10.7/32 -j SNAT --to-source 192.168.30.34                #绑定浮动IP后，虚拟机出去的包做SNAT，转换为浮动ip    
	-A neutron-vpn-agen-snat -j neutron-vpn-agen-float-snat
	-A neutron-vpn-agen-snat -o qg-084adea8-05 -j SNAT --to-source 192.168.30.29
	-A neutron-vpn-agen-snat -m mark ! --mark 0x2/0xffff -m conntrack --ctstate DNAT -j SNAT --to-source 192.168.30.29
	root@ubuntu:~# 


由上可见：每个浮动ip ，增加如下三个规则

	-A neutron-vpn-agent-PREROUTING -d <floatingip> -j DNAT --to-destination <fixedip> 
	#从别的机器上访问虚机，DST IP 由浮动IP改为固定IP 
    -A neutron-vpn-agent-OUTPUT -d <floatingip> -j DNAT --to <fixedip>                 
	#从本机访问虚机，Dst IP 由浮动IP该为访问固定IP
	-A neutron-vpn-agent-float-snat -s <fixedip> -j SNAT --to <floatingip>
	#虚机访问外网，将Src IP 由固定IP改为浮动IP

这里可以看到当设置了浮动IP以后，SNAT不在使用External Gateway的IP，而是使用浮动IP。虽然entires依然存在，但是因为链neutron-vpn-agent-float-snat 比 neutron-vpn-agent-snat靠前而优先得到执行。



## 2.基于流表实现浮动ip
本文档分析opendaylight netvirt采用纯流表实现浮动ip功能，netvirt三层路由实现采用DVR模式，东西向流量在计算节点通过流表路由转发，南北向流量也直接从计算节点发出。所有流表规则仅仅配置在运行有虚拟机的计算节点上，这样基于纯流表转发可以实现更高的转发性能。其采用流表实现浮动ip的功能.
### 2.1 openstack与odl对接环境搭建
#### 2.1.1.对接环境信息
本次对接版本信息如下

openstack：mitaka controller（1）+copute(2)+ network(2)

opendaylight：boron-sr1

networking-Odl：mitaka

对接文档可以参考如下：

http://www.sdnlab.com/19336.html

http://www.sdnlab.com/18099.html

本文档主要探求odl三层网络功能，3层路由功能不再由L3-agent提供，采用odl netvirt提供三层网络功能。这需要在控制节点上neutron.conf修改service_plugins，同时sdn控制器中安装netvirt的feature。原有openstack环境网络节点上最终只存在dhcp-agent，metadata-agent。

修改为service_plugins = odl-router

控制器中安装的feature

feature:install odl-netvirt-openstack odl-dlux-core


### 2.1.2 外网配置
odl和openstack环境对接成功后，每个节点会自动生成br-int网桥，这里将生成的br-int既作为集成网桥，又作为外部网桥，然后绑定外部网卡用于虚拟机外部网络出口

配置如下：

ovs-vsctl add-port br-int eth2

ovs-vsctl set Open_vSwitch eba4b3eb-c957-402a-b0d0-8931567b18eb other_config:provider_mappings=external:eth2

这里eth2为外网网卡，external为flat映射的物理网络名称。最后eth2作为br-int的端口后，需要将eth2的外网ip配置到br-int网桥上

参考配置脚本如下：

计算节点：

``` bash

rm -rf /etc/openvswitch/conf.db
service openvswitch-switch restart

sdn_controller=192.168.26.6
db_id=`ovs-vsctl show | awk 'NR==1{print}'`
local_ip='192.168.10.22'
external_bridge=br-int
external_nic=eth2
provider_mappings=external:$external_nic

ovs-vsctl set-manager tcp:$sdn_controller:6640
sleep 3
ovs-vsctl set Open_vSwitch $db_id other_config={'local_ip'=$local_ip}
ovs-vsctl add-port $external_bridge $external_nic
ovs-vsctl set Open_vSwitch $db_id other_config:provider_mappings=$provider_mappings

```


网络节点：

原有网络节点不需要添加外部网卡，虚拟机外部流量直接从计算节点出去。
这里网络节点只提供dhcp，metadata服务。

``` bash

rm -rf /etc/openvswitch/conf.db
service openvswitch-switch restart

sdn_controller=192.168.26.6
db_id=`ovs-vsctl show | awk 'NR==1{print}'`
local_ip='192.168.10.23'
external_bridge=br-int
external_nic=eth2
provider_mappings=external:$external_nic

ovs-vsctl set-manager tcp:$sdn_controller:6640
sleep 3
ovs-vsctl set Open_vSwitch $db_id other_config={'local_ip'=$local_ip}
```

### 2.2 组网拓扑图
对接成功后，创建网络，虚拟机，创建路由，路由连接到flat类型的外网，虚拟机绑定floatingip后，可以连接外网。
	
	root@controller:~# neutron net-list
	+--------------------------------------+---------+-------------------------------------------------------+
	| id                                   | name    | subnets                                               |
	+--------------------------------------+---------+-------------------------------------------------------+
	| 28fd1e77-c42d-4d15-9e49-e2514ceed4ff | net-2   | 7b28ca28-6038-468d-b323-8f76aaaa1384 192.168.101.0/24 |
	| c4d3ce39-5d8c-4d1d-9e0c-c0903a8e1637 | ext-net | 2ccf0aec-5dc6-490f-a94b-da4e8ca320b7 192.168.30.0/24  |
	| e4b54ad7-f6fe-4990-b735-36602a534b36 | net-1   | 4c13885a-663e-435b-a1da-ab6e82938ee8 192.168.100.0/24 |
	+--------------------------------------+---------+-------------------------------------------------------+
	root@controller:~#

	root@controller:~# neutron subnet-list
	+--------------------------------------+------------+------------------+------------------------------------------------------+
	| id                                   | name       | cidr             | allocation_pools                                     |
	+--------------------------------------+------------+------------------+------------------------------------------------------+
	| 7b28ca28-6038-468d-b323-8f76aaaa1384 | subnet-2   | 192.168.101.0/24 | {"start": "192.168.101.2", "end": "192.168.101.254"} |
	| 4c13885a-663e-435b-a1da-ab6e82938ee8 | subnet-1   | 192.168.100.0/24 | {"start": "192.168.100.2", "end": "192.168.100.254"} |
	| 2ccf0aec-5dc6-490f-a94b-da4e8ca320b7 | ext-subnet | 192.168.30.0/24  | {"start": "192.168.30.38", "end": "192.168.30.45"}   |
	+--------------------------------------+------------+------------------+------------------------------------------------------+
	root@controller:~# 

	root@controller:~# neutron port-list
	+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------------+
	| id                                   | name | mac_address       | fixed_ips                                                                            |
	+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------------+
	| 0233af76-4218-4d72-aa1a-d94ad1212ced |      | fa:16:3e:e9:9b:28 | {"subnet_id": "4c13885a-663e-435b-a1da-ab6e82938ee8", "ip_address": "192.168.100.3"} |
	| 1c7cb47d-b2e8-4483-b3b8-16dd8eb87b9f |      | fa:16:3e:b0:49:7d | {"subnet_id": "2ccf0aec-5dc6-490f-a94b-da4e8ca320b7", "ip_address": "192.168.30.39"} |
	| 20e08acf-432a-4bcd-891a-544c9d3ab14f |      | fa:16:3e:08:92:58 | {"subnet_id": "7b28ca28-6038-468d-b323-8f76aaaa1384", "ip_address": "192.168.101.5"} |
	| 277ea1d6-7b31-4370-802f-ce3c4db09cf3 |      | fa:16:3e:76:69:01 | {"subnet_id": "4c13885a-663e-435b-a1da-ab6e82938ee8", "ip_address": "192.168.100.6"} |
	| 286d65b5-930c-445c-a957-d58d339bb10f |      | fa:16:3e:e7:2c:21 | {"subnet_id": "7b28ca28-6038-468d-b323-8f76aaaa1384", "ip_address": "192.168.101.1"} |
	| 2a7c6781-b58e-4f59-95b3-d539ec701d55 |      | fa:16:3e:90:4a:e0 | {"subnet_id": "4c13885a-663e-435b-a1da-ab6e82938ee8", "ip_address": "192.168.100.2"} |
	| 436f6d2c-ee92-49bf-8259-660f40cb2978 |      | fa:16:3e:39:b4:06 | {"subnet_id": "2ccf0aec-5dc6-490f-a94b-da4e8ca320b7", "ip_address": "192.168.30.38"} |
	| 7103ab9e-b355-4da9-a2c5-a56e244c1cf1 |      | fa:16:3e:ef:bc:13 | {"subnet_id": "2ccf0aec-5dc6-490f-a94b-da4e8ca320b7", "ip_address": "192.168.30.40"} |
	| 7232931c-314e-42c5-9401-b27058567671 |      | fa:16:3e:cd:7e:e8 | {"subnet_id": "7b28ca28-6038-468d-b323-8f76aaaa1384", "ip_address": "192.168.101.4"} |
	| 7e2e1a47-f43c-4e6c-81ca-1a9b21ebfd4c |      | fa:16:3e:06:e1:36 | {"subnet_id": "4c13885a-663e-435b-a1da-ab6e82938ee8", "ip_address": "192.168.100.5"} |
	| 859897b0-08ab-45d5-b01f-717da6b98c40 |      | fa:16:3e:d2:41:24 | {"subnet_id": "4c13885a-663e-435b-a1da-ab6e82938ee8", "ip_address": "192.168.100.4"} |
	| 8c9fdf55-baca-4491-8eac-07956b2eb10e |      | fa:16:3e:fc:02:94 | {"subnet_id": "7b28ca28-6038-468d-b323-8f76aaaa1384", "ip_address": "192.168.101.3"} |
	| 92b2f1a4-9baa-41d7-a837-6e0bd3bd0c4e |      | fa:16:3e:59:56:48 | {"subnet_id": "7b28ca28-6038-468d-b323-8f76aaaa1384", "ip_address": "192.168.101.2"} |
	| bf4c38ed-89f0-45be-9596-9794451e8362 |      | fa:16:3e:33:77:83 | {"subnet_id": "4c13885a-663e-435b-a1da-ab6e82938ee8", "ip_address": "192.168.100.1"} |
	| c93414f4-31d6-4525-9d44-8cf75b95bc13 |      | fa:16:3e:d4:70:b1 | {"subnet_id": "7b28ca28-6038-468d-b323-8f76aaaa1384", "ip_address": "192.168.101.6"} |
	+--------------------------------------+------+-------------------+--------------------------------------------------------------------------------------+
	root@controller:~# 

	root@controller:~# neutron floatingip-list
	+--------------------------------------+------------------+---------------------+--------------------------------------+
	| id                                   | fixed_ip_address | floating_ip_address | port_id                              |
	+--------------------------------------+------------------+---------------------+--------------------------------------+
	| 678d65a8-d428-4c0b-b45f-18fc11d96e48 | 192.168.100.5    | 192.168.30.40       | 7e2e1a47-f43c-4e6c-81ca-1a9b21ebfd4c |
	| cbb54caa-42d6-48db-b515-35da755e7e41 | 192.168.100.4    | 192.168.30.39       | 859897b0-08ab-45d5-b01f-717da6b98c40 |
	+--------------------------------------+------------------+---------------------+--------------------------------------+
	root@controller:~# 

对应拓扑图信息如下

![图片一](/images/netvirt/WechatIMG2.jpeg)

如上图，tap设备“：”后面为openflow端口号

计算节点1网桥信息：

	compute1：
	
	root@compute1:~# ovs-vsctl show
	f9796b8f-8713-4e32-a54c-c7f6a6f67da6
	    Manager "tcp:192.168.26.6:6640"
	        is_connected: true
	    Bridge br-int
	        Controller "tcp:192.168.26.6:6653"
	            is_connected: true
	        fail_mode: secure
	        Port br-int
	            Interface br-int
	                type: internal
	        Port "tunfa249823f54"
	            Interface "tunfa249823f54"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.22", remote_ip="192.168.10.23"}
	        Port "tap7e2e1a47-f4"
	            Interface "tap7e2e1a47-f4"
	        Port "tun5dd98a99854"
	            Interface "tun5dd98a99854"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.22", remote_ip="192.168.10.25"}
	        Port "tun4a3b08ef6e9"
	            Interface "tun4a3b08ef6e9"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.22", remote_ip="192.168.10.24"}
	        Port "tap7232931c-31"
	            Interface "tap7232931c-31"
	        Port "tap859897b0-08"
	            Interface "tap859897b0-08"
	        Port "eth2"
	            Interface "eth2"
	    ovs_version: "2.5.2"
	root@compute1:~# 

### 2.3 odl netvirt流表管道图
![图片二](/images/netvirt/netvirt_pipeline.jpg)
如上，本文档主要分析浮动ip的流表管道；dvr模式下东西向，南北向流表管道

### 2.4 浮动ip应用场景流表分析

#### 2.4.1.外部主机访问浮动ip

外部主机：192.168.26.5/ 00:50:56:9b:de:b8
 
floatingip：192.168.30.39/fa:16:3e:b0:49:7d

这里192.168.30.39 对应的fixed_ip:192.168.100.4,该虚拟机位于计算节点compute1

inbound

外部访问floatingip的流量从eth2进入，如上图其openflow端口号为1，

	root@compute1:~# ovs-appctl ofproto/trace br-int in_port=1,dl_src=00:0c:29:3b:13:cd,dl_dst=fa:16:3e:b0:49:7d,ip,ip_src=192.168.30.26,ip_dst=192.168.30.39 |grep "Rule\|actions"
	Rule: table=0 cookie=0x8000000 priority=1,in_port=1
	OpenFlow actions=write_metadata:0x390000000001/0xffffff0000000001,goto_table:17
	    Rule: table=17 cookie=0x8000001 priority=5,metadata=0x390000000000/0xffffff0000000000
	    OpenFlow actions=write_metadata:0xc0003900000222fa/0xfffffffffffffffe,goto_table:19
	        Rule: table=19 cookie=0x8000009 priority=20,metadata=0x222fa/0xfffffffe,dl_dst=fa:16:3e:b0:49:7d
	        OpenFlow actions=goto_table:21
	            Rule: table=21 cookie=0x8000003 priority=42,ip,metadata=0x222fa/0xfffffffe,nw_dst=192.168.30.39
	            OpenFlow actions=goto_table:25
	                Rule: table=25 cookie=0x8000004 priority=10,ip,nw_dst=192.168.30.39
	                OpenFlow actions=set_field:192.168.100.4->ip_dst,write_metadata:0x222fe/0xfffffffe,goto_table:27
	                    Rule: table=27 cookie=0x8000004 priority=10,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4
	                    OpenFlow actions=resubmit(,21)
	                        Rule: table=21 cookie=0x8000003 priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4
	                        OpenFlow actions=write_actions(group:150004)
	        Rule: table=220 cookie=0x6900000 priority=6,reg6=0x4100
	        OpenFlow actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffffffffffffe,goto_table:251
	            Rule: table=251 cookie=0x6900000 priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000
	            OpenFlow actions=resubmit(,220)
	                Rule: table=220 cookie=0x8000007 priority=7,reg6=0xe0004100
	                OpenFlow actions=output:2
	Datapath actions: set(eth(src=00:0c:29:3b:13:cd,dst=fa:16:3e:d2:41:24)),3
	root@compute1:~# 

详细流表信息如下：

	cookie=0x8000000, duration=2423.295s, table=0, n_packets=206693, n_bytes=136490247, priority=1,in_port=1 actions=write_metadata:0x390000000001/0xffffff0000000001,goto_table:17
	
	cookie=0x8000001, duration=2376.578s, table=17, n_packets=204203, n_bytes=133019131, priority=5,metadata=0x390000000000/0xffffff0000000000 actions=write_metadata:0xc0003900000222fa/0xfffffffffffffffe,goto_table:19
	
	cookie=0x8000009, duration=2454.466s, table=19, n_packets=2689, n_bytes=226142, priority=20,metadata=0x222fa/0xfffffffe,dl_dst=fa:16:3e:b0:49:7d actions=goto_table:21                             
	#-----------dl_dst为floatingip的mac
	
	cookie=0x8000003, duration=2535.279s, table=21, n_packets=2851, n_bytes=239750, priority=42,ip,metadata=0x222fa/0xfffffffe,nw_dst=192.168.30.39 actions=goto_table:25
	#-----------表21，实现2层转发，3层路由功能，默认ip路由则进入表26，nw_dst为floatingingip,floatingip 路由到表25
	
	cookie=0x8000004, duration=2662.422s, table=25, n_packets=3083, n_bytes=259238, priority=10,ip,nw_dst=192.168.30.39 actions=set_field:192.168.100.4->ip_dst,write_metadata:0x222fe/0xfffffffe,goto_table:27                                     
	#-----------这里进行DNAT转换，更改数据包的源ip为fixed_ip，对应虚拟机ip192.168.100.4，类似于iptable中neutron-vpn-agent-PREROUTING 链中所做的变化
	
	
	cookie=0x8000004, duration=2751.404s, table=27, n_packets=3261, n_bytes=274190, priority=10,ip,metadata=0x222fe0xfffffffe,nw_dst=192.168.100.4 actions=resubmit(,21)                            
	#-----------重新跳转到21，进入正常转发流表
	
	cookie=0x8000003, duration=2872.350s, table=21, n_packets=3402, n_bytes=286048, priority=42,ip,metadata=0x222fe0xfffffffe,nw_dst=192.168.100.4 actions=write_actions(group:150004)
	
	group_id=150004,type=all,bucket=actions=set_field:fa:16:3e:d2:41:24->eth_dst,load:0x4100->NXM_NX_REG6[],resubmit(,220)
	#---------修正数据包的目的mac为fixed_ip的mac，将某个数字写入寄存器，组表里面全部都是修改包的目的Mac地址
	
	cookie=0x6900000, duration=3199.482s, table=220, n_packets=4224, n_bytes=354756, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffffffffffffe,goto_table:251
	#----------进入虚拟机端口的包进行分类
	
	cookie=0x6900000, duration=3292.344s, table=251, n_packets=4318, n_bytes=365236, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)
	#---------------安全组实现，进入虚拟机的acl规则过滤（本文档中安全组实现选择的是stateless模式，不能进行状态追踪
    cookie=0x8000007, duration=3398.701s, table=220, n_packets=4633, n_bytes=388734, priority=7,reg6=0xe0004100 actions=output:2
    #----------------匹配寄存器的值，进入相应的openflow端口
以上外部访问floatingip的流量就被路由到相应的虚拟机中了，下面outbound即为虚拟机回包的过程

outbound

	cookie=0x8000000, duration=3728.375s, table=0, n_packets=2769, n_bytes=266202, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=3789.445s, table=17, n_packets=2834, n_bytes=272460, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40
	#----------------包分类，配置该规则的需进行acl访问控制
	
	cookie=0x6900000, duration=3853.775s, table=40, n_packets=2792, n_bytes=273360, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)
	#--------------安全组实现，从虚拟机发送的流量进行acl规则过滤；通过则返回正常分类流程
	
	cookie=0x8000001, duration=3993.032s, table=17, n_packets=3044, n_bytes=292704, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19
	#-----------------分配到表19，判断是否存在floatingip和网关相关操作
	
	cookie=0x8000009, duration=4075.919s, table=19, n_packets=2917, n_bytes=285866, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:33:77:83 actions=goto_table:21
	#------------进行mac地址的转换，dl_dst为网关(192.168.100.1)mac,这里由于回包的目的ip为192.168.30.29，属于跨网段ip，包需要先发给网关，包的源，目的ip不变，只改变目的mac为网关mac，二层通信中mac必须是同一网段的
	
	cookie=0x8000004, duration=4711.549s, table=21, n_packets=3551, n_bytes=347998, priority=10,ip,metadata=0x222fe/0xfffffffe actions=goto_table:26
	#-----------表21，实现2层转发，3层路由功能，默认ip路由则进入表26，
	
	cookie=0x8000004, duration=4697.160s, table=26, n_packets=3586, n_bytes=351428, priority=10,ip,metadata=0x222fe/0xfffffffe,nw_src=192.168.100.4 actions=set_field:192.168.30.39->ip_src,write_metadata:0x222fa/0xfffffffe,goto_table:28
	#--------做floatingip的SNAT转化，将fixed_ip转化为floatingip。这样需要更改数据包的源ip。表26 执行floatingip的SNAT改变，改变源ip。
	这里类似于虚拟机出去时，Iptable中neutron-vpn-agent-float-snat链功能，需将将Src IP由固定IP改为浮动IP

	cookie=0x8000004, duration=4912.715s, table=28, n_packets=3802, n_bytes=372596, priority=10,ip,metadata=0x222fa/0xfffffffe,nw_src=192.168.30.39 actions=set_field:fa:16:3e:b0:49:7d->eth_src,group:200004      
	#---------- 表28 执行floatingip的snat改变  改变源mac，修改为floatingip的mac	
	group_id=200004,type=all,bucket=actions=set_field:f4:cf:e2:69:34:4f->eth_dst,load:0x3900->NXM_NX_REG6[],resubmit(,220)
	#-------组表修改mac地址，f4:cf:e2:69:34:4f 为外网192.168.30.254 网关mac；这里包达到私有网络网关192.168.100.1后，dst mac为该网关的mac；
	当SNAT完成后，src ip为192.168.30.39，而目的ip为192.168.26.5,两者不在同一个网段，需要将包发送到192.168.30.0/24网段的网关，
	这个过程中目的Ip为外部主机192.168.26.5，目的mac需要变化为网关mac。同上，二层通信，源和目的mac需要必须要是同一网段的，上面已经将源mac变化为192.168.30.39的mac，这里需修改目的mac为网关mac

	
	cookie=0x8000007, duration=5423.025s, table=220, n_packets=368290, n_bytes=217570301, priority=7,reg6=0x3900 actions=output:1
	#-------表220,根据包寄存器值，发送包到相应openflow端口
	通信过程中arp解析通过packet_in，packet_out消息对与控制器交互实现


#### 2.4.2 绑定floatingip的虚拟机间相互访问
(同一个计算节点，都位于compute1)

虚拟机A：192.168.100.4(192.168.30.39)
 
虚拟机B：192.168.100.5(192.168.30.40)

虚拟机A ping 虚拟机B的floatingip 192.168.30.40

outbound ----icmp request

	cookie=0x8000000, duration=496736.592s, table=0, n_packets=9974, n_bytes=1118500, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=496734.338s, table=17, n_packets=9974, n_bytes=1118500, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40
	#-------包分类，从虚拟机发出的包需进行acl控制，进入表40
	
	cookie=0x6900000, duration=503280.813s, table=40, n_packets=9860, n_bytes=1118690, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)
	#------安全组处理 出虚拟机acl规则过滤
	
	cookie=0x8000001, duration=503427.291s, table=17, n_packets=10374, n_bytes=1170500, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19
	
	cookie=0x8000009, duration=605366.539s, table=19, n_packets=114645, n_bytes=11926620, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:33:77:83 actions=goto_table:21
	
	cookie=0x8000004, duration=605670.065s, table=21, n_packets=20421, n_bytes=2708348, priority=10,ip,metadata=0x222fe/0xfffffffe actions=goto_table:26
	
	cookie=0x8000004, duration=605689.730s, table=26, n_packets=19518, n_bytes=2619564, priority=10,ip,metadata=0x222fe/0xfffffffe,nw_src=192.168.100.4 actions=set_field:192.168.30.39->ip_src,write_metadata:0x222fa/0xfffffffe,goto_table:28
	
	cookie=0x8000004, duration=605744.711s, table=28, n_packets=19628, n_bytes=2633864, priority=10,ip,metadata=0x222fa/0xfffffffe,nw_src=192.168.30.39 actions=set_field:fa:16:3e:b0:49:7d->eth_src,group:200004
	
	group_id=200004,type=all,bucket=actions=set_field:f4:cf:e2:69:34:4f->eth_dst,load:0x3900->NXM_NX_REG6[],resubmit(,220)
	
	cookie=0x8000007, duration=606114.190s, table=220, n_packets=30345955, n_bytes=16630300353, priority=7,reg6=0x3900 actions=output:1
	
	cookie=0x8000000, duration=2423.295s, table=0, n_packets=206693, n_bytes=136490247, priority=1,in_port=1 actions=write_metadata:0x390000000001/0xffffff0000000001,goto_table:17
	
	cookie=0x8000001, duration=2376.578s, table=17, n_packets=204203, n_bytes=133019131, priority=5,metadata=0x390000000000/0xffffff0000000000 actions=write_metadata:0xc0003900000222fa/0xfffffffffffffffe,goto_table:19
	
	cookie=0x8000009, duration=606061.322s, table=19, n_packets=1565, n_bytes=151161, priority=20,metadata=0x222fa/0xfffffffe,dl_dst=fa:16:3e:ef:bc:13 actions=goto_table:21
	#-----------dl_dst为floatingip 192.168.30.40的mac
	
	cookie=0x8000003, duration=606154.558s, table=21, n_packets=1658, n_bytes=160275, priority=42,ip,metadata=0x222fa/0xfffffffe,nw_dst=192.168.30.40 actions=goto_table:25
	#----------表21，实现2层转发，3层路由功能，默认ip路由则进入表26，nw_dst为floatingingip,floatingip 路由到表25
	
	cookie=0x8000004, duration=606201.679s, table=25, n_packets=1706, n_bytes=164979, priority=10,ip,nw_dst=192.168.30.40 actions=set_field:192.168.100.5->ip_dst,write_metadata:0x222fe/0xfffffffe,goto_table:27
	#--------这里进行DNAT转换，更改数据包的源ip为fixed_ip，对应虚拟机ip192.168.100.5，类似于iptable中neutron-vpn-agent-PREROUTING 链中所做的变
	
	cookie=0x8000004, duration=606265.218s, table=27, n_packets=1769, n_bytes=171153, priority=10,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.5 actions=resubmit(,21)
	#----------重新跳转到21，进入正常转发流表
	
	cookie=0x8000003, duration=606394.448s, table=21, n_packets=1840, n_bytes=178111, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.5 actions=write_actions(group:150003)
	
	group_id=150003,type=all,bucket=actions=set_field:fa:16:3e:06:e1:36->eth_dst,load:0x4200->NXM_NX_REG6[],resubmit(,220)
	#---------------修正数据包的目的mac为fixed_ip的mac，将某个数字写入寄存器，
	                组表里面全部都是修改包的目的Mac地址
	
	cookie=0x6900000, duration=606456.585s, table=220, n_packets=9390, n_bytes=684606, priority=6,reg6=0x4200 actions=load:0xe0004200->NXM_NX_REG6[],write_metadata:0xe000420000000000/0xfffffffffffffffe,goto_table:251
	#---------------进入虚拟机端口的包进行分类
	
	cookie=0x6900000, duration=93328.042s, table=251, n_packets=5139, n_bytes=502600, priority=61005,ip,metadata=0x420000000000/0x1fffff0000000000 actions=resubmit(,220)
	#----------------安全组实现，进入虚拟机的acl规则过滤（本文档中安全组实现选择的是stateless模式，不能进行状态追踪）
	
	cookie=0x8000007, duration=606553.742s, table=220, n_packets=9490, n_bytes=694238, priority=7,reg6=0xe0004200 actions=output:3
	#----------------匹配寄存器的值，进入相应的openflow端口


inbound---reply

	cookie=0x8000000, duration=515151.152s, table=0, n_packets=2582, n_bytes=250374, priority=4,in_port=3 actions=write_metadata:0x420000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=515190.554s, table=17, n_packets=2626, n_bytes=254574, priority=1,metadata=0x420000000000/0xffffff0000000000 actions=write_metadata:0xa000420000000000/0xfffffffffffffffe,goto_table:40
	#---------设置安全组规则流表
	
	
	cookie=0x6900000, duration=515313.604s, table=40, n_packets=2546, n_bytes=253992, priority=61005,ip,metadata=0x420000000000/0x1fffff0000000000 actions=resubmit(,17)
	#------匹配通过后，跳入表17
	
	cookie=0x8000001, duration=515383.147s, table=17, n_packets=2827, n_bytes=273768, priority=5,metadata=0xa000420000000000/0xffffff0000000000 actions=write_metadata:0xc0004200000222fe/0xffffff
	fffffffffe,goto_table:19
	
	cookie=0x8000009, duration=608887.237s, table=19, n_packets=126179, n_bytes=13258108, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:33:77:83 actions=goto_table:21
	
	cookie=0x8000004, duration=608983.971s, table=21, n_packets=29458, n_bytes=3785658, priority=10,ip,metadata=0x222fe/0xfffffffe actions=goto_table:26
	
	cookie=0x8000004, duration=608953.184s, table=26, n_packets=3934, n_bytes=390174, priority=10,ip,metadata=0x222fe/0xfffffffe,nw_src=192.168.100.5 actions=set_field:192.168.30.40->ip_src,write_metadata:0x
	222fa/0xfffffffe,goto_table:28
	
	cookie=0x8000004, duration=608979.335s, table=28, n_packets=3961, n_bytes=392820, priority=10,ip,metadata=0x222fa/0xfffffffe,nw_src=192.168.30.40 actions=set_field:fa:16:3e:ef:bc:13->eth_src,group:20000	
	
	group_id=200004,type=all,bucket=actions=set_field:f4:cf:e2:69:34:4f->eth_dst,load:0x3900->NXM_NX_REG6[],resubmit(,220)
	
	cookie=0x8000007, duration=609237.877s, table=220, n_packets=30515319, n_bytes=16716214252, priority=7,reg6=0x3900 actions=output:1
	
	cookie=0x8000000, duration=2423.295s, table=0, n_packets=206693, n_bytes=136490247, priority=1,in_port=1 actions=write_metadata:0x390000000001/0xffffff0000000001,goto_table:17
	
	cookie=0x8000001, duration=2376.578s, table=17, n_packets=204203, n_bytes=133019131, priority=5,metadata=0x390000000000/0xffffff0000000000 actions=write_metadata:0xc0003900000222fa/0xfffffff
	ffffffffe,goto_table:19
	
	cookie=0x8000009, duration=2454.466s, table=19, n_packets=2689, n_bytes=226142, priority=20,metadata=0x222fa/0xfffffffe,dl_dst=fa:16:3e:b0:49:7d actions=goto_table:21
	#-----------dl_dst为floatingip的mac
	
	cookie=0x8000003, duration=2535.279s, table=21, n_packets=2851, n_bytes=239750, priority=42,ip,metadata=0x222fa/0xfffffffe,nw_dst=192.168.30.39 actions=goto_table:25                             	#----------表21，实现2层转发，3层路由功能，默认ip路由则进入表26，nw_dst为				 floatingingip,floatingip 路由到表25
	
	cookie=0x8000004, duration=2662.422s, table=25, n_packets=3083, n_bytes=259238, priority=10,ip,nw_dst=192.168.30.39 actions=set_field:192.168.100.4->ip_dst,write_metadata:0x222fe/0xfffffffe,
	goto_table:27
	#---------------这里进行DNAT转换，更改数据包的源ip为fixed_ip，对应虚拟机   ip192.168.100.4，类似于iptable中neutron-vpn-agent-PREROUTING 链中所做的变化
	
	cookie=0x8000004, duration=2751.404s, table=27, n_packets=3261, n_bytes=274190, priority=10,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4 actions=resubmit(,21)                             	#---------------重新跳转到21，进入正常转发流表
	
	cookie=0x8000003, duration=2872.350s, table=21, n_packets=3402, n_bytes=286048, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4 actions=write_actions(group:150004)
	
	group_id=150004,type=all,bucket=actions=set_field:fa:16:3e:d2:41:24->eth_dst,load:0x4100->NXM_NX_REG6[],resubmit(,220)
	#---------------修正数据包的目的mac为fixed_ip的mac，将某个数字写入寄存器，
	                组表里面全部都是修改包的目的Mac地址
	
	cookie=0x6900000, duration=3199.482s, table=220, n_packets=4224, n_bytes=354756, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffffffffffffe,goto_table:251
	#---------------进入虚拟机端口的包进行分类
	
	cookie=0x6900000, duration=3292.344s, table=251, n_packets=4318, n_bytes=365236, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)
	#----------------安全组实现，进入虚拟机的acl规则过滤（本文档中安全组实现选择的是stateless模式，不能进行状态追踪）
	
	cookie=0x8000007, duration=3398.701s, table=220, n_packets=4633, n_bytes=388734, priority=7,reg6=0xe0004100 actions=output:2


