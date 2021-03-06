# OpenFlow网络中组播机制的研究



## 系统设计思想

### 组播组成员管理

传统网络中，主机与路由器之间通过运行 **IGMP** 协议来交互，在SDN中， **IGMP** 报文在控制器上实现。

**IGMP** **V3 ** 有两种报文，分别是查询报文与报告报文。控制器接收到查询报文之后，不进行处理，直接转发给 **OF** 交换机；报告报文由主机发出，用于声明加入某个组播组或响应 **IP** 组播路由器的查询。

在组播控制器中，通过解析报告报文的每个 **Group** **Record** 的 **Record** **Type** 字段来确定当前网络中的组播组成员情况。

### 组播转发树

在 **OpenFlow** 网络中生成组播转发树是以边界 **OF** 交换机作为根节点，组播控制器在生成组播转发树之前要获得全局信息，包含整个网络拓扑和各个链路状态等。

* 获取网络拓扑

SDN中使用 **LLDP** 协议作为链路发现协议，该协议提供标准的链路发现方法，可将设备能力、设备标识、接口标识、管理地址等信息组成不同的 **TLV** (Type/Length/Value)，并将这些信息封装在 **LLDPDU** (链路层发现协议数据单元)中，发送给与自己直连的邻居节点，邻居以标准的 **MIB** (管理信息库)的形式将接收到的信息保存，以供网管系统判断查询链路状况。

