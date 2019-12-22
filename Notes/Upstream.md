# 使用Nginx代理Tomcat集群

## 开始

**效果如下：**

![](https://tva1.sinaimg.cn/large/006tNbRwly1ga5mh9d57og30hs08wno3.gif)

1. 使用`docker-compose`启动3个`tomcat`

   docker-compose.yml内容如下:

   ```yml
   version: '3'
   services:
     tomcat1:
       restart: always
       image: tomcat
       ports:
         - 8080:8080
     tomcat2:
       restart: always
       image: tomcat
       ports:
         - 8081:8080
     tomcat3:
       restart: always
       image: tomcat
       ports:
         - 8082:8080
   ```

   `docker-compose`启动

   ```bash
   docker-compose up -d
   ```

2. 依次进入每个tomcat中，修改`index.jsp`

   ```bash
   # tomcat 1
   docker exec -it ngnix-learn_tomcat1_1 bash
   echo '<h1>Tomcat 1</h1>' > webapp/ROOT/index.jsp
   # tomcat 2
   docker exec -it ngnix-learn_tomcat2_1 bash
   echo '<h1>Tomcat 2</h1>' > webapp/ROOT/index.jsp
   # tomcat 3
   docker exec -it ngnix-learn_tomcat3_1 bash
   echo '<h1>Tomcat 3</h1>' > webapp/ROOT/index.jsp
   ```

3. 访问测试是否启动成功

   ```bash
   http://localhost:8080 #tomcat1
   http://localhost:8081 #tomcat2
   http://localhost:8082 #tomcat3
   ```

4. 修改`nginx.conf`配置

   ```conf
   upstream tomcats {
     server 192.168.0.101:8080;
     server 192.168.0.101:8081;
     server 192.168.0.101:8082;
   }
   
   server {
     listen 81;
     server_name www.tomcats.com;
   
     location / {
     	proxy_pass http://tomcats;
     }
   }
   ```

   5.重启测试

   ```bash
   # 测试配置是否正确
   nginx/sbin/nginx -t
   # 重启
   nginx/sbin/nginx -s reload
   ```

   

## Nginx负载均衡（Load-Balance）策略

- 轮询默认
- 加权轮询  权重
- IP-HASH
- URL_HASH
- LEAST_CONN 最少连接

## Nginx负载均衡——加权轮询

配置如下:

```conf
upstream tomcats {
  server 192.168.0.101:8080 weight=1;
  server 192.168.0.101:8081 weight=2;
  server 192.168.0.101:8082 weight=3;
}
```

重启

```bash
# 测试配置是否正确
nginx/sbin/nginx -t
# 重启
nginx/sbin/nginx -s reload
```

效果：

![Dec-22-2019 17-44-30](https://tva1.sinaimg.cn/large/006tNbRwly1ga5o3ddlgsg30hs08w7wj.gif)

## Nginx负载均衡——IP_HASH

`ip_hash` 可以保证用户访问可以请求到上游服务中的固定的服务器，前提是用户ip没有发生更改。
使用ip_hash的注意点：
不能把后台服务器直接移除，只能标记`down`.

```
upstream tomcats {
	ip_hash;
	
  server 192.168.0.101:8080 weight=1;
  server 192.168.0.101:8081 weight=2;
  server 192.168.0.101:8082 weight=3;
}
```

### 一致性哈希算法

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1ga5p6dlxktj30yh0jf7cm.jpg" alt="image-20191222185447471" style="zoom: 33%;" />

ip_hash有一个问题就是，当集群节点发生变化（增加节点/删除节点）的时候需要重新计算所有ip的hash值

并且会丢失会话数据(session)

为了解决这个问题，所以nginx提供了一致性哈希算法来解决

在逻辑上将集群中的每台主机放在一个圆圈内（0-2^32-1）

用户会寻找离自己最近的一个节点（顺时针方向）

这样每个用户会依次落在相应的节点中

例如：现在节点3挂掉了，只需要将归属于节点3的用户重新挂在节点4上面即可，这样可以有效减少hash计算的次数

## Nginx负载均衡——URL_HASH

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1ga5pgm769aj30ny0iate7.jpg" alt="image-20191222190439364" style="zoom: 50%;" />

URL_HASH会将用户的请求进行hash来负载均衡到不同的服务器中

配置方式如下:

```
upstream tomcats {
	# url_hash
	hash $request_uri;
	
  server 192.168.0.101:8080;
  server 192.168.0.101:8081;
  server 192.168.0.101:8082;
}
```

## Nginx upstream指令参数

- max_conns	最大连接数（限流）

  配置如下

  ```
  # max_conns
  server 192.168.0.101:8080 max_conns=1;
  server 192.168.0.101:8081 max_conns=2;
  server 192.168.0.101:8082 max_conns=3;
  # 使用一个工作进程测试，防止nginx使用共享内存
  worker_processes  1; 
  ```

- slow_start     慢启动（只有nignx商业版支持）

  ```
  # slow_start 60s才能完成启动，权重会从0慢慢升级到10（需要60s）
  server 192.168.0.101:8080 weight=10 slow_start=60s;
  ```

- down

  ```
  # down 用于标记服务节点不可用：
  server 192.168.0.101:8080 down;
  ```

- backup

  ```
  # backup 表示当前服务器节点是备用机，只有在其他的服务器都宕机以后，自己才会加入到集群中，被用户访问到：
  server 192.168.0.101:8080 backup;
  ```

- max_fails

  ```
  # max_fails 当主机最大失败次数到达之后，就认为该主机挂掉了
  ```

- fail_timeout

  ```
  #  fail_timeout 失败超时时间 当请求失败时会等待一个时间后再次请求
  ```

  

## Keepalived 提高吞吐量

`keepalived`： 设置长连接处理的数量
`proxy_http_version`：设置长连接http版本为1.1
`proxy_set_header`：清除connection header 信息

```
upstream tomcats {
	keepalive 32;
}

server {
  listen       80;
  server_name  www.tomcats.com;

	location / {
    proxy_pass  http://tomcats;
    proxy_http_version 1.1;
    proxy_set_header Connection "";
	}
}

```

