# 01 - REST API 接口测试

> **难度：★☆☆☆☆** | **适用版本：JMeter 5.6.3** | **预计阅读：15分钟**

---

## 1. 场景概述

REST API 是当前最主流的接口通信方式。本章将带你从零搭建一个完整的 REST API 测试计划，覆盖 CRUD 全流程。

**你将学到：**
- HTTP Request 采样器的完整配置（GET/POST/PUT/DELETE/PATCH）
- JSON 请求体发送与响应提取
- 路径参数与查询参数处理
- HTTP Header Manager 的使用
- HTTP Request Defaults 复用配置
- JSON Extractor 提取响应数据

---

## 2. 测试环境准备

### 2.1 目标 API

我们以 [JSONPlaceholder](https://jsonplaceholder.typicode.com/)（一个免费的假数据 REST API）为例：

| 方法 | 端点 | 说明 |
|------|------|------|
| GET | `/posts` | 获取文章列表 |
| GET | `/posts/1` | 获取单篇文章（路径参数） |
| GET | `/posts?userId=1` | 按用户筛选（查询参数） |
| POST | `/posts` | 创建文章（JSON Body） |
| PUT | `/posts/1` | 更新文章（全量） |
| PATCH | `/posts/1` | 更新文章（部分） |
| DELETE | `/posts/1` | 删除文章 |

### 2.2 测试计划树结构

```
Test Plan: REST API 测试
├── HTTP Request Defaults                          # 公共配置
├── HTTP Header Manager                             # Content-Type 头
├── Thread Group: API 基准测试
│   ├── Simple Controller: GET 请求
│   │   ├── GET - 获取文章列表
│   │   └── GET - 获取单篇文章
│   ├── Simple Controller: POST 请求
│   │   ├── POST - 创建文章
│   │   └── JSON Extractor (提取新文章ID)
│   ├── Simple Controller: PUT/PATCH/DELETE
│   │   ├── PUT - 全量更新
│   │   ├── PATCH - 部分更新
│   │   └── DELETE - 删除文章
│   ├── View Results Tree
│   └── Aggregate Report
```

---

## 3. 步骤详解

### 步骤 1：创建公共配置

#### 1.1 添加 HTTP Request Defaults

右键 Test Plan → Add → Config Element → **HTTP Request Defaults**

| 参数 | 值 | 说明 |
|------|-----|------|
| Protocol | `https` | 使用 HTTPS |
| Server Name | `jsonplaceholder.typicode.com` | 目标服务器 |
| Port | `443` | HTTPS 默认端口 |

> **最佳实践：** 将 Server、Protocol、Port 等公共参数放入 HTTP Request Defaults，避免每个采样器中重复配置。子采样器中留空即可继承。

#### 1.2 添加 HTTP Header Manager

右键 Test Plan → Add → Config Element → **HTTP Header Manager**

| Name | Value |
|------|-------|
| `Content-Type` | `application/json; charset=UTF-8` |

> **注意：** 放在 Test Plan 级别会让所有采样器都带上这个 Header。

---

### 步骤 2：创建线程组

右键 Test Plan → Add → Threads (Users) → **Thread Group**

| 参数 | 值 | 说明 |
|------|-----|------|
| Number of Threads | `1` | 单线程（功能验证） |
| Ramp-up Period | `1` | 1秒内启动 |
| Loop Count | `1` | 执行1次 |

> **说明：** 先用单线程验证功能，确认无误后再加压。

---

### 步骤 3：配置 GET 请求

#### 3.1 获取文章列表

右键 Thread Group → Add → Sampler → **HTTP Request**

| 参数 | 值 |
|------|-----|
| Name | `GET - 获取文章列表` |
| Method | `GET` |
| Path | `/posts` |

**无需 Body Data，无需额外参数。**

#### 3.2 获取单篇文章（路径参数）

复制上一个采样器，修改：

| 参数 | 值 |
|------|-----|
| Name | `GET - 获取单篇文章` |
| Method | `GET` |
| Path | `/posts/1` |

#### 3.3 按用户筛选（查询参数）

复制采样器，修改：

| 参数 | 值 |
|------|-----|
| Name | `GET - 按用户筛选文章` |
| Method | `GET` |
| Path | `/posts` |

切换到 **Parameters** 标签页：

| Name | Value | Encode? | Include Equals? |
|------|-------|---------|-----------------|
| `userId` | `1` | ✓ | ✓ |

> JMeter 会自动将 Parameters 拼接为查询字符串：`/posts?userId=1`

---

### 步骤 4：配置 POST 请求

右键 Thread Group → Add → Sampler → **HTTP Request**

| 参数 | 值 |
|------|-----|
| Name | `POST - 创建文章` |
| Method | `POST` |
| Path | `/posts` |

切换到 **Body Data** 标签页，输入：

```json
{
    "title": "JMeter REST API 测试",
    "body": "这是一篇通过 JMeter 创建的文章",
    "userId": 1
}
```

> **⚠️ 重要：** 一旦切换到 Body Data 模式并离开该节点，将无法切换回 Parameters 标签页，除非先清空 Body Data。

#### 4.1 提取创建的文章 ID

右键 POST 采样器 → Add → Post Processors → **JSON Extractor**

| 参数 | 值 | 说明 |
|------|-----|------|
| Names of created variables | `postId` | 变量名 |
| JSON Path expressions | `$.id` | 提取响应的 `id` 字段 |
| Default Values | `NOT_FOUND` | 提取失败时的默认值 |

**验证提取结果：** 可以在后续采样器的 Path 中使用 `${postId}` 引用。

---

### 步骤 5：配置 PUT/PATCH/DELETE 请求

#### 5.1 PUT - 全量更新

| 参数 | 值 |
|------|-----|
| Name | `PUT - 全量更新文章` |
| Method | `PUT` |
| Path | `/posts/${postId}` |

**Body Data：**

```json
{
    "id": ${postId},
    "title": "更新后的标题",
    "body": "更新后的内容",
    "userId": 1
}
```

#### 5.2 PATCH - 部分更新

| 参数 | 值 |
|------|-----|
| Name | `PATCH - 部分更新文章` |
| Method | `PATCH` |
| Path | `/posts/${postId}` |

**Body Data：**

```json
{
    "title": "仅更新标题"
}
```

> **注意：** Java HTTP 实现不支持 PATCH 方法。需要在 HTTP Request 的 Advanced 标签页中选择 `HttpClient4` 实现。

#### 5.3 DELETE - 删除文章

| 参数 | 值 |
|------|-----|
| Name | `DELETE - 删除文章` |
| Method | `DELETE` |
| Path | `/posts/${postId}` |

**无需 Body Data。**

---

### 步骤 6：添加监听器

#### 6.1 View Results Tree

右键 Thread Group → Add → Listener → **View Results Tree**

> 用于开发调试阶段查看每个请求的 Request/Response 详情。

#### 6.2 Aggregate Report

右键 Thread Group → Add → Listener → **Aggregate Report**

> 用于查看汇总统计：Average、Median、90% Line、Throughput 等。

---

## 4. 完整 REST 方法速查表

| HTTP 方法 | 用途 | Body Data | Path 示例 | 安全幂等 |
|-----------|------|-----------|-----------|----------|
| **GET** | 获取资源 | 不需要 | `/api/users/1` | 是 |
| **POST** | 创建资源 | JSON | `/api/users` | 否 |
| **PUT** | 全量更新 | JSON | `/api/users/1` | 是 |
| **PATCH** | 部分更新 | JSON（仅需改的字段） | `/api/users/1` | 否 |
| **DELETE** | 删除资源 | 不需要 | `/api/users/1` | 是 |
| **HEAD** | 获取响应头 | 不需要 | `/api/users` | 是 |
| **OPTIONS** | 查询支持的方法 | 不需要 | `/api/users` | 是 |

---

## 5. 路径参数 vs 查询参数

| 类型 | 写法 | 使用场景 |
|------|------|----------|
| **路径参数** | `/posts/${postId}` | 资源标识（ID等） |
| **查询参数** | `/posts?userId=1` | 筛选、分页、排序 |

**路径参数**直接在 Path 中用 `${变量}` 引用。**查询参数**在 Parameters 标签页中填写，JMeter 自动拼接。

**带分页的查询参数示例：**

| Name | Value |
|------|-------|
| `_page` | `1` |
| `_limit` | `10` |
| `_sort` | `id` |
| `_order` | `desc` |

实际请求：`/posts?_page=1&_limit=10&_sort=id&_order=desc`

---

## 6. 常见问题排查

### 6.1 中文乱码

**现象：** 响应或请求中中文显示为乱码。

**解决：**
1. HTTP Request 中设置 `Content encoding: UTF-8`
2. HTTP Header Manager 添加 `Content-Type: application/json; charset=UTF-8`
3. 如果使用 CSV 参数化，CSV Data Set Config 中设置 `File encoding: UTF-8`

### 6.2 415 Unsupported Media Type

**现象：** 服务器返回 415 状态码。

**原因：** 缺少 `Content-Type: application/json` 头。

**解决：** 确保 HTTP Header Manager 中包含 Content-Type。

### 6.3 PATCH 方法报错

**现象：** Java 实现不支持 PATCH 方法。

**解决：** 在 HTTP Request 的 Advanced 标签页中，将 Implementation 改为 `HttpClient4`。

### 6.4 变量未替换

**现象：** 请求中显示 `${postId}` 而不是实际值。

**原因：** 变量未被正确定义或作用域不对。

**解决：**
1. 检查 JSON Extractor 是否在正确的位置（必须是被提取采样器的子元素）
2. 检查 JSONPath 表达式是否正确
3. 使用 Debug Sampler + View Results Tree 查看所有变量值

---

## 7. 进阶建议

1. **使用模板变量：** 将 JSON Body 中的值替换为 `${变量}`，配合 CSV Data Set Config 实现数据驱动
2. **添加断言：** 每个采样器添加 JSON Assertion 或 Response Assertion 验证响应
3. **提取关联数据：** POST 创建后提取 ID，PUT/PATCH/DELETE 使用提取的 ID（详见"动态关联与提取"章节）
4. **使用 Transaction Controller：** 将一组关联的 API 调用包裹在事务中，获取端到端响应时间
5. **加压测试：** 功能验证通过后，调整 Thread Group 参数进行负载测试

---

> **下一章：** [02-参数化与数据驱动测试](./02-参数化与数据驱动测试.md) - 了解如何用 CSV 数据驱动你的 API 测试
