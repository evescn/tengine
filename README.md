# Simple docker image of Tengine web server based on Alpine #
***

- [more about Tengine](http://tengine.taobao.org)
- [docs](http://tengine.taobao.org/documentation.html)

in this image added [Upstream check module](http://tengine.taobao.org/document/http_upstream_check.html)

to start just type something like that:
```
docker exec -it -d -v example.com.conf:/etc/nginx/conf.d/example.com.conf \
    -p "80:80" -p 443:443 axizdkr/tengine
```
or if you want rewrite nginx.conf you can type something like that:

```
docker exec -it -d -v example.com.conf:/etc/nginx/conf.d/example.com.conf \
    -v nginx.conf:/etc/nginx/nginx.conf \
    -p "80:80" -p 443:443 axizdkr/tengine
```

[example of conf see on github](https://github.com/Axizdkr/tengine)

## docker-compose 部署

```
version: '3'

services:
  tengine:
    image: harbor.dayuan1997.com/devops/tengine:v2.4.0-nginx-module-vts
    hostname: tengine
    container_name: tengine
    ports:
      - "80:80"
      - "443:443"
      - "9112:9112"
    volumes:
      - /data/tengine/conf.d/:/etc/nginx/conf.d/
      - /data/tengine/devops.d/:/etc/nginx/devops.d/
      - /data/tengine/nginx.conf:/etc/nginx/nginx.conf
      - /data/tengine/certs/:/etc/nginx/certs/
      - /data/tengine/logs/:/var/log/nginx/
      # 挂载业务前端 html
      - /data/devops/xxxx/:/data/xxxx/
    restart: always
```

### nginx.conf 配置

[nginx](./nginx.conf)

### 反向代理配置

```
upstream  test-k8s-t1 {
    server 10.0.0.101:20080  weight=1  max_fails=2  fail_timeout=3;
    server 10.0.0.102:20080  weight=1  max_fails=2  fail_timeout=3;
    server 10.0.0.103:20080  weight=1  max_fails=2  fail_timeout=3;
    keepalive 150;
}

server {
    listen 80;
    server_name ~^t1-.+\.evescn\.com$;
    access_log  /var/log/nginx/t1-access.log  main;
    #rewrite ^(.*)$ https://${server_name}$1 permanent; 
    rewrite ^(.*)$ https://$host$1 permanent;
}

server {
    listen 443 ssl;
    server_name ~^t1-.+\.evescn\.com$;
    access_log  /var/log/nginx/t1-access.log  main;

    ssl_certificate /etc/nginx/certs/dayuan1997.com.pem;
    ssl_certificate_key /etc/nginx/certs/dayuan1997.com.key;

    location / {
        proxy_pass      http://test-k8s-t1;

        # Proxy headers
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # Proxy timeouts
        proxy_connect_timeout              65s;
        proxy_send_timeout                 65s;
        proxy_read_timeout                 65s;
    }
}
```