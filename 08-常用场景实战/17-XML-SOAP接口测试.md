# 17-XML/SOAP接口测试 📋

> **难度：★★★☆☆** | XPath Extractor + XPath Assertion + XML Schema Assertion + HTTP Request (XML Body)

---

## 1. 场景概述

### 适用场景

- **SOAP Web Service**：传统的 SOAP 协议接口测试
- **XML-RPC**：基于 XML 的 RPC 调用
- **REST XML 接口**：返回 XML 格式的 REST API
- **企业系统集成**：ERP、CRM 等传统企业应用的 XML 接口

### XML vs JSON 在 JMeter 中的差异

| 维度 | JSON | XML |
|------|------|-----|
| **请求体** | `Content-Type: application/json` | `Content-Type: text/xml` 或 `application/soap+xml` |
| **提取器** | JSON Extractor / JSON JMESPath | XPath Extractor / XPath2 Extractor |
| **断言** | JSON Assertion / JMESPath Assertion | XPath Assertion / XML Schema Assertion |
| **命名空间** | 无 | 需要处理 XML 命名空间 |
| **性能** | 解析快 | 解析较慢 |

---

## 2. SOAP 接口测试

### 2.1 SOAP 请求示例

```
Test Plan
├── HTTP Request Defaults
│   ├── Protocol: http
│   ├── Server: www.dneonline.com
│   └── Port: 80
├── HTTP Header Manager
│   ├── Content-Type: text/xml;charset=UTF-8
│   └── SOAPAction: "http://tempuri.org/Add"
├── Thread Group
│   ├── HTTP Request
│   │   ├── Method: POST
│   │   ├── Path: /calculator.asmx
│   │   └── Body Data:
│   │       <soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
│   │         <soap:Body>
│   │           <Add xmlns="http://tempuri.org/">
│   │             <intA>${intA}</intA>
│   │             <intB>${intB}</intB>
│   │           </Add>
│   │         </soap:Body>
│   │       </soap:Envelope>
│   ├── XPath2 Extractor
│   │   └── 提取返回结果中的 AddResult
│   └── XPath2 Assertion
│       └── 验证返回的 AddResult 等于预期值
└── View Results Tree
```

### 2.2 SOAP 关键 Header

```xml
<!-- HTTP Header Manager 配置 -->
Content-Type: text/xml;charset=UTF-8
SOAPAction: "http://tempuri.org/Add"
```

> ⚠️ `SOAPAction` 的值必须与 WSDL 中定义的一致，包括引号。

### 2.3 SOAP 参数化

```xml
<!-- 参数化 SOAP 请求体 -->
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Header>
    <AuthHeader xmlns="http://tempuri.org/">
      <Username>${soapUser}</Username>
      <Password>${soapPass}</Password>
    </AuthHeader>
  </soap:Header>
  <soap:Body>
    <GetCustomer xmlns="http://tempuri.org/">
      <CustomerId>${customerId}</CustomerId>
    </GetCustomer>
  </soap:Body>
</soap:Envelope>
```

---

## 3. XPath 提取器

### 3.1 XPath Extractor 配置

| 配置项 | 值 | 说明 |
|--------|-----|------|
| **Apply to** | Main sample only | 应用范围 |
| **Reference Name** | `addResult` | 存储结果的变量名 |
| **XPath Query** | `//AddResult/text()` | XPath 表达式 |
| **Default Value** | `ERROR` | 未匹配时的默认值 |
| **Use Tidy** | ✅ | 容错处理（对不规范 HTML/XML 有效） |
| **Use Namespaces** | ✅ | 处理 XML 命名空间 |
| **XML Parsing Options** | | |
| ├─ Use Tidy | ✅ | 使用 JTidy 修正不规范的 XML |
| ├─ Quiet | ✅ | 不显示 Tidy 警告 |
| ├─ Report Errors | ✅ | 报告解析错误 |
| ├─ Show Warnings | ☐ | 显示警告 |
| └─ Use Namespace | ✅ | 启用命名空间 |

### 3.2 XPath2 Extractor

XPath2 支持更强大的 XPath 2.0 语法：

