下面给你一套 **一键生成自建 CA + 服务端证书** 的脚本，像是一台安静运转的小型“铸证工坊”。
脚本使用 OpenSSL，目录结构清晰，输出内容可直接给 Nginx 使用。

所有命令可以放进 `gen-cert.sh` 直接运行。

---

# 🟦 一键脚本：生成自建 CA + 服务端证书

```bash
#!/bin/bash

set -e

BASE_DIR=certs
CA_DIR=$BASE_DIR/ca
SERVER_DIR=$BASE_DIR/server

mkdir -p $CA_DIR $SERVER_DIR

echo "==> 1. 生成 CA 私钥"
openssl genrsa -out $CA_DIR/ca.key 4096

echo "==> 2. 生成 CA 根证书（有效期 10 年）"
openssl req -x509 -new -nodes \
  -key $CA_DIR/ca.key \
  -sha256 \
  -days 3650 \
  -subj "/C=CN/ST=DevOps/L=Local/O=MyCA/OU=RootCA/CN=My-Root-CA" \
  -out $CA_DIR/ca.crt

echo "==> 3. 生成服务器私钥"
openssl genrsa -out $SERVER_DIR/server.key 2048

echo "==> 4. 生成服务器 CSR"
openssl req -new \
  -key $SERVER_DIR/server.key \
  -subj "/C=CN/ST=DevOps/L=Local/O=MyServer/OU=IT/CN=localhost" \
  -out $SERVER_DIR/server.csr

echo "==> 5. 生成服务器扩展配置 ext 文件"
cat > $SERVER_DIR/server.ext <<EOF
authorityKeyIdentifier=keyid,issuer
basicConstraints=CA:FALSE
keyUsage = digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = localhost
DNS.2 = 127.0.0.1
IP.1 = 127.0.0.1
EOF

echo "==> 6. 用 CA 签发服务端证书"
openssl x509 -req \
  -in $SERVER_DIR/server.csr \
  -CA $CA_DIR/ca.crt \
  -CAkey $CA_DIR/ca.key \
  -CAcreateserial \
  -out $SERVER_DIR/server.crt \
  -days 825 \
  -sha256 \
  -extfile $SERVER_DIR/server.ext

echo "==> 7. 拼接 server.crt + CA 链（可选）"
cat $SERVER_DIR/server.crt $CA_DIR/ca.crt > $SERVER_DIR/server.fullchain.crt

echo ""
echo "==> 全部完成！输出结构如下："
tree $BASE_DIR

echo ""
echo "==> Nginx 配置参考："
echo "ssl_certificate      /etc/nginx/certs/server.fullchain.crt"
echo "ssl_certificate_key  /etc/nginx/certs/server.key"
```

---

# 🟩 输出目录结构（执行完自动出现）

```
certs/
 ├── ca/
 │    ├── ca.key               # CA 私钥
 │    ├── ca.crt               # 根证书
 │    └── ca.srl               # 序列号
 └── server/
      ├── server.key           # 服务端私钥
      ├── server.csr           # 证书签发请求
      ├── server.crt           # 服务端证书
      ├── server.ext           # 扩展字段 (SAN 等)
      └── server.fullchain.crt # 服务端证书 + CA 链
```

---

# 🟧 Nginx 配置（直接可用）

```nginx
ssl_certificate      /etc/nginx/certs/server.fullchain.crt;
ssl_certificate_key  /etc/nginx/certs/server.key;
```

---

# 🟨 可选：添加 SAN（域名/IP）

直接修改 `server.ext` 里面的：

```
DNS.1 = example.com
DNS.2 = *.example.com
IP.1 = 10.0.0.10
```

---

# 🟪 想进一步提升吗？

我可以继续帮你生成：

* 自动更新脚本
* 带中级 CA（Intermediate CA）的完整 PKI（更真实）
* Docker + Nginx 一键部署脚本
* 证书自动分发方案
* Kubernetes Secret + Ingress 的 HTTPS 配置

你想扩展到哪一步？
