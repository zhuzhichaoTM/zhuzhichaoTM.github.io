---
title: netvirt-DVR路由流表分析
date: 2017-09-13 14:50:49
tags: [netvirt, opendayligth, openstack, dvr]
categories: [openstack, opendaylight, netvirt]
comments: true
---

# Dvr 流表分析
阅读本文前请参考阅读[netvirt-浮动ip之流表实现分析](http://www.lz1999.com/2017/09/13/netvirt-%E6%B5%AE%E5%8A%A8ip%E4%B9%8B%E6%B5%81%E8%A1%A8%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90/)，分析环境都是基于该文档中的物理环境。

本文档流表分析所依据的物理拓扑图和流表管道如下：
![图片一](/images/netvirt/WechatIMG2.jpeg)
如上图，tap设备“：”后面为openflow端口号

![图片二](/images/netvirt/netvirt_pipeline.jpg)
	
## 1.东西向流表分析
### 1.1 同子网，同计算节点虚拟机间互访

虚拟机A: 192.168.100.4 compute1

虚拟机B：192.168.100.5 compute1

192.168.100.4 ping 192.168.100.5

outbound ---request

	cookie=0x8000000, duration=496736.592s, table=0, n_packets=9974, n_bytes=1118500, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=496734.338s, table=17, n_packets=9974, n_bytes=1118500, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40
	
	cookie=0x6900000, duration=503280.813s, table=40, n_packets=9860, n_bytes=1118690, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)
	
	cookie=0x8000001, duration=503427.291s, table=17, n_packets=10374, n_bytes=1170500, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19
	
	cookie=0x1080000, duration=515251.235s, table=19, n_packets=26179005, n_bytes=14185066330, priority=0 actions=resubmit(,17)
	
	cookie=0x8040000, duration=514616.493s, table=17, n_packets=2063, n_bytes=184176, priority=6,metadata=0xc000410000000000/0xffffff0000000000 actions=write_metadata:0xe000411388000000/0xfffffffffffffffe,goto_table:50
	
	cookie=0x8051388, duration=514712.959s, table=50, n_packets=2164, n_bytes=193738, priority=20,metadata=0x411388000000/0x1fffffffff000000,dl_src=fa:16:3e:d2:41:24 actions=goto_table:51
	
	cookie=0x8031388, duration=514965.945s, table=51, n_packets=2046, n_bytes=193452, priority=20,metadata=0x1388000000/0xffff000000,dl_dst=fa:16:3e:06:e1:36 actions=load:0x4200->NXM_NX_REG6[],resubmit(,220)
	
	cookie=0x6900000, duration=513923.972s, table=220, n_packets=1703, n_bytes=149276, priority=6,reg6=0x4200 actions=load:0xe0004200->NXM_NX_REG6[],write_metadata:0xe000420000000000/0xfffffffffffffffe,goto_table:251
	
	 cookie=0x6900000, duration=668.645s, table=251, n_packets=1236, n_bytes=120198, priority=61005,ip,metadata=0x420000000000/0x1fffff0000000000 actions=resubmit(,220)	
	
	cookie=0x8000007, duration=513815.884s, table=220, n_packets=1589, n_bytes=138496, priority=7,reg6=0xe0004200 actions=output:3

