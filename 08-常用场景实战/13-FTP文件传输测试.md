# 13-FTP文件传输测试 📁

> **难度：★★★☆☆** | FTP Request 采样器 + FTP Request Defaults 配置元素

---

## 1. 场景概述

### 适用场景

- **文件服务器性能测试**：FTP/SFTP 服务器的上传下载吞吐能力
- **批量文件传输**：测试大量小文件或大文件的并发传输性能
- **文件同步系统**：验证文件同步服务的可靠性
- **存储系统评估**：NAS/对象存储的 FTP 网关性能

### FTP Request 支持的模式

| 操作 | JMeter 命令 | 说明 |
|------|-------------|------|
| **get (RETR)** | `get` | 从服务器下载文件 |
| **put (STOR)** | `put` | 上传文件到服务器 |
| **login** | 内置 | 建立 FTP 连接和登录 |

---

## 2. 测试计划构建

### 2.1 FTP 文件下载测试

```
Test Plan
├── FTP Request Defaults              # 全局 FTP 默认配置
│   ├── Server: ftp.example.com
│   ├── Port: 21
│   ├── Username: ${ftpUser}
│   └── Password: ${ftpPass}
├── User Defined Variables
│   ├── ftpUser = testuser
│   ├── ftpPass = testpass
│   └── remoteDir = /data/files/
├── Thread Group (文件下载)
│   ├── CSV Data Set Config (文件名列表)
│   │   ├── Filename: ftp_files.csv
│   │   └── Variable Names: fileName,fileSize
│   ├── FTP Request - 下载文件
│   │   ├── Remote File: ${remoteDir}${fileName}
│   │   ├── Local File: ./downloads/${fileName}
│   │   └── get(RETR) ✅
│   ├── Size Assertion
│   │   └── File Size: ${fileSize} (期望大小)
│   └── View Results Tree
└── Aggregate Report
```

### 2.2 FTP Request 配置详解

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **Server** | `ftp.example.com` | FTP 服务器地址 |
| **Port** | `21` | FTP 端口（SFTP 用 22） |
| **Remote File** | `/data/files/${fileName}` | 远程文件路径（可含变量） |
| **Local File** | `./downloads/${fileName}` | 本地保存路径 |
| **Local File Contents** | `内容字符串` | 上传时的文件内容（与 Local File 二选一） |
| **get(RETR)** | ✅ | 下载模式 |
| **put(STOR)** | ☐ | 上传模式 |
| **Use Binary mode?** | ✅ | 二进制模式（推荐始终开启） |
| **Save File in Response?** | ✅ | 将文件内容保存到响应中 |
| **Username** | `testuser` | 用户名（留空则匿名） |
| **Password** | `testpass` | 密码 |

### 2.3 FTP 上传测试

```
Thread Group (文件上传)
├── FTP Request - 上传文件
│   ├── Remote File: /data/upload/${__UUID}.dat
│   ├── Local File: ./testdata/1MB_testfile.dat
│   ├── get(RETR) ☐
│   ├── put(STOR) ✅
│   └── Use Binary mode? ✅
└── Duration Assertion
    └── 上传超时 > 30000ms 视为失败
```

### 2.4 大文件分块上传模拟

```groovy
// JSR223 PreProcessor - 动态生成不同大小的测试文件
def sizeMB = vars.get("fileSizeMB") as int;
def filePath = "./testdata/generated_${sizeMB}MB.dat";

def file = new File(filePath);
if (!file.exists()) {
    def random = new Random();
    def buffer = new byte[1024 * 1024]; // 1MB buffer
    file.withOutputStream { out ->
        for (int i = 0; i < sizeMB; i++) {
            random.nextBytes(buffer);
            out.write(buffer);
        }
    }
}

vars.put("localFilePath", filePath);
vars.put("remoteFileName", "upload_${sizeMB}MB_${__UUID}.dat");
```

---

## 3. FTP 参数化

### 3.1 CSV 文件列表

```csv
# ftp_files.csv
fileName,fileSize,expectedMd5
report_2024.pdf,2048000,d41d8cd98f00b204e9800998ecf8427e
image_001.jpg,512000,098f6bcd4621d373cade4e832627b4f6
data_dump.csv,10485760,5d41402abc4b2a76b9719d911017c592
```

### 3.2 FTP Request Defaults 复用配置

