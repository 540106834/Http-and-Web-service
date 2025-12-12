你遇到的情况其实是 **浏览器在访问某个 URL 时直接下载文件而不是显示页面**，这是由 HTTP 响应头或浏览器行为决定的。我们可以系统地分析原因。

---

# 1️⃣ 主要原因：HTTP 响应头 `Content-Disposition`

如果服务器返回的头里包含：

```
Content-Disposition: attachment; filename="xxx.txt"
```

* **attachment** → 浏览器会直接下载，而不是打开
* **filename** → 默认保存文件名

> 不管你访问的是 HTML、TXT、PDF，只要这个头存在，浏览器就会直接下载。

---

# 2️⃣ 服务器 MIME 类型也会影响

浏览器通常根据 **Content-Type** 决定处理方式：

| Content-Type             | 浏览器默认行为             |
| ------------------------ | -------------------------- |
| text/html                | 直接渲染页面               |
| text/plain               | 显示文本                   |
| application/pdf          | 内置 PDF 查看器显示        |
| application/octet-stream | **强制下载**（通用二进制） |

如果你用 Nginx 或 Docker 返回固定文本：

```nginx
return 200 "Hello HTTPS from Docker Nginx!\n";
```

默认 Content-Type 是 **text/plain**，

* Chrome、Edge 会直接显示文本
* IE / Edge 早期版本可能会弹出下载
* 通过 `curl` 可以看到真实响应头

你可以指定 Content-Type 强制浏览器渲染：

```nginx
location / {
    add_header Content-Type text/plain;
    return 200 "Hello HTTPS from Docker Nginx!\n";
}
```

---

# 3️⃣ 浏览器或 Windows 行为

* **Windows 上的 IE/Edge/Chrome** 对未知 MIME 或某些 text/plain 文件，有时会直接下载
* **安全设置**：IE 安全策略或 Chrome “自动下载文件类型” 也会导致直接下载
* **文件扩展名**：如果 URL 以 `.txt`、`.pdf` 结尾且浏览器安全策略启用下载，也可能直接下载

---

# 4️⃣ 解决方法

### 方法 A：修改 Nginx 响应头

如果你希望 **显示在浏览器而不是下载**：

```nginx
location / {
    default_type text/plain;   # 或 text/html
    return 200 "Hello HTTPS from Docker Nginx!\n";
}
```

* `text/html` → 浏览器会当 HTML 渲染
* `text/plain` → 浏览器会显示文本，不下载

---

### 方法 B：检查是否有 Content-Disposition

确保没有类似：

```nginx
add_header Content-Disposition "attachment; filename=hello.txt";
```

删除即可。

---

### 方法 C：浏览器端设置（Windows）

* Chrome / Edge: 设置 → 下载 → 关闭“自动下载”
* IE / Edge：Internet Options → Security → Custom Level → 文件下载启用

---

# 🔹 总结

Windows 浏览器直接下载文件的可能原因：

1. HTTP 响应头带 `Content-Disposition: attachment`
2. MIME 类型是 `application/octet-stream` 或浏览器不认识
3. 浏览器或 Windows 安全策略自动下载某些类型

**最可靠的解决方案**是 **修改 Nginx 返回的 Content-Type** 或 **不要使用 attachment**。

---

我可以帮你改写你的 Docker Nginx 配置，让 **访问根路径既能 HTTPS 正常显示文本，又能显示在浏览器里而不是下载**。

你希望我直接写这个配置吗？