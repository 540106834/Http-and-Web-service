让我们把 nginx 的配置旅程布置成一张“码头施工图”，既清晰又顺手✨
下面是 **在 Docker 中部署 nginx:1.24，并加载自定义证书** 的完整方式。

---

## 1. 目录结构规划

给你的 Nginx 一个小港口，让证书、配置各就各位：

```
/opt/nginx/
 ├── certs/
 │    ├── server.crt
 │    └── server.key
 ├── nginx.conf
 └── default.conf   # 可选
```

证书格式不限（PEM/CRT 都行，只要内容正确）。

---

## 2. 写一个轻盈的 nginx.conf

像把守港口的灯塔一样，它指挥 TLS、路由、日志一切活动。

### nginx.conf（推荐）

```nginx
user  nginx;
worker_processes  auto;

events {
    worker_connections 1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    sendfile        on;
    keepalive_timeout  65;

    include /etc/nginx/conf.d/*.conf;
}
```

---

## 3. default.conf 写 HTTPS 服务

放进 `/opt/nginx/default.conf`：

```nginx
server {
    listen       443 ssl;
    server_name  localhost;

    ssl_certificate      /etc/nginx/certs/server.crt;
    ssl_certificate_key  /etc/nginx/certs/server.key;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;

    location / {
        return 200 "Hello HTTPS from Docker Nginx!\n";
    }
}

server {
    listen 80;
    return 301 https://$host$request_uri;
}
```

---

## 4. 用 Docker 启动 nginx 1.24

像推开舱门，直接启航：

```bash
docker run -d \
  --name my-nginx \
  -p 80:80 \
  -p 443:443 \
  -v /opt/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /opt/nginx/default.conf:/etc/nginx/conf.d/default.conf:ro \
  -v /opt/nginx/certs:/etc/nginx/certs:ro \
  nginx:1.24
```

注意：

* `-v certs:/etc/nginx/certs` 用于挂载你的证书
* `-v nginx.conf:/etc/nginx/nginx.conf` 是主配置
* `nginx:1.24` 镜像内已经包含 OpenSSL，不必额外装

---

## 5. 检查配置是否无懈可击

进入容器探查：

```bash
docker exec -it my-nginx nginx -t
```

---

## 6. 浏览器访问

```
https://<你的服务器 IP>
curl --cacert ca.crt https://example.com #自签证书访问
```

如果是自签证书，浏览器会提示“不完全信任”，属于正常现象。
若是你自己的 CA，需要把 RootCA 导入系统或浏览器。

---

## 7. 如果你使用自定义颁发的证书（自建 CA）

只需保证 Nginx 使用的 server.crt 内部已经 **拼好服务端证书 + 中级证书链**：

```
-----BEGIN CERTIFICATE-----
# leaf cert
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
# intermediate
-----END CERTIFICATE-----
```

放一起才避免“链不完整”的错误。

---

## 如果你愿意，我可以再帮你：

🟦 自动生成自建 CA + 服务端证书脚本
🟩 把 chain.pem 合并到 crt 的命令
🟧 整个 HTTPS + Docker + 自动更新证书的部署方案
🟨 或帮你把你的证书文件内容分析一遍（删掉敏感部分就好）

要继续吗？
