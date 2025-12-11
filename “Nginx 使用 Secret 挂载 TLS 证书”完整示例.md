这是一套可直接上手的 **“Nginx 使用 Secret 挂载 TLS 证书”完整示例**。
像给 Nginx 穿上一件闪着银线的加密披风，全部配置从 Secret 到 Deployment 再到 Service 都备齐了。

示例包含：

1. 创建 tls Secret（证书+私钥）
2. Pod/Deployment 挂载 Secret
3. Nginx SSL 配置加载 Secret 文件
4. Service 访问方式
5. 完整文件结构无缺口
6. 整条链路的运行逻辑简明图

---

# ① 创建 TLS Secret（提供 HTTPS 必备的钥匙对）

Kubernetes 需要证书和私钥：

```
tls.crt   ← 公钥证书
tls.key   ← 私钥
```

创建方式：

```
kubectl create secret tls nginx-tls \
  --cert=tls.crt \
  --key=tls.key
```

生成后的 Secret 类型为：

```
type: kubernetes.io/tls
```

---

# ② 编辑 Nginx 配置文件，开启 HTTPS

我们单独写一个 nginx.conf，通过 ConfigMap 注入。

## **nginx-config.yaml**

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nginx-config
data:
  nginx.conf: |
    events {}

    http {
      server {
        listen 443 ssl;

        ssl_certificate     /etc/nginx/tls/tls.crt;
        ssl_certificate_key /etc/nginx/tls/tls.key;

        location / {
          return 200 "Hello HTTPS from Nginx + Secret!";
        }
      }
    }
```

注意路径：

```
/etc/nginx/tls/    ← 这里挂载 TLS Secret
```

---

# ③ 部署 Nginx（挂载 ConfigMap + Secret）

这是整个工程的灵魂部分。

## **deployment.yaml**

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-https
spec:
  replicas: 1
  selector:
    matchLabels:
      app: nginx-https
  template:
    metadata:
      labels:
        app: nginx-https
    spec:
      containers:
      - name: nginx
        image: nginx:1.25
        ports:
        - containerPort: 443
        volumeMounts:
        - name: nginx-config
          mountPath: /etc/nginx/nginx.conf
          subPath: nginx.conf
        - name: tls-secret
          mountPath: /etc/nginx/tls
          readOnly: true

      volumes:
      - name: nginx-config
        configMap:
          name: nginx-config

      - name: tls-secret
        secret:
          secretName: nginx-tls
```

关键点：

| 功能              | 行为             |
| --------------- | -------------- |
| ConfigMap       | 覆盖 Nginx 主配置文件 |
| Secret          | 提供证书文件         |
| tls.crt/tls.key | 被挂成容器内部的两个文件   |

---

# ④ 暴露 Service（HTTPS 端口）

## **service.yaml**

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-https-svc
spec:
  type: ClusterIP
  selector:
    app: nginx-https
  ports:
  - port: 443
    targetPort: 443
    protocol: TCP
```

你也可以改为 `LoadBalancer` 或 `NodePort`。

---

# ⑤ 整套资源应用方式

```
kubectl apply -f nginx-config.yaml
kubectl apply -f deployment.yaml
kubectl apply -f service.yaml
```

---

# ⑥ 验证 HTTPS 是否正常工作

方法：

```
kubectl port-forward svc/nginx-https-svc 8443:443
curl -k https://localhost:8443
```

输出：

```
Hello HTTPS from Nginx + Secret!
```

就成功了。

---

# ⑦ 完整链路图（帮助你快速脑内成像）

```
Secret (tls.crt + tls.key)
         ↓
kubelet 投影成文件
         ↓
/etc/nginx/tls/ (容器)
         ↓
Nginx ssl_certificate
         ↓
Nginx 监听 443
         ↓
Service 暴露 HTTPS
         ↓
客户端 curl / browser
```

像一条丝线从 Secret 延伸到容器内部，再被 Nginx 收进手中，让 TLS 整套仪式顺利点亮。

---

# 需要我给你生成一个带注释的“教学版 YAML 套装”吗？

或者你想继续深入：

* Secret 热更新在 TLS 文件上有效吗？
* 使用 Ingress TLS 更合适还是 Nginx Pod 内 TLS 更合适？
* 怎样用 cert-manager 自动生成证书？

你来挑。