| 特性 | XPath Extractor | XPath2 Extractor |
|------|:---:|:---:|
| XPath 1.0 | ✅ | ✅ |
| XPath 2.0 | ❌ | ✅ |
| 命名空间 | 手动配置 | 自动处理 |
| 性能 | 快 | 较慢 |

```xml
<!-- XPath2 Extractor 示例 -->
<XPath2Extractor>
  <stringProp name="names">customerName</stringProp>
  <stringProp name="xpathQueries">
    //ns:Customer/ns:Name/text()
  </stringProp>
  <stringProp name="matchNumbers">1</stringProp>
  <stringProp name="defaultValues">NOT_FOUND</stringProp>
</XPath2Extractor>
```

### 3.3 常见 XPath 表达式

| 表达式 | 说明 | 示例 |
|--------|------|------|
| `//tagName/text()` | 提取标签文本 | `//Name/text()` → "John" |
| `//tagName/@attr` | 提取属性值 | `//Customer/@id` → "123" |
| `//tagName[1]/text()` | 提取第一个匹配 | 取第一个结果 |
| `//tagName[last()]/text()` | 提取最后一个 | 取最后一个结果 |
| `//tagName[@attr='value']/text()` | 条件过滤 | `//User[@role='admin']/Name/text()` |
| `count(//tagName)` | 计数 | `count(//Order)` → "5" |
| `//tagName[position()<=3]` | 前 N 个 | 前 3 个元素 |

---

## 4. XML 断言

### 4.1 XPath Assertion

验证 XML 响应中包含特定节点或值：

```xml
<XPathAssertion>
  <boolProp name="XPath.validating">true</boolProp>
  <boolProp name="XPath.tolerant">true</boolProp>
  <boolProp name="XPath.namespace">true</boolProp>
  <stringProp name="XPath.xpath">//AddResult > 0</stringProp>
</XPathAssertion>
```

### 4.2 XML Schema Assertion

验证 XML 响应是否符合 XSD Schema 定义：

```xml
<XMLSchemaAssertion>
  <stringProp name="xmlschema_assertion_filename">
    ./schemas/calculator_response.xsd
  </stringProp>
</XMLSchemaAssertion>
```

**XSD Schema 示例** (`calculator_response.xsd`)：

```xml
<?xml version="1.0" encoding="UTF-8"?>
<xs:schema xmlns:xs="http://www.w3.org/2001/XMLSchema"
           targetNamespace="http://tempuri.org/">
  <xs:element name="AddResponse">
    <xs:complexType>
      <xs:sequence>
        <xs:element name="AddResult" type="xs:int"/>
      </xs:sequence>
    </xs:complexType>
  </xs:element>
</xs:schema>
```

### 4.3 XML Assertion

验证 XML 格式是否合法（格式良好）：

```xml
<XMLAssertion>
  <!-- 无需配置，自动验证 XML 格式 -->
</XMLAssertion>
```

---

## 5. 命名空间处理

### 5.1 带命名空间的 SOAP 响应

```xml
<soap:Envelope xmlns:soap="http://schemas.xmlsoap.org/soap/envelope/">
  <soap:Body>
    <AddResponse xmlns="http://tempuri.org/">
      <AddResult>30</AddResult>
    </AddResponse>
  </soap:Body>
</soap:Envelope>
```

### 5.2 XPath Extractor 命名空间配置

在 XPath Extractor 中勾选 `Use Namespaces` 后，需要手动映射前缀：

```
Namespace prefix: ns1
Namespace URI: http://tempuri.org/
```

然后 XPath 表达式变为：

```
//ns1:AddResult/text()
```

### 5.3 XPath2 Extractor 命名空间（自动）

XPath2 Extractor 自动从 XML 文档中解析命名空间，无需手动配置前缀映射，直接使用原文档中的前缀：

```
//soap:Body//AddResult/text()
```

---

## 6. 实战场景

### 6.1 SOAP 完整压测脚本