inbound---reply

	cookie=0x8000000, duration=515151.152s, table=0, n_packets=2582, n_bytes=250374, priority=4,in_port=3 actions=write_metadata:0x420000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=515190.554s, table=17, n_packets=2626, n_bytes=254574, priority=1,metadata=0x420000000000/0xffffff0000000000 actions=write_metadata:0xa000420000000000/0xfffffffffffffffe,goto_table:40
	#---------设置安全组规则流表

	cookie=0x6900000, duration=515313.604s, table=40, n_packets=2546, n_bytes=253992, priority=61005,ip,metadata=0x420000000000/0x1fffff0000000000 actions=resubmit(,17)
	#---------匹配通过后，跳入表17
	
	cookie=0x8000001, duration=515383.147s, table=17, n_packets=2827, n_bytes=273768, priority=5,metadata=0xa000420000000000/0xffffff0000000000 actions=write_metadata:0xc0004200000222fe/0xfffffffffffffffe,goto_table:19
	
	cookie=0x1080000, duration=516101.855s, table=19, n_packets=26233690, n_bytes=14213658127, priority=0 actions=resubmit(,17)
	
	cookie=0x8040000, duration=515612.100s, table=17, n_packets=2898, n_bytes=275324, priority=6,metadata=0xc000420000000000/0xffffff0000000000 actions=write_metadata:0xe000421388000000/0xfffffffffffffffe,goto_table:50

	cookie=0x8031388, duration=515683.933s, table=51, n_packets=2804, n_bytes=265496, priority=20,metadata=0x1388000000/0xffff000000,dl_dst=fa:16:3e:d2:41:24 actions=load:0x4100->NXM_NX_REG6[],resubmit(,220)

	cookie=0x6900000, duration=515718.509s, table=220, n_packets=42123, n_bytes=3444466, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffffffffffffe,goto_table:251

	cookie=0x6900000, duration=2833.379s, table=251, n_packets=41918, n_bytes=3447056, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)
	
	cookie=0x8000007, duration=516041.828s, table=220, n_packets=42468, n_bytes=3477044, priority=7,reg6=0xe0004100 actions=output:2
	


### 1.2 同子网，不同计算节点虚拟机间互访

虚拟机A: 192.168.100.4 compute1

虚拟机B：192.168.100.6 compute2

192.168.10.4 (compute1) ping 192.168.10.6 (compute2)  (不同计算节点)

