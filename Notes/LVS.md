# LVS

## 介绍

LVS（Linux Virtual Server）即Linux虚拟服务器，是由章文嵩博士主导的开源负载均衡项目，目前LVS已经被集成到Linux内核模块中。该项目在Linux内核中实现了基于IP的数据请求负载均衡调度方案，终端互联网用户从外部访问公司的外部负载均衡服务器，终端用户的Web请求会发送给LVS调度器，调度器根据自己预设的算法决定将该请求发送给后端的某台Web服务器，比如，轮询算法可以将外部的请求平均分发给后端的所有服务器，终端用户访问LVS调度器虽然会被转发到后端真实的服务器，但如果真实服务器连接的是相同的存储，提供的服务也是相同的服务，最终用户不管是访问哪台真实服务器，得到的服务内容都是一样的，整个集群对用户而言都是透明的。

## 为什么要使用LVS+Nginx

- LVS基于四层网络协议（Nginx基于七层），工作效率更高
- 单个Nginx如果承受不了压力，需要Nginx集群
- LVS可以充当Nginx集群的调度者
- Nginx要将用户的请求进行转发，而LVS可以只接受请求不响应，提高工作效率

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacg1ic9i6j31oi0u04h2.jpg" style="zoom:33%;" />

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacg1xu5s3j31m00u01dj.jpg" style="zoom:33%;" />

可以看出LVS和Nginx对于用户请求的响应可以有不同的处理方式，LVS可以接受更大的用户请求

## LVS三种模式

- NAT模式

  <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gablq7u4l7j31ml0u0nif.jpg" style="zoom: 33%;" />

  #### 什么是NAT？

  NAT（Network Address Translation），即网络地址转换。当在专用网内部的一些主机本来已经分配到了本地IP地址（即仅在本专用网内使用的专用地址），但现在又想和因特网上的主机通信（并不需要加密）。简单来说NAT就是将内网ip和公网地址进行转换，这样就可以保护内网服务器提高安全性。

  NAT模式是基于网络地址的转换，当用户通过浏览器输入ip时，LVS会虚拟一个公网ip出来（这里假设为192.168.1.150），经过LVS负载后会将请求分发到负载的某台服务器中。当服务器处理完请求后，会将请求发给LVS再由LVS发送给客户端（类似于Nginx反向代理功能）。

- TUN模式

  <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gabm2gy8iij31ly0u01f9.jpg" style="zoom:33%;" />

  TUN是一种IP隧道模式（即在逻辑上将每台服务器和客户端之间形成一条隧道），TUN和NAT接收请求的工作模式类似。唯一的区别在于TUN模式在返回用户请求的时候是不经过LVS服务器，而是直接回将响应发送至客户端浏览器。这样可以减少LVS的压力，上行压力较小（LVS），下行流量大（因为下行时理论上可以挂无数服务）。唯一的缺点是每一个服务器必须配备一张网卡，并且服务端的ip地址会暴露给客户端。

- DR模式

  <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gabmbhep3xj31f60u07sc.jpg" style="zoom:33%;" />

  DR（Direct Route）模式是对TUN模式的进一步优化，它解决了TUN模式将后端服务器地址暴露在公网上的缺点。DR模式会有一个路由（Router）存在，所有后端服务的响应会发送至Router中，这里Router也有一个虚拟ip（VIP）。所有用户响应会交由Router来发送至客户端，这样就能保护后端服务提高服务的安全性。

  

## LVS的负载均衡算法

### 静态算法

静态：根据LVS本身自由的固定的算法分发用户请求。

1. 轮询（Round Robin 简写’rr’）：轮询算法假设所有的服务器处理请求的能力都一样的，调度器会把所有的请求平均分配给每个真实服务器。（同Nginx的轮询）
2. 加权轮询（Weight Round Robin 简写’wrr’）：安装权重比例分配用户请求。权重越高，被分配到处理的请求越多。（同Nginx的权重）
3. 源地址散列（Source Hash 简写’sh’）：同一个用户ip的请求，会由同一个RS来处理。（同Nginx的ip_hash）
4. 目标地址散列（Destination Hash 简写’dh’）：根据url的不同，请求到不同的RS。（同Nginx的url_hash）

