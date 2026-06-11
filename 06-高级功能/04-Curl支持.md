# 04 - Curl 支持

JMeter 支持从 Curl 命令导入和生成测试计划。

---

## 如何输入命令

在 JMeter 中通过菜单使用 Curl 功能：

**Tools → Import from cURL**

直接将 Curl 命令粘贴到弹出的对话框中，JMeter 会解析命令并生成相应的 HTTP Request 采样器。

---

## 支持的 Curl 选项

| Curl 选项 | 说明 | JMeter 映射 |
|-----------|------|------------|
| `-X, --request` | HTTP 方法 | HTTP Request Method |
| `-H, --header` | 请求头 | HTTP Header Manager |
| `-d, --data` | POST 数据 | Body Data |
| `--data-raw` | 原始 POST 数据 | Body Data |
| `--data-binary` | 二进制 POST 数据 | Body Data |
| `-u, --user` | 认证信息 | HTTP Authorization Manager |
| `-b, --cookie` | Cookie | HTTP Cookie Manager |
| `-A, --user-agent` | User-Agent | HTTP Header Manager |
| `-e, --referer` | Referer | HTTP Header Manager |
| `-F, --form` | 表单数据 | Parameters / File Upload |
| `--compressed` | 压缩 | HTTP Header Manager |
| `-k, --insecure` | 跳过 SSL 验证 | HTTP Request 高级选项 |
| `--proxy` | 代理 | 命令行代理参数 |

---

## 不支持的选项

以下 Curl 选项 JMeter **不支持**导入：

| 选项 | 原因 |
|------|------|
| `-o, --output` | 输出到文件（JMeter 监听器处理） |
| `-i, --include` | 包含响应头 |
| `-L, --location` | 自动跟随重定向 |
| `--connect-timeout` | 连接超时 |
| `--max-time` | 最大执行时间 |
| `-s, --silent` | 静默模式 |
| `-v, --verbose` | 详细输出 |

---

## 使用示例

### 基本 GET 请求

```bash
curl https://api.example.com/users
```

JMeter 将生成：
- HTTP Request (GET, `/users`)
- Server: `api.example.com`
- Protocol: `https`

### 带请求头的 POST 请求

```bash
curl -X POST https://api.example.com/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"123456"}'
```

JMeter 将生成：
- HTTP Request (POST, `/login`)
- HTTP Header Manager: `Content-Type: application/json`
- Body Data: `{"username":"admin","password":"123456"}`

### 带认证的请求

```bash
curl -u admin:password https://api.example.com/admin
```

JMeter 将生成：
- HTTP Authorization Manager (Basic Auth)

### 带 Cookie 的请求

```bash
curl -b "session=abc123" https://app.example.com/dashboard
```

JMeter 将生成：
- HTTP Cookie Manager: `session=abc123`

### 文件上传

```bash
curl -F "file=@document.pdf" https://upload.example.com/files
```

JMeter 将生成：
- HTTP Request (POST, multipart/form-data)
- 文件上传参数: `file` → `document.pdf`

---

## 注意事项

1. Curl 导入功能主要用于**快速创建测试计划**，并非所有 Curl 选项都支持
2. 复杂场景建议手动调整生成的测试元素
3. 生成的请求默认使用 HttpClient4 实现
4. 导入后可配合 CSV Data Set Config 进行参数化
