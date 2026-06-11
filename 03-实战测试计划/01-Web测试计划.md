# 01 - Web 测试计划

## 4. 构建 Web 测试计划

本节介绍如何创建一个基本的 Web 测试计划。将创建 5 个用户，向 JMeter 网站的两个页面发送请求，每个用户运行 2 次。总请求数：(5 用户) × (2 请求) × (2 次循环) = **20 个 HTTP 请求**。

使用的元素：Thread Group、HTTP Request、HTTP Request Defaults、Graph Results。

---

## 4.1 添加用户（线程组）

1. 右键点击 Test Plan → **Add → Threads (Users) → Thread Group**
2. 设置线程组属性：
   - **Name**：`JMeter Users`
   - **Number of Threads**：`5`
   - **Ramp-Up Period**：`1` 秒（设为 0 会同时启动所有用户）
   - **Loop Count**：`2`

---

## 4.2 添加默认 HTTP 请求属性

1. 右键点击 Thread Group → **Add → Config Element → HTTP Request Defaults**
2. 设置：
   - **Server Name or IP**：`jmeter.apache.org`
   - 其他字段保留默认值

> HTTP Request Defaults 不会发送请求，它仅定义 HTTP Request 元素使用的默认值。

---

## 4.3 添加 Cookie 支持

几乎所有 Web 测试都应使用 Cookie 支持：

1. 右键点击 Thread Group → **Add → Config Element → HTTP Cookie Manager**
2. 每个线程获得独立的 Cookie，但在所有 HTTP Request 对象间共享

---

## 4.4 添加 HTTP 请求

### 第一个请求（首页）

1. 右键点击 Thread Group → **Add → Sampler → HTTP Request**
2. 设置：
   - **Name**：`Home Page`
   - **Path**：`/`
   - Server Name 已在 HTTP Request Defaults 中配置

### 第二个请求（Changes 页面）

1. 添加另一个 HTTP Request
2. 设置：
   - **Name**：`Changes`
   - **Path**：`/changes.html`

---

## 4.5 添加监听器查看/存储测试结果

1. 右键点击 Thread Group → **Add → Listener → View Results Tree**
2. 再添加一个 Aggregate Report 查看统计信息
3. 设置文件路径保存结果

---

## 4.6 登录网站

对于需要登录的网站：

1. 添加 HTTP Request，方法设为 **POST**
2. 需要了解表单字段名称和目标页面
3. 设置 Path 为提交按钮的目标 URL
4. 点击 Add 按钮两次，输入用户名和密码参数
5. 如果表单包含隐藏字段，也需要添加

**示例配置**：

| 参数 | 说明 |
|------|------|
| Path | `/login` |
| Method | POST |
| username | `testuser` |
| password | `testpass` |

---

## 4.7 同用户 vs 不同用户

在 Thread Group 中，可以选择模拟同一用户多次迭代，还是不同用户各迭代一次：

- 配置 Cookie Manager、Cache Manager、Authorization Manager 的行为
- 可选择在每个管理器内直接清除，或由 Thread Group 控制

### HTTP 请求方法说明

| 方法 | 用途 |
|------|------|
| **GET** | 获取资源 |
| **POST** | 提交表单数据 |
| **PUT** | 更新资源 |
| **DELETE** | 删除资源 |
| **HEAD** | 获取响应头 |
| **OPTIONS** | 查询支持的方法 |
| **PATCH** | 部分更新资源 |
