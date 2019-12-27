# 安装Nginx

1、安装nginx依赖环境

```bash
 yum install -y gcc-c++ \
                pcre pcre-devel \
                zlib zlib-devel \
                openssl openssl-devel
```

- gcc Nginx需要gcc编译
- prce 用于解析正则表达式
- zlib 压缩和解压缩依赖
- ssl 安全的加密的套接字协议层，用于HTTP安全传输，也就是https

2、下载安装包

官网地址http://nginx.org/

```bash
# 下载
wget http://nginx.org/download/nginx-1.16.1.tar.gz
# 解压
tar -zxvf nginx-1.16.1.tar.gz
```

3、编译之前，先创建nginx临时目录，如果不创建，在启动nginx的过程中会报错

```bash
mkdir /var/temp/nginx -p
```

4、安装前配置

为了之后方便查看，将一些文件指定到/var/temp/nginx中

```bash
./configure \
--prefix=/usr/local/nginx \
--pid-path=/var/temp/nginx/run/nginx.pid \
--lock-path=/var/temp/nginx/lock/nginx.lock \
--error-log-path=/var/temp/nginx/logs/error.log \
--http-log-path=/var/temp/nginx/logs/access.log \
--with-http_gzip_static_module \
--http-client-body-temp-path=/var/temp/nginx/client \
--http-proxy-temp-path=/var/temp/nginx/proxy \
--http-fastcgi-temp-path=/var/temp/nginx/fastcgi \
--http-uwsgi-temp-path=/var/temp/nginx/uwsgi \
--http-scgi-temp-path=/var/temp/nginx/scgi
```

- 配置命令：

  | 命令                          | 解释                                 |
  | :---------------------------- | :----------------------------------- |
  | –prefix                       | 指定nginx安装目录                    |
  | –pid-path                     | 指向nginx的pid                       |
  | –lock-path                    | 锁定安装文件，防止被恶意篡改或误操作 |
  | –error-log                    | 错误日志                             |
  | –http-log-path                | http日志                             |
  | –with-http_gzip_static_module | 启用gzip模块，在线实时压缩输出数据流 |
  | –http-client-body-temp-path   | 设定客户端请求的临时目录             |
  | –http-proxy-temp-path         | 设定http代理临时目录                 |
  | –http-fastcgi-temp-path       | 设定fastcgi临时目录                  |
  | –http-uwsgi-temp-path         | 设定uwsgi临时目录                    |
  | –http-scgi-temp-path          | 设定scgi临时目录                     |

5、编译安装

```bash
make && make install
```

6、进入sbin目录启动nginx

```bash
./nginx
```

- 停止：./nginx -s stop
- 重新加载：./nginx -s reload
- 测试配置文件是否正确: ./nginx -t

##  错误记录

 1、使用 `-t` 检查配置文件时，提示不能绑定80端口

提示信息如下：

```bash
[admin@centos-linux sbin]$ ./nginx -t
nginx: the configuration file /home/admin/Ngnix-Learn/nginx/conf/nginx.conf syntax is ok
nginx: [emerg] bind() to 0.0.0.0:80 failed (13: Permission denied)
nginx: configuration file /home/admin/Ngnix-Learn/nginx/conf/nginx.conf test failed
```

解决办法：

1. 由于启动时非root用户无法启动80端口，改为root启动即可
2. 将nginx的端口改为1024以上

2、启动后访问80端口提示： 403 forbidden

由于woker和master进程的所属用户不一致导致

```bash
[admin@centos-linux sbin]$ ps -ef | grep nginx
root      7255     1  0 08:59 ?        00:00:00 nginx: master process ./nginx
nobody    7256  7255  0 08:59 ?        00:00:00 nginx: worker process
admin     7260  1744  0 09:00 pts/0    00:00:00 grep --color=auto nginx
```

解决办法:修改`conf/nginx.conf` 将user 默认的`nobody`改为`root`