![LLDP](https://github.com/BokalaQuan/RyuNote/blob/master/LLDP.png)

> 1. 控制器向所有与之相连的 **OF** 交换机 **Packet_Out** 封装  **LLDP** 数据包的消息；
> 2. 该消息命令 **OF** 交换机将此数据包转发到所有端口；
> 3. 交换机 **Packet_In** 数据包给控制器；
> 4. 控制器接收到 **Packet_In** 消息后，分析该数据包并将 **OF** 交换机的连接信息保存到链路发现表中。

* 获取链路状态信息

```c
/* Body of reply to OFPST_PORT request. If a counter is unsupported, 
* set the field to all ones. */
struct ofp_port_stats {
  uint16_t port_no;
  uint8_t pad[6];	/* Align to 64-bits. */
  uint64_t rx_packets;
  uint64_t tx_packets;
  uint64_t rx_bytes;
  uint64_t tx_bytes;
  uint64_t rx_dropped;
  uint64_t tx_dropped;
  uint64_t rx_errors;
  uint64_t tx_errors;
  uint64_t rx_frame_err;
  uint64_t rx_over_err;
  uint64_t rx_crc_err;
  uint64_t collisions;
};
```

其中 *tx_bytes* 字段表示当前该端口发送了多少字节数，可通过定时获取该字段来间接计算链路带宽利用率。

网络拓扑作为图的节点，各个链路利用率作为边的权值，控制器可以利用 **Dijkstra** 算法计算从根节点到目的节点的最优转发路径。

* 有新的组成员加入时：若加入新的组播地址，即当前 **OpenFlow** 网络中还没有该组播，那以直连路由器的 **OF** 交换机作为目的节点，生成最优转发路径；若该组播在网络中已经存在，那以直连路由器的 **OF** 交换机作为根节点，组播转发树中所有的 **OF** 交换机作为目的节点，利用 **Dijkstra** 算法计算得到离根节点最近的目的节点的路径，然后将该路径发现加入组播转发树，形成新枝。
* 有组成员离开时：若该组成员是该组播组的最后一个成员，则直接删除所有的转发路径；若不是，则对转发树进行枝剪，从叶子结点向根节点进行反向删除，一直删除到有分叉的节点。

### 组播服务质量



## 系统设计实现

### 总体设计

![sd](https://github.com/BokalaQuan/RyuNote/blob/master/%E7%B3%BB%E7%BB%9F%E8%AE%BE%E8%AE%A1.png)

### 拓扑发现模块

| 接口名称                      | 调用方法                               | 功能                        |
| :------------------------ | ---------------------------------- | ------------------------- |
| get all the switches      | GET /v1.0/topology/switches        | 获取OpenFlow网络中所有OF交换机的信息   |
| get the switch            | GET /v1.0/topology/switches/<dpid> | 获取编号为dpid的OF交换机信息         |
| get all the links         | GET /v1.0/topology/links           | 获取OpenFlow网络中所有OF交换机的连接信息 |
| get the links of a switch | GET /v1.0/topology/links/<dpid>    | 获取编号为dpid的OF交换机的连接信息      |



**Ryu**启动时加上==--observe-links== 参数，表明 **Ryu** 监听链路发现事件。组件 **rest_topology** 通过调用 topology 的 get_link 接口实现了获取拓扑的功能，所以控制器可以直接调用 **rest_topology** 的 **API** 来获取 **OpenFlow** 网络的拓扑。



### 链路状态获取模块

```python
self.bd_matrix	# 每个端口的总带宽
self.tx_matrix	# 每个端口的tx_bypes
self.dij_matrix	# 各个链路利用率
self.port_loop	# 线程用来定时time遍历读取每个OF交换机的每个端口的统计数据
```

链路利用率的计算公式
$$
speed=(tx.bytes - tx.matrix[m][n])/time
$$

$$
dij.matrix[m][n]=speed/bd.matrix[m][n]
$$

```python
def port_loop(self):
    while True:
        # 遍历获取所有的OF交换机的端口统计信息
        for dpid in dpids:
            ports = get_ports_stats(dpid)
            # 遍历编号为dpid的OF交换机的所有端口的统计信息
            for port in ports[dpid]:
                # 判断当前port是否在links中
                # 如果在，则读取该port的tx_bytes字段的数据进行并行计算
                if port_no in links:
                    speed = (tx_bytes - tx_matrix[m][n]) / time
                    link_utilization = speed / bandwidth
                    dij_matrix[m][n] = link_utilization
                    tx_matrix[m][n] = tx_bytes
		# 睡眠time时间
        sleep(time)
```



### 组播报文解析与转发模块

![组播报文解析与转发](https://github.com/BokalaQuan/RyuNote/blob/master/%E7%BB%84%E6%92%AD%E6%8A%A5%E6%96%87%E8%A7%A3%E6%9E%90%E4%B8%8E%E8%BD%AC%E5%8F%91.png)



1. 控制器接收到OF交换机的Packet_In消息后，首先判断其以太网协议类型，如果是IPV4协议（0x0800），则继续解析，若不是，则丢弃该报文；
2. 判断IPV4报文首部的协议类型，若是IGMP协议，则继续解析，若不是，则丢弃该报文；
3. 判断IGMP报文首部的协议类型，若是Membership Query(0x11)，则是查询报文，则不做处理，直接将此报文Packet_Out给与主机相连接的OF交换机上；若是Membership Report(0x22)，则是报告报文，继续解析；
4. IGMP V3报告报文中有 M 个 **Group** **Records** ，逐一进行解析。判断每个Record中**Record** **Type** 字段，若是Change To Exclude Mode，则是**Join**，若是  Change To Include Mode，则是 **Leave**。将报文Packet_Out给与每个组播路由器直连的OF交换机上。
5. 确定是 **Join** 或 **Leave** 后，则通知组播组成员管理模块进行处理。处理完成之后，再通知组播转发树生成模块继续处理。



### 组播组成员管理模块



|    组播地址    |  主机IP地址   |
| :--------: | :-------: |
| group_ip_0 | ip_00 ... |
| group_ip_1 | ip_10 ... |



![组播成员管理](https://github.com/BokalaQuan/RyuNote/blob/master/%E7%BB%84%E6%92%AD%E6%88%90%E5%91%98%E7%AE%A1%E7%90%86.png)



### 组播转发树生成模块

![Multicast Tree Created](https://github.com/BokalaQuan/RyuNote/blob/master/%E7%BB%84%E6%92%AD%E7%94%9F%E6%88%90%E6%A0%91%E8%AE%A1%E7%AE%97.png)



* 加入组播

1. 加入新组播

> 若是加入新组播，那么该模块就以直连路由器的OF交换机作为根节点，直连主机的OF交换机作为目的节点，运行Dijkstra算法来计算生成最优转发路径，格式如下：
>
> ***dij_path = [{dpid1 : port1}, {dpid2 : port2}, ...]*** 
>
> 该转发路径最后指向Packet_In报告报文的OF交换机，最后，该交换机将组播流转发给主机，最终的转发路径 *path* 为：
>
> ***path = dij_path + {in_dpid : in_port}*** 
>
> ***in_dpid***  是 Packet_In报文的OF交换机编号，***in_port***  是OF交换机接收到报文的端口。

2. 加入已经存在的组播

> 如果是已经存在的组播，那么组播转发树已经存在。以直连主机的OF交换机作为根节点，组播转发树中所有的交换机作为目的节点，利用Dijkstra算法计算得到离根节点最近的目的节点 ***dipd_t*** 的路径，然后得到路径 ***leaf_dij_path***，格式如下：
>
>  ***leaf_dij_path = [{dpid_t : port_t}, ...]*** 
>
> 搜索组播转发树得到边界OF交换机到编号 dpid_t 的OF交换机的路径 root_path。最后得到完整的path：
>
> ***path = root_path + leaf_dij_path + {in_dpid : in_port}*** 
>
> 组播转发树的结构：
>
> ***opt = {dpid1 : [port1], dpid2 : [port21, port22], ...}***
>
> 所有组播的转发树保存在 opts 中， 结构如下：
>
> ***opts = {'group_ip0': opt0, 'group_ip1':opt1, ...}***

* 离开组播



### 组播转发树更新模块

![组播树更新](https://github.com/BokalaQuan/RyuNote/blob/master/%E7%BB%84%E6%92%AD%E6%A0%91%E6%9B%B4%E6%96%B0.png)



### 流表下发模块

| 接口名称                             | 调用方法                                     | 功能   |
| -------------------------------- | :--------------------------------------- | ---- |
| get all the switches             | get_all_switches()                       |      |
| get all the links                | get_all_links()                          |      |
| get the switch                   | get_switch(dpid)                         |      |
| get all flows stats              | get_all_flows_stats()                    |      |
| get ports stats                  | get_ports_stats(dpid)                    |      |
| get ports description            | get_ports_description(dpid)              |      |
| add a flow entry                 | add_flow_entry(dpid, priority, match, actions) |      |
| modify all matching flow entries | modify_all_matching_flow_entries(dpid, priority, match, actions) |      |
| modify flow entry strictly       | modify_flow_entry_strictly(dpid, priority, match, actions) |      |
| delete all matching flow entries | delete_all_matching_flow_entries(dpid, priority, match, actions) |      |
| delete flow entry strictly       | delete_flow_entry_strictly(dpid, priority, match, actions) |      |
| delete all flow entries          | delete_all_flow_entries(dpid)            |      |
| get queue status                 | get_queue_status(dpid)                   |      |
| get queue configuration          | get_queue_configuration(dpid)            |      |
| set queue                        | set_queue(dpid, port_name, q_type, max_rate, queues) |      |
| delete queue                     | delete_queue(dpid)                       |      |
| get all of QoS rules             | get_qos_rules(dpid)                      |      |
| set a QoS rule                   | set_qos_rule(dpid, priority, match, actions) |      |
| delete QoS rules                 | delete_qos_rules(dpid, qos_id)           |      |

组件 **rest_topology** 通过调用 topology 的 get_link 接口实现了获取拓扑的功能，组件 **rest_qos** 提供查询、配置 Queue 和QoS的**API**。

![流标下发](https://github.com/BokalaQuan/RyuNote/blob/master/%E6%B5%81%E8%A1%A8%E4%B8%8B%E5%8F%91.png)





