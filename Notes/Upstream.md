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

   