### 动态算法

动态：会根据流量的不同，或者服务器的压力不同来分配用户请求，这是动态计算的。

1. 最小连接数（Least Connections 简写’lc’）：把新的连接请求分配到当前连接数最小的服务器。
2. 加权最少连接数（Weight Least Connections 简写’wlc’）：服务器的处理性能用数值来代表，权重越大处理的请求越多。Real Server 有可能会存在性能上的差异，wlc动态获取不同服务器的负载状况，把请求分发到性能好并且比较空闲的服务器。
3. 最短期望延迟（Shortest Expected Delay 简写’sed’）：特殊的wlc算法。举例阐述，假设有ABC三台服务器，权重分别为1、2、3 。如果使用wlc算法的话，当一个新请求进来，它可能会分给ABC中的任意一个。使用sed算法后会进行如下运算：
   - A：（1+1）/1=2
   - B：（1+2）/2=3/2
   - C：（1+3）/3=4/3
     最终结果，会把这个请求交给得出运算结果最小的服务器。
4. 最少队列调度（Never Queue 简写’nq’）：永不使用队列。如果有Real Server的连接数等于0，则直接把这个请求分配过去，不需要在排队等待运算了（sed运算）。

### 总结：

LVS在实际使用过程中，负载均衡算法用的较多的分别为wlc或wrr，简单易用。

## 搭建LVS-DR模式- 配置LVS节点与ipvsadm

### 主机信息

- LVS（一台）
  - VIP：10.211.55.100
  - DIP：10.211.55.29
- Nginx（两台）
  - （N1）RIP：10.211.55.26
  - （N2）RIP：10.211.55.27

### 关闭网络配置管理器

所有计算机节点关闭网络配置管理器，因为有可能会和网络接口冲突：

```bash
systemctl stop NetworkManager 
systemctl disable NetworkManager
```

### 创建子接口

1、进入到网卡配置目录，拷贝`eth0`并且创建子接口：

```bash
cd /etc/sysconfig/network-scripts/
ls -l | grep eth0
# 注：`1`为别名，可以任取其他数字都行
cp ifcfg-eth0 ifcfg-eth0:1
```

