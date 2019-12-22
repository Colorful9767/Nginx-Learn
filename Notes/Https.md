# 使用Nginx配置HTTPS域名证书

#### 1. 安装SSL模块

要在nginx中配置https，就必须安装ssl模块，也就是: `http_ssl_module`。

- 进入到nginx的解压目录： /home/software/nginx-1.16.1

- 新增ssl模块(原来的那些模块需要保留)

  ```xml
  ./configure \
  --prefix=/usr/local/nginx \
  --pid-path=/var/run/nginx/nginx.pid \
  --lock-path=/var/lock/nginx.lock \
  --error-log-path=/var/log/nginx/error.log \
  --http-log-path=/var/log/nginx/access.log \
  --with-http_gzip_static_module \
  --http-client-body-temp-path=/var/temp/nginx/client \
  --http-proxy-temp-path=/var/temp/nginx/proxy \
  --http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
  --http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
  --http-scgi-temp-path=/var/temp/nginx/scgi  \
  --with-http_ssl_module
  ```

- 编译和安装

  ```
  make
  
  make install
  ```

#### 2. 配置HTTPS

- 把ssl证书 `*.crt` 和 私钥 `*.key` 拷贝到`/usr/local/nginx/conf`目录中。

- 新增 server 监听 443 端口：

  ```xml
  server {
      listen       443;
      server_name  www.xxxx.com;
  
      # 开启ssl
      ssl     on;
      # 配置ssl证书
      ssl_certificate      1_www.xxxx.com_bundle.crt;
      # 配置证书秘钥
      ssl_certificate_key  2_www.xxxx.com.key;
  
      # ssl会话cache
      ssl_session_cache    shared:SSL:1m;
      # ssl会话超时时间
      ssl_session_timeout  5m;
  
      # 配置加密套件，写法遵循 openssl 标准
      ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
      ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:HIGH:!aNULL:!MD5:!RC4:!DHE;
      ssl_prefer_server_ciphers on;
      
      location / {
          proxy_pass http://tomcats/;
          index  index.html index.htm;
      }
   }
  ```

#### 3. 重启 nginx

```
./nginx -s reload
```

