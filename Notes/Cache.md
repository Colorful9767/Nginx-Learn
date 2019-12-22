# Nginx 缓存

<img src="https://tva1.sinaimg.cn/large/006tNbRwly1ga5pnxtcooj30uy0gw0xf.jpg" alt="image-20191222191141996" style="zoom: 50%;" />

1. 浏览器缓存：
   - 加速用户访问，提升单个用户（浏览器访问者）体验，缓存在本地
2. Nginx缓存
   - 缓存在nginx端，提升所有访问到nginx这一端的用户
   - 提升访问上游（upstream）服务器的速度
   - 用户访问仍然会产生请求流量

- 控制浏览器缓存：

  ```
  location /files {
      alias /home/admin;
      # expires 10s;
      # expires @22h30m;
      # expires -1h;
      # expires epoch;
      # expires off;
      expires max;
  }
  ```

- Nginx缓存

  ```
  # proxy_cache_path 设置缓存目录
  #       keys_zone 设置共享内存以及占用空间大小
  #       max_size 设置缓存大小
  #       inactive 超过此时间则被清理
  #       use_temp_path 临时目录，使用后会影响nginx性能
  proxy_cache_path /usr/local/nginx/upstream_cache keys_zone=mycache:5m max_size=1g inactive=1m use_temp_path=off;
  
  location / {
      proxy_pass  http://tomcats;
  
      # 启用缓存，和keys_zone一致
      proxy_cache mycache;
      # 针对200和304状态码缓存时间为8小时
      proxy_cache_valid   200 304 8h;
  }
  ```

  