outbound --request

	compute1:
	
	cookie=0x8000000, duration=496736.592s, table=0, n_packets=9974, n_bytes=1118500, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17

	cookie=0x6900000, duration=496734.338s, table=17, n_packets=9974, n_bytes=1118500, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40

	cookie=0x6900000, duration=503280.813s, table=40, n_packets=9860, n_bytes=1118690, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)

	cookie=0x8000001, duration=503427.291s, table=17, n_packets=10374, n_bytes=1170500, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19

	cookie=0x1080000, duration=515251.235s, table=19, n_packets=26179005, n_bytes=14185066330, priority=0 actions=resubmit(,17)

	cookie=0x8040000, duration=514616.493s, table=17, n_packets=2063, n_bytes=184176, priority=6,metadata=0xc000410000000000/0xffffff0000000000 actions=write_metadata:0xe000411388000000/0xfffffffffffffffe,goto_table:50

	cookie=0x8051388, duration=514712.959s, table=50, n_packets=2164, n_bytes=193738, priority=20,metadata=0x411388000000/0x1fffffffff000000,dl_src=fa:16:3e:d2:41:24 actions=goto_table:51
	#----------源mac是已知的就进行处理，跳入表51，
	不知道就需要向控制器发送packet_in消息请求流表
	
	cookie=0x8031388, duration=1926.951s, table=51, n_packets=663, n_bytes=62958, priority=20,metadata=0x1388000000/0xffff000000,dl_dst=fa:16:3e:76:69:01 actions=set_field:0x37->tun_id,output:6
	#-------根据mac地址，将数据包引入隧道口，最终路由到相应计算节点。
	隧道为vxlan隧道，这里设置了tun_id，对应于网络id
	
	这里tun4a3b08ef6e9的openflow端口号为6，tun4a3b08ef6e9建立了本地隧道ip192.168.10.22(copute1)到远端隧道ip192.168.10.24（compute2）之间的隧道
	
	compute2：
	
	从compute1发送的request包经过隧道进入compute2的tun843fe0e497d口，openflow端口号为5

	cookie=0x8000001, duration=517283.573s, table=0, n_packets=696080, n_bytes=50627237, priority=5,in_port=5 actions=write_metadata:0x450000000001/0x1fffff0000000001,goto_table:36

	cookie=0x9000037, duration=517454.470s, table=36, n_packets=1510, n_bytes=146767, priority=5,tun_id=0x37 actions=load:0x3700->NXM_NX_REG6[],resubmit(,220)
	
	cookie=0x6900000, duration=517496.024s, table=220, n_packets=1853, n_bytes=165085, priority=6,reg6=0x3700 actions=load:0xe0003700->NXM_NX_REG6[],write_metadata:0xe000370000000000/0xfffffffffffffffe,goto_table:251

	cookie=0x6900000, duration=4369.608s, table=251, n_packets=1494, n_bytes=147599, priority=61005,ip,metadata=0x370000000000/0x1fffff0000000000 actions=resubmit(,220)
	#---------设置安全组规则 进入虚拟机方向

	cookie=0x8000007, duration=517629.120s, table=220, n_packets=1994, n_bytes=178455, priority=7,reg6=0xe0003700 actions=output:2

	 2(tap277ea1d6-7b) 即为 192.168.100.6 的tap设备


 inbound--reply

	 compute2:

	cookie=0x8000000, duration=517856.535s, table=0, n_packets=1971, n_bytes=188958, priority=4,in_port=2 actions=write_metadata:0x370000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=517898.901s, table=17, n_packets=2016, n_bytes=193256, priority=1,metadata=0x370000000000/0xffffff0000000000 actions=write_metadata:0xa000370000000000/0xfffffffffffffffe,goto_table:40             
	
	cookie=0x6900000, duration=517937.452s, table=40, n_packets=1910, n_bytes=186934, priority=61005,ip,metadata=0x370000000000/0x1fffff0000000000 actions=resubmit(,17）
	#----------表40 设置安全组规则的流表
	
	cookie=0x8000001, duration=518052.654s, table=17, n_packets=2181, n_bytes=208866, priority=5,metadata=0xa000370000000000/0xffffff0000000000 actions=write_metadata:0xc0003700000222fe/0xfffffffffffffffe,goto_table:19

	cookie=0x1080000, duration=518471.445s, table=19, n_packets=26430474, n_bytes=14300659600, priority=0 actions=resubmit(,17)
	
	cookie=0x8040000, duration=518091.122s, table=17, n_packets=2220, n_bytes=212632, priority=6,metadata=0xc000370000000000/0xffffff0000000000 actions=write_metadata:0xe000371388000000/0xfffffffffffffffe,goto_table:50

	cookie=0x8051388, duration=518183.825s, table=50, n_packets=2318, n_bytes=221900, priority=20,metadata=0x371388000000/0x1fffffffff000000,dl_src=fa:16:3e:76:69:01 actions=goto_table:51
	#-------判断是否为已知的mac，其他的向控制器发送消息获取

	cookie=0x8031388, duration=3428.435s, table=51, n_packets=2252, n_bytes=213864, priority=20,metadata=0x1388000000/0xffff000000,dl_dst=fa:16:3e:d2:41:24 actions=set_field:0x41->tun_id,output:5                     	#-----在数据包中添加tun_id，然后从5口发出

	 5(tun843fe0e497d）隧道端口，其本端隧道ip192.168.10.24（compute2），远端隧道ip为192.168.10.22(compute1)
	
	compute1:
	对端compute1 中6(tun4a3b08ef6e9)接受compute2发送的vxlan报文
	
	cookie=0x8000001, duration=518621.082s, table=0, n_packets=698909, n_bytes=50665838, priority=5,in_port=6 actions=write_metadata:0x440000000001/0x1fffff0000000001,goto_table:36
	
	cookie=0x9000041, duration=518642.831s, table=36, n_packets=2765, n_bytes=266049, priority=5,tun_id=0x41 actions=load:0x4100->NXM_NX_REG6[],resubmit(,220)
	
	cookie=0x6900000, duration=518666.508s, table=220, n_packets=45241, n_bytes=3740582, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffffffffffffe,goto_table:251

	cookie=0x6900000, duration=5560.074s, table=251, n_packets=44640, n_bytes=3713812, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)

	cookie=0x8000007, duration=518769.020s, table=220, n_packets=45348, n_bytes=3750788, priority=7,reg6=0xe0004100 actions=output:2

	最终进入虚拟机192.168.100.4

### 1.3 不同子网，同计算节点虚拟机间互访

虚拟机A：192.168.100.4  compute1

虚拟机B：192.168.101.4  compute1

192.168.100.4 ping 192.168.101.4

outbound --request

	cookie=0x8000000, duration=496736.592s, table=0, n_packets=9974, n_bytes=1118500, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=496734.338s, table=17, n_packets=9974, n_bytes=1118500, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40
	
	cookie=0x6900000, duration=503280.813s, table=40, n_packets=9860, n_bytes=1118690, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)
	
	cookie=0x8000001, duration=503427.291s, table=17, n_packets=10374, n_bytes=1170500, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19
	
	cookie=0x8000009, duration=520097.535s, table=19, n_packets=27174, n_bytes=3302480, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:33:77:83 actions=goto_table:21
	#------表19 L3网关，floatingip相关mac处理。fa:16:3e:33:77:83为192.168.100.1默认网关的mac。
	由于是跨网段，其目的ip为192.168.101.4，但是目的mac还是为192.168.100.1默认网关的mac

	cookie=0x8000003, duration=7046.973s, table=21, n_packets=498, n_bytes=48804, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.101.4 actions=write_actions(group:150011)
	
	group_id=150011,type=all,bucket=actions=set_field:fa:16:3e:cd:7e:e8->eth_dst,load:0x4800->NXM_NX_REG6[],resubmit(,220)
	#------修改目的mac（网关mac）为真实的目的mac

	cookie=0x6900000, duration=7323.597s, table=220, n_packets=922, n_bytes=88952, priority=6,reg6=0x4800 actions=load:0xe0004800->NXM_NX_REG6[],write_metadata:0xe000480000000000/0xfffffffffffffffe,goto_table:251
	#---------安全组，入方向过滤
	
	cookie=0x6900000, duration=7378.653s, table=251, n_packets=907, n_bytes=90066, priority=61005,ip,metadata=0x480000000000/0x1fffff0000000000 actions=resubmit(,220)
	
	cookie=0x8000007, duration=7424.997s, table=220, n_packets=1026, n_bytes=98976, priority=7,reg6=0xe0004800 actions=output:7