![](https://tva1.sinaimg.cn/large/006tNbRwly1gacgv0wz6sj30ws0a00v5.jpg)

2、修改`ifcfg-eth0:1`子接口配置： 

```bash
# Generated by parse-kickstart
BOOTPROTO=static
DEVICE=eth0
ONBOOT=yes
IPADDR=10.211.55.100
NETMASK=255.255.255.0
```

3、重启`network`服务

```bash
systemctl restart network
# service network restart
```

4、查看当前网卡信息

```bash
ip addr
```

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gach3xe0bgj31l00jk0xh.jpg" style="zoom: 50%;" />

### 安装`ipvsadm`

现如今的centos都是集成了LVS，所以`ipvs`是自带的，相当于苹果手机自带ios，我们只需要安装`ipvsadm`即可（ipvsadm是管理集群的工具，通过ipvs可以管理集群，查看集群等操作）

1、`yum`安装

```bash
yum install -y ipvsadm
```

2、检查是否安装成功

```bash
ipvsadm -Ln
```

![](https://tva1.sinaimg.cn/large/006tNbRwly1gach5w4j2xj313606mabg.jpg)

注：阿里云不支持虚拟IP，需要购买他的负载均衡服务.腾讯云支持虚拟IP，但是需要额外购买，一台节点最大支持10个虚拟ip

## 搭建LVS-DR模式- 为两台RS配置虚拟IP

### 配置虚拟网络子接口（回环接口）

1、查看当前网络信息

![](https://tva1.sinaimg.cn/large/006tNbRwly1gachkbe7qqj31kq0i243q.jpg)

1、进入到网卡配置目录，找到lo（本地环回接口，用户构建虚拟网络子接口），拷贝一份新的随后进行修改：

![](https://tva1.sinaimg.cn/large/006tNbRwly1gachjqnzl0j30xi06imyk.jpg)

2、修改内容如下：

```bash
DEVICE=lo
# 这里需要将ip指定为VIP的地址
IPADDR=10.211.55.100
NETMASK=255.255.255.255
NETWORK=127.0.0.0
BROADCAST=127.255.255.255
ONBOOT=yes
NAME=loopback
```

3、重启后通过ip addr 查看如下，表示ok：

![](https://tva1.sinaimg.cn/large/006tNbRwly1gachmbfs16j319y0k2dk9.jpg)

## 搭建LVS-DR模式- 为两台RS配置arp

### ARP响应级别与通告行为 的概念

1. arp-ignore：ARP响应级别（处理请求）

   - 0：只要本机配置了ip，就能响应请求

   - 1：请求的目标地址到达对应的网络接口，才会响应请求

2. arp-announce：ARP通告行为（返回响应）

   - 0：本机上任何网络接口都向外通告，所有的网卡都能接受到通告
   - 1：尽可能避免本网卡与不匹配的目标进行通告
   - 2：只在本网卡通告

### 配置ARP

1. 打开sysctl.conf:

   ```bash
   vim /etc/sysctl.conf
   ```

2. 配置`所有网卡`、`默认网卡`以及`虚拟网卡`的arp响应级别和通告行为，分别对应：`all`，`default`，`lo`：

   ```conf
   # configration for lvs
   net.ipv4.conf.all.arp_ignore = 1
   net.ipv4.conf.default.arp_ignore = 1
   net.ipv4.conf.lo.arp_ignore = 1
   
   net.ipv4.conf.all.arp_announce = 2
   net.ipv4.conf.default.arp_announce = 2
   net.ipv4.conf.lo.arp_announce = 2
   ```

3. 刷新配置文件：

   ```bash
   sysctl -p
   ```

4. 增加一个网关，用于接收数据报文，当有请求到本机后，会交给lo去处理：

   ```bash
   # 这里指定host为VIP
   route add -host 10.211.55.100 dev lo:1
   ```

5. 防止重启失效，做如下处理，用于开机自启动：

   ```bash
   echo "route add -host 10.211.55.100 dev lo:1" >> /etc/rc.local
   ```

## 搭建LVS-DR模式- 使用ipvsadm配置集群规则

1. 创建LVS节点，用户访问的集群调度者

   ```bash
   ipvsadm -A -t 10.211.55.100:80 -s rr -p 5
   ```

   - -A：添加集群
   - -t：tcp协议
   - ip地址：设定集群的访问ip，也就是LVS的虚拟ip
   - -s：设置负载均衡的算法，rr表示轮询
   - -p：设置连接持久化的时间

2. 创建2台RS真实服务器

   ```bash
   ipvsadm -a -t 10.211.55.100:80 -r 10.211.55.26:80 -g
   ipvsadm -a -t 10.211.55.100:80 -r 10.211.55.27:80 -g
   ```

   - -a：添加真实服务器
   - -t：tcp协议
   - -r：真实服务器的ip地址
   - -g：设定DR模式

3. 保存到规则库，否则重启失效

   ```bash
   ipvsadm -S
   ```

4. 检查集群

   - 查看集群列表

     ```bash
     ipvsadm -Ln
     ```

   - 查看集群状态

     ```bash
     ipvsadm -Ln --stats
     ```

5. 其他命令：

   ```bash
   # 重启ipvsadm，重启后需要重新配置
   service ipvsadm restart
   # 查看持久化连接
   ipvsadm -Ln --persistent-conn
   # 查看连接请求过期时间以及请求源ip和目标ip
   ipvsadm -Lnc
   
   # 设置tcp tcpfin udp 的过期时间（一般保持默认）
   ipvsadm --set 1 1 1
   # 查看过期时间
   ipvsadm -Ln --timeout
   ```

6. 更详细的帮助文档：

   ```bash
   ipvsadm -h
   man ipvsadm
   ```

   