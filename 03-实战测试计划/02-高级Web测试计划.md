# 02 - 高级 Web 测试计划

## 5. 构建高级 Web 测试计划

### 5.1 使用 URL 重写处理用户会话

如果 Web 应用程序使用 URL 重写（而非 Cookie）来保存会话信息，需要做额外的配置。

**URL 重写示例**：
```
http://example.com/index.html?jsessionid=ABC123
```

#### 配置步骤

1. **添加 HTTP URL Re-writing Modifier**
   - 右键点击 Thread Group → **Add → Pre Processors → HTTP URL Re-writing Modifier**
   - 设置 **Session Argument Name**（如 `jsessionid`）

2. **从响应中提取会话 ID**
   - 使用 **Regular Expression Extractor** 从响应中提取会话 ID
   - 存储为变量供后续请求使用

---

### 5.2 使用 HTTP 头管理器

HTTP Header Manager 用于添加或覆盖 HTTP 请求头。

#### 配置步骤

1. 右键点击 Thread Group → **Add → Config Element → HTTP Header Manager**
2. 添加所需的请求头：

| 常用 Header | 值示例 |
|-------------|--------|
| `User-Agent` | `Mozilla/5.0 (Windows NT 10.0; Win64; x64)` |
| `Accept` | `text/html,application/json` |
| `Content-Type` | `application/json` |
| `Authorization` | `Bearer token123` |
| `Referer` | `http://example.com/previous-page` |

#### 作用域规则

HTTP Header Manager 遵循标准的作用域规则：
- 放在 Thread Group 级别 → 影响所有请求
- 放在 Controller 级别 → 仅影响该 Controller 下的请求
- 放在 Sampler 级别 → 仅影响该采样器

---

### 5.3 HTTP Authorization Manager

用于处理 HTTP 基本认证和摘要认证。

#### 配置参数

| 参数 | 说明 |
|------|------|
| **Base URL** | 需要认证的基础 URL |
| **Username** | 认证用户名 |
| **Password** | 认证密码 |
| **Domain** | NTLM 域名 |
| **Realm** | NTLM Realm |
| **Mechanism** | 认证机制：`BASIC_DIGEST`、`BASIC`、`DIGEST`、`KERBEROS` |

---

### 5.4 HTTP Cache Manager

模拟浏览器缓存行为。

#### 配置选项

| 选项 | 说明 |
|------|------|
| **Clear cache each iteration** | 每次迭代清除缓存 |
| **Use Thread Group configuration** | 由 Thread Group 控制清除 |
| **Max Number of elements** | 缓存最大条目数 |

---

### 5.5 处理文件上传

使用 HTTP Request 上传文件：

1. 设置 Method 为 **POST**
2. 在 **Files Upload** 标签页中：
   - **File Path**：要上传的文件路径
   - **Parameter Name**：表单中的参数名
   - **MIME Type**：文件的 MIME 类型

---

### 5.6 处理文件下载

使用 **Save Responses to a file** 监听器：

1. 添加 **Save Responses to a file** 监听器
2. 配置输出目录和文件名前缀
3. 可选择性保存成功的或失败的响应

---

### 5.7 使用 HTTP(S) Test Script Recorder

录制器是构建测试计划最有效的方式之一。

#### 录制步骤

1. 添加 HTTP(S) Test Script Recorder 到测试计划
2. 配置浏览器代理指向 JMeter（默认 `localhost:8888`）
3. 安装 JMeter 的 CA 证书以录制 HTTPS
4. 在浏览器中执行操作
5. 停止录制，清理和整理录制的请求

#### 过滤配置

| 模式类型 | 示例 | 说明 |
|----------|------|------|
| **包含模式** | `.*\\.jsp` | 仅录制 JSP 请求 |
| **包含模式** | `.*\\.php` | 仅录制 PHP 请求 |
| **排除模式** | `.*\\.gif` | 排除 GIF 图片 |
| **排除模式** | `.*\\.(css\|js\|png\|ico)` | 排除常见静态资源 |