inbound-reply

	cookie=0x8000000, duration=7588.975s, table=0, n_packets=1193, n_bytes=114914, priority=4,in_port=7 actions=write_metadata:0x480000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=7627.736s, table=17, n_packets=1234, n_bytes=118876, priority=1,metadata=0x480000000000/0xffffff0000000000 actions=write_metadata:0xa000480000000000/0xfffffffffffffffe,goto_table:40
	
	cookie=0x6900000, duration=7652.860s, table=40, n_packets=1206, n_bytes=118236, priority=61005,ip,metadata=0x480000000000/0x1fffff0000000000 actions=resubmit(,17)
	#----------处理安全组相关，出虚拟机方向过滤
	
	cookie=0x8000001, duration=7724.590s, table=17, n_packets=1335, n_bytes=128550, priority=5,metadata=0xa000480000000000/0xffffff0000000000 actions=write_metadata:0xc0004800000222fe/0xfffffffffffffffe,goto_table:19

	cookie=0x8000009, duration=7828.804s, table=19, n_packets=1241, n_bytes=121618, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:e7:2c:21 actions=goto_table:21
	#---------fa:16:3e:e7:2c:21 为192.168.101.1网关mac地址

	cookie=0x8000003, duration=521049.125s, table=21, n_packets=40184, n_bytes=3275909, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4 actions=write_actions(group:150004)
	
	group_id=150004,type=all,bucket=actions=set_field:fa:16:3e:d2:41:24->eth_dst,load:0x4100->NXM_NX_REG6[],resubmit(,220)

	cookie=0x6900000, duration=521119.648s, table=220, n_packets=47772, n_bytes=3983356, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffff
	ffffffffe,goto_table:251 

	cookie=0x6900000, duration=7981.008s, table=251, n_packets=47045, n_bytes=3949502, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)
	#-----------处理安全组相关 入虚拟机方向过滤

	cookie=0x8000007, duration=521198.634s, table=220, n_packets=47853, n_bytes=3991182, priority=7,reg6=0xe0004100 actions=output:2
	