```
Test Plan
├── FTP Request Defaults              # 全局默认值
│   ├── Server: ${__P(ftp.server,localhost)}
│   ├── Port: ${__P(ftp.port,21)}
│   ├── Username: ${__P(ftp.user,anonymous)}
│   └── Password: ${__P(ftp.pass,)}
├── Thread Group - 下载
│   └── FTP Request (只需配置 Remote/Local File)
├── Thread Group - 上传
│   └── FTP Request (只需配置 Remote/Local File)
└── Thread Group - 混合
    └── ...
```

---

## 4. 常见测试场景

### 4.1 并发下载压力测试

```
Thread Group (100 并发)
├── CSV Data Set Config
│   └── Sharing mode: Current Thread (每个线程独立遍历)
├── Uniform Random Timer (500-2000ms 随机延迟)
├── FTP Request (下载)
├── Response Assertion
│   └── Response Code = 200
└── Duration Assertion
    └── 单个文件下载 < 10000ms
```

### 4.2 文件完整性验证

```groovy
// JSR223 PostProcessor - 验证下载文件完整性
import java.security.MessageDigest;

def localFile = new File(vars.get("localFilePath"));
def expectedMd5 = vars.get("expectedMd5");

if (localFile.exists()) {
    def digest = MessageDigest.getInstance("MD5");
    localFile.eachByte(4096) { byte[] buf, int len ->
        digest.update(buf, 0, len);
    }
    def actualMd5 = digest.digest().encodeHex().toString();
    
    if (actualMd5 != expectedMd5) {
        prev.setSuccessful(false);
        prev.setResponseMessage("MD5 mismatch: expected=${expectedMd5}, actual=${actualMd5}");
    }
    
    vars.put("actualMd5", actualMd5);
} else {
    prev.setSuccessful(false);
    prev.setResponseMessage("File not found: " + localFile.path);
}
```

### 4.3 FTP 连接池压力

```
Thread Group (200 线程，逐步加压)
├── FTP Request (下载)
└── JSR223 PostProcessor
    └── 记录连接建立时间
```

通过逐步增加线程数，观察连接超时比例，找到 FTP 服务器的最大并发连接数。

---

## 5. 关键断言

| 断言类型 | 配置 | 用途 |
|----------|------|------|
| **Response Assertion** | Response Code = `200` 或 `Success` | 操作成功 |
| **Size Assertion** | Full Response: `>= ${fileSize}` | 文件大小正确 |
| **Duration Assertion** | `< ${maxDownloadTime}` | 传输超时检查 |
| **JSR223 Assertion** | MD5 校验 | 文件完整性 |
| **JSON Assertion** | （FTP 不适用） | - |

---

## 6. 命令行执行

```bash
# 基础执行
jmeter -n -t ftp_test.jmx -l results.jtl

# 带参数执行
jmeter -n -t ftp_test.jmx \
  -Jftp.server=192.168.1.100 \
  -Jftp.port=21 \
  -Jftp.user=loadtest \
  -Jftp.pass=test123 \
  -Jthreads=50 \
  -Jduration=300 \
  -l ftp_results.jtl \
  -e -o ftp_report/
```

---

## 7. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **Connection refused** | FTP 服务未启动或端口错误 | 确认 FTP 服务状态和端口 |
| **Login failed** | 用户名/密码错误 | 检查凭据，匿名登录留空 |
| **File not found** | 远程路径错误 | 使用 `pwd` 确认当前目录，用绝对路径 |
| **Passive mode timeout** | 防火墙/NAT 问题 | FTP 默认使用被动模式，检查防火墙策略 |
| **Out of memory** | 大文件加载到内存 | 关闭 `Save File in Response`，使用 Size Assertion 验证 |
| **Binary/ASCII mode 损坏** | 传输模式不正确 | 始终使用 Binary mode |

---

## 8. 性能优化建议

1. **Binary Mode**：始终开启，避免文本模式对二进制文件的损坏
2. **关闭响应保存**：大文件测试时关闭 `Save File in Response` 节省内存
3. **连接复用**：使用 FTP Request Defaults 统一配置
4. **文件大小梯度**：测试 1KB/1MB/100MB/1GB 不同大小的文件
5. **磁盘 IO 监控**：FTP 压测时同时监控服务器的磁盘 IO