```
Test Plan
├── User Defined Variables
│   ├── wsdl.host = www.dneonline.com
│   └── wsdl.path = /calculator.asmx
├── HTTP Request Defaults
│   ├── Server: ${wsdl.host}
│   └── Protocol: https
├── HTTP Header Manager
│   ├── Content-Type: text/xml;charset=UTF-8
│   └── SOAPAction: ""
├── CSV Data Set Config
│   ├── Filename: calc_data.csv
│   └── Variable Names: intA,intB,expectedResult
├── Thread Group
│   ├── HTTP Request (SOAP Add)
│   │   └── Path: ${wsdl.path}
│   ├── XPath2 Extractor
│   │   ├── Reference Name: actualResult
│   │   └── XPath Query: //AddResult/text()
│   ├── Response Assertion
│   │   └── ${actualResult} == ${expectedResult}
│   └── Duration Assertion
│       └── < 5000ms
└── Aggregate Report
```

### 6.2 XML REST API 测试

```xml
<!-- HTTP Request - XML REST API -->
POST /api/customers
Content-Type: application/xml

<Customer>
  <Name>${customerName}</Name>
  <Email>${customerEmail}</Email>
  <Address>
    <Street>${street}</Street>
    <City>${city}</City>
    <ZipCode>${zipCode}</ZipCode>
  </Address>
</Customer>
```

```xml
<!-- XPath2 Extractor - 从响应提取 ID -->
<XPath2Extractor>
  <stringProp name="names">customerId</stringProp>
  <stringProp name="xpathQueries">//Customer/Id/text()</stringProp>
</XPath2Extractor>
```

### 6.3 XML 响应解析

```groovy
// JSR223 PostProcessor - 复杂 XML 解析
import groovy.xml.XmlSlurper;
import groovy.xml.slurpersupport.GPathResult;

def responseXml = prev.getResponseDataAsString();
if (responseXml) {
    def xml = new XmlSlurper().parseText(responseXml);
    
    // 提取嵌套字段
    def customerId = xml.Body.AddResponse.AddResult.text();
    vars.put("customerId", customerId);
    
    // 遍历列表
    def items = xml.Body.GetOrdersResponse.Orders.Order;
    def totalAmount = 0;
    items.each { order ->
        totalAmount += order.Amount.toBigDecimal();
    }
    vars.put("totalAmount", totalAmount.toString());
    vars.put("orderCount", items.size().toString());
}
```

---

## 7. 关键断言总结

| 断言类型 | XML 适用 | 说明 |
|----------|:---:|------|
| **XML Assertion** | ✅ | 验证 XML 格式良好 |
| **XML Schema Assertion** | ✅ | 验证符合 XSD Schema |
| **XPath Assertion** | ✅ | 验证 XPath 表达式 |
| **XPath2 Assertion** | ✅ | XPath 2.0 版本 |
| **Response Assertion** | ✅ | 包含特定文本 |
| **Duration Assertion** | ✅ | 响应时间 |
| **Size Assertion** | ✅ | 响应大小 |

---

## 8. 常见问题排查

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| **XPath 返回空值** | 命名空间未配置 | 勾选 Use Namespaces 或使用 XPath2 Extractor |
| **XML Schema 断言失败** | XSD 路径错误或 Schema 不匹配 | 检查 XSD 文件路径，确认 namespace 一致 |
| **SOAP 请求返回 500** | SOAPAction 不正确或 Body 格式错误 | 从 WSDL 确认准确的 SOAPAction 和消息格式 |
| **Tidy 解析错误** | XML 不规范 | 勾选 Use Tidy + Quiet |
| **中文乱码** | 编码问题 | Header 设置 `charset=UTF-8`，Body 确保 UTF-8 |
| **响应体被截断** | XML 过大 | 增大 `view.results.tree.max_size` |

---

## 9. 性能优化建议

1. **XPath2 性能**：XPath2 Extractor 比 XPath Extractor 慢，简单场景用 XPath 1.0
2. **避免 `//` 全局搜索**：尽量使用绝对路径 `//soap:Body/ns:AddResponse/ns:AddResult` 替代 `//AddResult`
3. **命名空间缓存**：使用 XPath2 自动命名空间处理，减少配置开销
4. **XML 压缩**：大 XML 响应启用 GZIP 压缩
5. **Schema 验证**：仅在调试时启用 XML Schema Assertion，压测时关闭以提升性能