### 1.4 不同子网，不同计算节点虚拟机间互访

虚拟机A：192.168.100.4  compute1

虚拟机B：192.168.101.6  compute2

192.168.100.4 ping 192.168.101.6

outbound --request

	compute1：
	
	cookie=0x8000000, duration=496736.592s, table=0, n_packets=9974, n_bytes=1118500, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17

	cookie=0x6900000, duration=496734.338s, table=17, n_packets=9974, n_bytes=1118500, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40

	cookie=0x6900000, duration=503280.813s, table=40, n_packets=9860, n_bytes=1118690, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)

	cookie=0x8000001, duration=503427.291s, table=17, n_packets=10374, n_bytes=1170500, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19

	cookie=0x8000009, duration=520097.535s, table=19, n_packets=27174, n_bytes=3302480, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:33:77:83 actions=goto_table:21
	#------表19 L3网关，floatingip处理。fa:16:3e:33:77:83为192.168.100.1默认网关的mac。
	由于是跨网段，其目的ip为192.168.101.6，但是目的mac还是为192.168.100.1默认网关的mac：fa:16:3e:33:77:83

	cookie=0x8000003, duration=6703.524s, table=21, n_packets=541, n_bytes=53018, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.101.6 actions=write_actions(set_field:0x1118f->tun_id,output:6)

	compute1节点上的隧道口信息
	6(tun4a3b08ef6e9): addr:b2:85:a8:5e:1d:b5
	     config:     0
	     state:      0
	     speed: 0 Mbps now, 0 Mbp

	Port "tun4a3b08ef6e9"
	            Interface "tun4a3b08ef6e9"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.22", remote_ip="192.168.10.24"}	
	192.168.10.24为compute2的本地隧道Ip     
	从上面看到数据包从compute1的隧道端口tun4a3b08ef6e9发出，到compute2相应隧道口

	compute2节点上的隧道口信息
	Port "tun843fe0e497d"
	            Interface "tun843fe0e497d"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.24", remote_ip="192.168.10.22"}
	                
	5(tun843fe0e497d): addr:de:9d:07:58:63:68
	     config:     0
	     state:      0
	     speed: 0 Mbps now, 0 Mbps max
	
	compute2:

	cookie=0x8000001, duration=521797.419s, table=0, n_packets=705304, n_bytes=51363296, priority=5,in_port=5 actions=write_metadata:0x450000000001/0x1fffff0000000001,goto_table:36

	cookie=0x901118f, duration=8681.714s, table=36, n_packets=865, n_bytes=84770, priority=5,tun_id=0x1118f actions=write_actions(group:150013)
	
	group_id=150013,type=all,bucket=actions=set_field:fa:16:3e:d4:70:b1->eth_dst,load:0x4a00->NXM_NX_REG6[],resubmit(,220)
	#-----针对跨网段情况，修改目的mac地址
	
	cookie=0x6900000, duration=8845.413s, table=220, n_packets=1244, n_bytes=116700, priority=6,reg6=0x4a00 actions=load:0xe0004a00->NXM_NX_REG6[],write_metadata:0xe0004a0000000000/0xfffffffffffffffe,goto_table:251

	cookie=0x6900000, duration=8881.479s, table=251, n_packets=1140, n_bytes=112900, priority=61005,ip,metadata=0x4a0000000000/0x1fffff0000000000 actions=resubmit(,220)

	cookie=0x8000007, duration=8935.783s, table=220, n_packets=1337, n_bytes=125702, priority=7,reg6=0xe0004a00 actions=output:7

	7(tapc93414f4-31): addr:fe:16:3e:d4:70:b1
	     config:     0
	     state:      0
	     current:    10MB-FD COPPER
	     speed: 10 Mbps now, 0 Mbps max
	tapc93414f4-31对应于虚拟机192.168.101.6


