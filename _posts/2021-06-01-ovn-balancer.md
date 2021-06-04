---
layout: article
tags: OVN
title: OVN实验四：OVN负载均衡
mathjax: true
key: Linux
---

[参考](https://blog.csdn.net/zhengmx100/category_9268063.html)
{:.info}

## topo
```


              outside network
                 |
                 |
                 |
+----------------+------------------------------+
|              eth1                            |
|                                              |
|    sw outside                                |
|                                              |
|              outside-edge1                   |
+----------------------+-----------------------+
                       |
                       |
                       |
+----------------------------------------------+
|              edge1-outside                   |
|           02:0a:7f:00:01:29 10.127.0.129     |
|                                              |
|    Gateway router edge1                      |
|                                              |
|          02:ac:10:ff:00:01 172.16.255.1      |
|              edge1-transit                   |
+----------------------+-----------------------+
                       |
                       |
                       |
+----------------------+-----------------------+
|              transit-edge1                   |
|                                              |
|    sw transit                                |
|    172.16.255.0/30                           |
|                                              |
|              transit-tenant1                 |
+----------------------+-----------------------+
                       |
                       |
                       |
+----------------------+------------------------------------------------------+
|    router tenant1                                                           |
|                                                                             |
|                                                                             |
| 02:ac:10:ff:01:29                                       02:ac:10:ff:01:93   |
| 172.16.255.129                                          172.16.255.193      |
| tenant1-dmz                                             tenant1-inside      |
+----+----------------------------------------------------------+-------------+
     |                                                          |
     |                                                          |
     |                                                          |
+----+------------------------------+         +-----------------+-------------+
| dmz-tenant1                       |         |           inside-tenant1      |
|                                   |         |                               |
|             sw dmz                |         |           sw inside           |
|        172.16.255.128/26          |         |        172.16.255.192/26      |
|                                   |         |                               |
|  dmz-vm1                 dmz-vm2  |         | inside-vm1       inside-vm2   |
+-----+-----------------------+-----+         +-------+--------------+--------+
      |                       |                       |              |
      |                       |                       |              |
      |                       |                       |              |
+-----+------------+  +-------+-----------+  +--------+--------+   +-+----------------+
| 02:ac:10:ff:01:30|  | 02:ac:10:ff:01:31 |  |02:ac:10:ff:01:94|   | 02:ac:10:ff:01:95|
| 172.16.255.130   |  | 172.16.255.131    |  | 172.16.255.194  |   | 172.16.255.195   |
|      vm1         |  |      vm2          |  |     vm3         |   |     vm4          |
+-------------------  +-------------------+  +-----------------+   +------------------+
```

在实验三环境的基础上做下面的额外配置。
{:.info}

## 负载均衡应用于逻辑路由器
```
(1)负载平衡只能应用于“集中式”路由器（即网关路由器）。
(2)第1个注意事项已经决定了路由器上的负载平衡是非分布式服务。

# vm1 
mkdir /tmp/www
echo "i am vm1" > /tmp/www/index.html
cd /tmp/www
ip netns exec vm1 python3 -m http.erver 8000 &
ip route add 10.127.0.0/16 via 172.16.255.129 dev ens7  //因为我们要从外部访问vm，所以需要在vm中配置到外部网络的路由

# vm2
mkdir /tmp/www
echo "i am vm2" > /tmp/www/index.html
cd /tmp/www
ip netns exec vm1 python3 -m http.erver 8000 &
ip route add 10.127.0.0/16 via 172.16.255.129 dev ens7  //因为我们要从外部访问vm，所以需要在vm中配置到外部网络的路由

# 在北向数据库的load_balancer表中创建一个条目，并将生成的UUID存储到变量“uuid”。 我们将在后面的命令中引用这个变量。
uuid=`ovn-nbctl create load_balancer vips:10.127.0.254="172.16.255.130,172.16.255.131"`
echo $uuid

# 在OVN网关路由器“edge1”上开启负载均衡器功能。
ovn-nbctl set logical_router edge1 load_balancer=$uuid
ovn-nbctl get logical_router edge1 load_balancer

# 从外部网络访问web服务器，可以看到负载均衡生效了
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm2
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm2
[root@dell-per730-42 ~]# curl 10.127.0.254:8000
i am vm1

# OVN负载均衡不会检查某个服务器是否正在运行，所以如果我们在vm1或者vm2中停止web服务，
# 那么某些时候访问web服务就会失败

# 清理
ovn-nbctl clear logical_router edge1 load_balancer
ovn-nbctl destroy load_balancer $uuid
```

## 负载均衡应用于逻辑交换机
```
uuid=`ovn-nbctl create load_balancer vips:172.16.255.62="172.16.255.130,172.16.255.131"`
echo $uuid
ovn-nbctl set logical_switch inside load_balancer=$uuid
ovn-nbctl get logical_switch inside load_balancer

# vm1和vm2需要配置到inside的路由
ip route add 172.16.255.192/26 via 172.16.255.129 dev ens7

# vm3发起请求
[root@localhost ~]# curl 172.16.255.62:8000
i am vm2
[root@localhost ~]# curl 172.16.255.62:8000
i am vm2
[root@localhost ~]# curl 172.16.255.62:8000
i am vm1
[root@localhost ~]# curl 172.16.255.62:8000
i am vm2
[root@localhost ~]# curl 172.16.255.62:8000
i am vm2

# 从vm1和vm2发起请求是不通的
# 这是因为只能对客户端的逻辑交换机而不是服务器的逻辑交换机应用负载平衡
# 向下面这样把负载均衡应用于dmz交换机之后，不论从哪个vm发起请求，就都不通了
ovn-nbctl clear logical_switch inside load_balancer
ovn-nbctl set logical_switch dmz load_balancer=$uuid
ovn-nbctl get logical_switch dmz load_balancer
ovn-nbctl clear logical_switch dmz load_balancer
ovn-nbctl destroy load_balancer $uuid
```

