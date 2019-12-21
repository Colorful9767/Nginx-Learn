# Nginx日志切割

现有的日志都会存在 `access.log` 文件中，但是随着时间的推移，这个文件的内容会越来越多，体积会越来越大，不便于运维人员查看，所以我们可以通过把这个大的日志文件切割为多份不同的小文件作为日志，切割规则可以以`天`为单位，如果每天有几百G或者几个T的日志的话，则可以按需以`每半天`或者`每小时`对日志切割一下。

**具体步骤如下：**

1. 创建一个shell可执行文件：`cut_logs_everyday.sh`，内容为：

   ```shell
   #!/bin/bash
   LOG_PATH="/var/temp/nginx/logs"
   RECORD_TIME=$(date -d "yesterday" +%Y-%m-%d+%H:%M)
   PID=/var/temp/nginx/run/nginx.pid
   mv ${LOG_PATH}/access.log ${LOG_PATH}/access.${RECORD_TIME}.log
   mv ${LOG_PATH}/error.log ${LOG_PATH}/error.${RECORD_TIME}.log
   
   #向Nginx主进程发送信号，用于重新打开日志文件
   kill -USR1 `cat $PID`
   ```

2. 为`cut_logs_everyday.sh`添加可执行的权限：

   ```bash
   chmod +x ./cut_logs_everyday.sh
   ```

3. 测试日志切割后的结果:

   ```bash
   ./cut_logs_everyday.sh
   ```

## 使用Crontab定时切割日志

1、安装`crontab`

```bash
yum install crontabs
```

2、启动服务

```bash
systemctl start crond
```

3、添加任务

```bash
crontab -e
# 这里路径要使用绝对路径
*/1 * * * * /home/admin/Ngnix-Learn/cut_logs_everyday.sh
```

- 查看任务列表 `crontab -l`
- 新建任务 `crontab -e`

