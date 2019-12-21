# Nginx核心配置

## 1、设置worker进程的用户，指的linux中的用户，会涉及到nginx操作目录或文件的一些权限，默认为`nobody`

```
user root;
```

## 2、worker进程工作数设置，一般来说CPU有几个，就设置几个，或者设置为N-1也行

```
worker_processes 1;
```

## 3、nginx 日志级别，错误级别从左到右越来越大

- debug
- info
-  notice
-  warn
- error
- crit
- alert
- emerg

## 4、设置nginx进程 pid

```
pid        logs/nginx.pid;
```

## 5、设置工作模式

```
events {
    # 默认使用epoll
    use epoll;
    # 每个worker允许连接的客户端最大连接数
    worker_connections  10240;
}
```

## 6、http 是指令块，针对http网络传输的一些指令配置

```
http {
}
```

## 7、include 引入外部配置，提高可读性，避免单个配置文件过大

```
include       mime.types;
```

## 8、设定日志格式，`main`为定义的格式名称，如此 access_log 就可以直接使用这个变量了

| 参数名                | 参数意义                             |
| :-------------------- | :----------------------------------- |
| $remote_addr          | 客户端ip                             |
| $remote_user          | 远程客户端用户名，一般为：’-’        |
| $time_local           | 时间和时区                           |
| $request              | 请求的url以及method                  |
| $status               | 响应状态码                           |
| $body_bytes_send      | 响应客户端内容字节数                 |
| $http_referer         | 记录用户从哪个链接跳转过来的         |
| $http_user_agent      | 用户所使用的代理，一般来时都是浏览器 |
| $http_x_forwarded_for | 通过代理服务器来记录客户端的ip       |

## 9、`sendfile`使用高效文件传输，提升传输性能。启用后才能使用`tcp_nopush`，是指当数据表累积一定大小后才发送，提高了效率。

```
sendfile        on;
tcp_nopush      on;
```

## 10、`keepalive_timeout`设置客户端与服务端请求的超时时间，保证客户端多次请求的时候不会重复建立新的连接，节约资源损耗。

```
#keepalive_timeout  0;
keepalive_timeout  65;
```

## 11、`gzip`启用压缩，html/js/css压缩后传输会更快

```
gzip on;
```

## 12、`server`可以在`http`指令块中设置多个虚拟主机

- listen 监听端口
- server_name localhost、ip、域名
- location 请求路由映射，匹配拦截
- root 请求位置
- index 首页设置

```xml
server {
  listen       88;
  server_name  localhost;

  location / {
    root   html;
    index  index.html index.htm;
	}
}
```

## 13、root与alias

假如图片路径为：/home/admin/Ngnix-Learn/files/img/go.png

- root 

  ```xml
  location /img {
      root /home/admin/Ngnix-Learn/files;
  }
  ```

  解析方式：访问root+location拼接起来的值

  用户访问的时候请求为：`url:port/img/go.png`

- alias

  ```
  location /img {
  		alias /home/admin/Ngnix-Learn/files/img;
  }
  ```

  解析方式：访问location的值替换为alias的值

  请求地址：`url:port/img/go.png`

  二者虽然最后访问的地址都一样，但是注意二者解析方式的区别

**注意：root和alias里面的路径一定要加上`;`不然会报错**

## 14、location 的匹配规则

- `空格`：默认匹配，普通匹配

  ```xml
  location / {
       root /home;
  }
  ```

- `=`：精确匹配

  ```xml
  location = /imooc/img/face1.png {
      root /home;
  }
  ```

- `~*`：匹配正则表达式，不区分大小写

  ```xml
  #符合图片的显示
  location ~ \.(GIF|jpg|png|jpeg) {
      root /home;
  }
  ```

- `~`：匹配正则表达式，区分大小写

  ```xml
  #GIF必须大写才能匹配到
  location ~ \.(GIF|jpg|png|jpeg) {
      root /home;
  }
  ```

- `^~`：以某个字符路径开头

  ```xml
  location ^~ /imooc/img {
      root /home;
  }
  ```