inbound---reply

	compute2:
	
	cookie=0x8000000, duration=9410.115s, table=0, n_packets=1763, n_bytes=168566, priority=4,in_port=7 actions=write_metadata:0x4a0000000000/0xffffff0000000001,goto_table:17

	cookie=0x6900000, duration=9441.356s, table=17, n_packets=1797, n_bytes=171842, priority=1,metadata=0x4a0000000000/0xffffff0000000000 actions=write_metadata:0xa0004a0000000000/0xfffffffffffffffe,goto_table:40

	cookie=0x6900000, duration=9480.426s, table=40, n_packets=1748, n_bytes=171048, priority=61005,ip,metadata=0x4a0000000000/0x1fffff0000000000 actions=resubmit(,17)

	cookie=0x8000001, duration=9521.398s, table=17, n_packets=1878, n_bytes=179724, priority=5,metadata=0xa0004a0000000000/0xffffff0000000000 actions=write_metadata:0xc0004a00000222fe/0xfffffffffffffffe,goto_table:19

	cookie=0x8000009, duration=9611.179s, table=19, n_packets=1774, n_bytes=173852, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:e7:2c:21 actions=goto_table:21
	#-------------fa:16:3e:e7:2c:21 为192.168.101.1网关的mac地址

	cookie=0x8000003, duration=8001.326s, table=21, n_packets=1854, n_bytes=181692, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4 actions=write_actions(set_field:0x11186->tun_id,output:5）	

	compute2隧道口信息
	5(tun843fe0e497d): addr:de:9d:07:58:63:68
	    config:     0
	    state:      0
	    speed: 0 Mbps now, 0 Mbps max
	Port "tun843fe0e497d"
	            Interface "tun843fe0e497d"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.24", remote_ip="192.168.10.22"}
	
	compute1隧道口信息

	Port "tun4a3b08ef6e9"
	            Interface "tun4a3b08ef6e9"
	                type: vxlan
	                options: {key=flow, local_ip="192.168.10.22", remote_ip="192.168.10.24"}
	
	6(tun4a3b08ef6e9): addr:b2:85:a8:5e:1d:b5
	     config:     0
	     state:      0
	     speed: 0 Mbps now, 0 Mbps max

	compue1:

	cookie=0x8000001, duration=523029.274s, table=0, n_packets=707739, n_bytes=51372368, priority=5,in_port=6 actions=write_metadata:0x440000000001/0x1fffff0000000001,goto_table:36
	
	cookie=0x9011186, duration=523051.365s, table=36, n_packets=2073, n_bytes=203154, priority=5,tun_id=0x11186 actions=write_actions(group:150004)
	
	 group_id=150004,type=all,bucket=actions=set_field:fa:16:3e:d2:41:24->eth_dst,load:0x4100->NXM_NX_REG6[],resubmit(,220)
	
	cookie=0x6900000, duration=523112.961s, table=220, n_packets=49851, n_bytes=4181890, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffff
	ffffffffe,goto_table:251
	
	cookie=0x6900000, duration=9977.597s, table=251, n_packets=49033, n_bytes=4144326, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)
	
	cookie=0x8000007, duration=523183.920s, table=220, n_packets=49923, n_bytes=4188834, priority=7,reg6=0xe0004100 actions=output:2

