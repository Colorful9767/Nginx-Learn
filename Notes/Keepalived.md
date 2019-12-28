# Keepalived安装

1、下载安装包

官网地址:https://www.keepalived.org/download.html

```bash
wget https://www.keepalived.org/software/keepalived-2.0.19.tar.gz
```

2、解压

```bash
tar -zxvf keepalived-2.0.19.tar.gz
```

3、配置

```bash
./configure \
--prefix=/usr/local/keepalived \
--sysconf=/etc
```

4、编译安装

```bash
make && make install
```

- prefix：keepalived安装的位置
- sysconf：keepalived核心配置文件所在位置，固定位置，改成其他位置则keepalived启动不了，/var/log/messages中会报错

**注意：**

- 安装过程中出现如下警告：

  <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacei64skbj31zm03ot9a.jpg" style="zoom:50%;" />

  ```bash
  yum install -y libnl libnl-devel
  ```

- 安装过程中如果提示缺少`openssl`

  <img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacegl1ui0j30xa03omxq.jpg" style="zoom: 50%;" />

  ```bash
  yum install -y openssl-devel
  ```

5、进入Keepalived安装目录

```bash
whereis keepalived
cd /etc/keepalived
```

![](https://tva1.sinaimg.cn/large/006tNbRwly1gacek8rxzfj30z202mq3e.jpg)

## Keepalived 双机主备

**注:我测试环境的ip和图上的地址不一致，根据自身环境ip进行相应修改**

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacfj5p1k8j31aj0u0dzh.jpg" style="zoom:33%;" />

### 主配置（Master）

1、查看本机网卡信息

```bash
# 我这台虚拟机的网卡为eth0，有的虚拟机可能是ens33之类的名字
ip addr
```

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gaceqg1itoj31km0hon1y.jpg" style="zoom:33%;" />

2、修改`keepalived.conf`

养成好习惯，修改之前最好先备份一次`cp keepalived.conf keepalived.conf.bak`

```conf
global_defs {
   # 路由id：当前安装keepalived的节点主机标识符，保证全局唯一
   router_id keep_1
}

vrrp_instance VI_1 {
    # 表示状态 MASTER|BACKUP
    state MASTER
    # 该实例绑定的网卡
    interface eth0
    # 保证主备节点一致即可
    virtual_router_id 51
    # 权重，master权重一般高于backup，如果有多个，那就是选举，谁的权重高，谁就当选
    priority 100
    # 主备之间同步检查时间间隔2s，单位秒
    advert_int 2
    # 认证权限密码，防止非法节点进入
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    # 虚拟出来的ip，可以有多个（vip）
    virtual_ipaddress {
        10.211.55.100
    }
}
```

3、启动`keepalived`

```bash
# 默认路径在/usr/local/keepalived/sbin 最好先用whereis命令查看一下路径
./keepalived
```

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacevx5r0lj30vm08mgnb.jpg" style="zoom: 50%;" />

4、查看现在的网卡信息，会发现在`eth0`下面会多出一个虚拟ip（即之前再keepalived配置文件中设置的vip）

```bash
ip addr
```

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gaceye2pkpj31ki0miwjz.jpg" style="zoom:50%;" />

5、查看keepalived进程

```bash
ps -ef | grep keepalived
```

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacf0cuk5dj316y06smys.jpg" style="zoom:50%;" />

### 备用机配置（Backup）

```conf
global_defs {
   router_id keep_2
}

vrrp_instance VI_1 {
    state BACKUP
    interface eth0
    virtual_router_id 51
    priority 80
    advert_int 2
    authentication {
        auth_type PASS
        auth_pass 1111
    }
    virtual_ipaddress {
        10.211.55.100
    }
}
```

这里所有配置几乎和`Master`一样，不同的配置有以下几点：

- `router_id`	这里可以认为是keepalived的唯一id，需要区分开两台keepalived服务
- `state` 备用机状态BACKUP
- `priority` 备用机的优先级自然需要低于主节点，官方推荐Master和Backup的优先级差值为50

其余配置一样即可

同样需要进入sbin目录下启动keepalived

```bash
# 启动
./keepalived
# 查看进程状态
ps -ef | grep keepalived
# 查看虚拟ip是否配置成功
ip addr
```

## Keepalived双主热备

keepalived双主热备，示意图如下：

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1gacfmjp0jjj31n60u04qp.jpg" style="zoom:33%;" />

注意：这里的DNS轮询一般都是交由第三方云服务提供商处理，即访问一个域名会负载到两个ip地址上，也可以认为是一种DNS端的LB（Load Balance）

核心思想就是讲Master/Backup两台主机互为主备关系。即：

对于主机A它自己是自己的Master，同时也是主机B的Backup

同样的

对于主机B它自己是自己的Master，同时也是主机A的Backup

这样无论哪台主机挂了，另一台就会成为替代者持续服务，这样就解决了只有一台主机在工作，另一台有可能一直不工作。

提高了主机的利用率。

## Keepalived配置Nginx自动重启

1、增加Nginx重启检测脚本`check_nginx_alive_or_not.sh`

```shell
#!/bin/bash

A=`ps -C nginx --no-header |wc -l`
# 判断nginx是否宕机，如果宕机了，尝试重启
if [ $A -eq 0 ];then
    /usr/local/nginx/sbin/nginx
    # 等待一小会再次检查nginx，如果没有启动成功，则停止keepalived，使其启动备用机
    sleep 3
    if [ `ps -C nginx --no-header |wc -l` -eq 0 ];then
        killall keepalived
    fi
fi
```

2、增加运行权限

```bash
chmod +x ./check_nginx_alive_or_not.sh
```

3、配置`keepalived.conf`监听nginx脚本

```conf
vrrp_script check_nginx_alive {
    script "/etc/keepalived/check_nginx_alive_or_not.sh"
    interval 2 # 每隔两秒运行上一行脚本
    weight 10 # 如果脚本运行失败，则升级权重+10
}

track_script {
		# 追踪 nginx 脚本
    check_nginx_alive   
}
```

4、重启`keepalived`

```bash
# 获得进程号
ps -ef | grep keepalived
# 强制结束进程
kill -9 {pid}
# 启动keepalived
./keepalived
```