## 2.南北向流表分析
### 2.1 虚拟机访问外网网关

虚拟机 192.168.100.4

外网网关为 10.89.1.254

流表规则同外部网络访问floatingip，inbound和outbound调换即可

参考阅读[netvirt-浮动ip之流表实现分析](http://www.lz1999.com/2017/09/13/netvirt-%E6%B5%AE%E5%8A%A8ip%E4%B9%8B%E6%B5%81%E8%A1%A8%E5%AE%9E%E7%8E%B0%E5%88%86%E6%9E%90/)

### 2.2 绑定floatingip的虚拟机访问内部私网网关
虚拟机A: 192.168.100.4 floatingip:192.168.30.39

子网网关: 192.168.100.1

outbound----request

	cookie=0x8000000, duration=496736.592s, table=0, n_packets=9974, n_bytes=1118500, priority=4,in_port=2 actions=write_metadata:0x410000000000/0xffffff0000000001,goto_table:17
	
	cookie=0x6900000, duration=496734.338s, table=17, n_packets=9974, n_bytes=1118500, priority=1,metadata=0x410000000000/0xffffff0000000000 actions=write_metadata:0xa000410000000000/0xfffffffffffffffe,goto_table:40
	
	cookie=0x6900000, duration=503280.813s, table=40, n_packets=9860, n_bytes=1118690, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,17)
	
	cookie=0x8000001, duration=503427.291s, table=17, n_packets=10374, n_bytes=1170500, priority=5,metadata=0xa000410000000000/0xffffff0000000000 actions=write_metadata:0xc0004100000222fe/0xfffffffffffffffe,goto_table:19
	
	cookie=0x8000009, duration=503475.503s, table=19, n_packets=10311, n_bytes=1180354, priority=20,metadata=0x222fe/0xfffffffe,dl_dst=fa:16:3e:33:77:83 actions=goto_table:21
	#---------处理l3网关，floatingip的表

	cookie=0x8000003, duration=503681.784s, table=21, n_packets=461, n_bytes=45178, priority=10,icmp,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.1,icmp_type=8,icmp_code=0 actions=move:NXM_OF_
	ETH_SRC[]->NXM_OF_ETH_DST[],set_field:fa:16:3e:33:77:83->eth_src,move:NXM_OF_IP_SRC[]->NXM_OF_IP_DST[],set_field:192.168.100.1->ip_src,set_field:0->icmp_type,load:0->NXM_OF_IN_PORT[],resubmit(,21)
	#-------根据匹配到的request包自学习构建reply包的流表规则，源mac设置为目的mac，
	网关mac设置为源mac；源ip设置为目的ip，目的网关ip设置为源ip.set_field:0->icmp_type
	即设定icmp包为reply

inbound--reply

	cookie=0x8000003, duration=511668.356s, table=21, n_packets=37816, n_bytes=3043845, priority=42,ip,metadata=0x222fe/0xfffffffe,nw_dst=192.168.100.4 actions=write_actions(group:150004)
	#------------匹配reply包，目的地址为192.168.100.4，进入组表

	group_id=150004,type=all,bucket=actions=set_field:fa:16:3e:d2:41:24->eth_dst,load:0x4100->NXM_NX_REG6[],resubmit(,220)
	#---------------设置目的mac
	
	 cookie=0x6900000, duration=512056.590s, table=220, n_packets=38583, n_bytes=3108914, priority=6,reg6=0x4100 actions=load:0xe0004100->NXM_NX_REG6[],write_metadata:0xe000410000000000/0xfffffffffffffffe,goto_table:251

	cookie=0x6900000, duration=512094.034s, table=251, n_packets=38326, n_bytes=3095040, priority=61005,ip,metadata=0x410000000000/0x1fffff0000000000 actions=resubmit(,220)

	cookie=0x8000007, duration=512142.152s, table=220, n_packets=38671, n_bytes=3117370, priority=7,reg6=0xe0004100 actions=output:2                       ![]()