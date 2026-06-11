# 08 - CI/CD 集成压测

> **难度：★★★★☆** | **适用版本：JMeter 5.6.3** | **预计阅读：25分钟**

---

## 1. 场景概述

将 JMeter 集成到 CI/CD 流水线中，实现每次部署自动运行性能测试，在性能退化时自动告警或阻断发布。

**你将学到：**
- Jenkins Pipeline 集成 JMeter
- 命令行参数化执行
- Dashboard 自动生成与归档
- Pass/Fail 阈值自动判断
- Docker 容器化 JMeter
- GitHub Actions / GitLab CI 集成示例

---

## 2. Jenkins Pipeline 集成

### 2.1 完整 Pipeline 示例

```groovy
pipeline {
    agent any
    
    environment {
        JMETER_HOME = '/opt/apache-jmeter-5.6.3'
        RESULTS_DIR = "results/${env.BUILD_NUMBER}"
        REPORT_DIR  = "reports/${env.BUILD_NUMBER}"
    }
    
    stages {
        stage('Checkout') {
            steps {
                git url: 'https://github.com/your-org/perf-tests.git',
                    branch: 'main'
            }
        }
        
        stage('Smoke Test') {
            steps {
                sh '''
                    ${JMETER_HOME}/bin/jmeter -n \
                        -t tests/smoke_test.jmx \
                        -Jenv=${ENV} \
                        -Jthreads=5 \
                        -Jduration=60 \
                        -l ${RESULTS_DIR}/smoke_test.csv \
                        -e -o ${REPORT_DIR}/smoke_test
                '''
            }
        }
        
        stage('Performance Test') {
            when {
                expression { env.RUN_PERF == 'true' }
            }
            steps {
                sh '''
                    ${JMETER_HOME}/bin/jmeter -n \
                        -t tests/full_load.jmx \
                        -Jenv=${ENV} \
                        -Jthreads=${THREADS} \
                        -Jduration=${DURATION} \
                        -l ${RESULTS_DIR}/perf_test.csv \
                        -e -o ${REPORT_DIR}/perf_test
                '''
            }
        }
        
        stage('Evaluate Results') {
            steps {
                script {
                    def reportFile = readFile "${REPORT_DIR}/perf_test/statistics.json"
                    def stats = new groovy.json.JsonSlurper().parseText(reportFile)
                    
                    def avgResponseTime = stats.Total.meanResTime
                    def errorRate = stats.Total.errorPct
                    def throughput = stats.Total.throughput
                    
                    echo "平均响应时间: ${avgResponseTime}ms"
                    echo "错误率: ${errorRate}%"
                    echo "吞吐量: ${throughput}/sec"
                    
                    // SLA 检查
                    if (avgResponseTime > 2000) {
                        error "响应时间 SLA 失败: ${avgResponseTime}ms > 2000ms"
                    }
                    if (errorRate > 1.0) {
                        error "错误率 SLA 失败: ${errorRate}% > 1%"
                    }
                    if (throughput < 100) {
                        error "吞吐量 SLA 失败: ${throughput}/sec < 100/sec"
                    }
                }
            }
        }
    }
    
    post {
        always {
            // 归档结果
            archiveArtifacts artifacts: "${RESULTS_DIR}/**/*.csv", allowEmptyArchive: true
            archiveArtifacts artifacts: "${REPORT_DIR}/**/*", allowEmptyArchive: true
            
            // 发布 HTML 报告
            publishHTML(target: [
                reportName: "JMeter Performance Report",
                reportDir: "${REPORT_DIR}/perf_test",
                reportFiles: 'index.html',
                keepAll: true
            ])
        }
        failure {
            // 发送告警
            emailext(
                subject: "[JMeter] 性能测试失败 - Build ${env.BUILD_NUMBER}",
                body: "性能测试未通过 SLA 阈值，请检查报告",
                to: "${ALERT_EMAIL}"
            )
        }
    }
    
    parameters {
        choice(name: 'ENV', choices: ['dev', 'staging', 'prod'], description: '测试环境')
        string(name: 'THREADS', defaultValue: '100', description: '并发线程数')
        string(name: 'DURATION', defaultValue: '300', description: '持续时间(秒)')
        booleanParam(name: 'RUN_PERF', defaultValue: false, description: '是否运行性能测试')
    }
}
```

### 2.2 关键步骤说明

| 阶段 | 说明 |
|------|------|
| **Smoke Test** | 每次构建必跑，5线程×60秒快速验证 |
| **Performance Test** | 可选，参数化线程数和时长 |
| **Evaluate Results** | 解析 Dashboard 的 `statistics.json`，判断 SLA |
| **Post Actions** | 归档结果、发布报告、发送告警 |

### 2.3 statistics.json 关键字段

```json
{
    "Total": {
        "sampleCount": 50000,
        "errorCount": 250,
        "errorPct": 0.5,
        "meanResTime": 245.6,
        "medianResTime": 210.0,
        "minResTime": 45.0,
        "maxResTime": 3500.0,
        "pct1ResTime": 450.0,
        "pct2ResTime": 850.0,
        "pct3ResTime": 1500.0,
        "throughput": 166.7,
        "receivedKBytesPerSec": 1024.5,
        "sentKBytesPerSec": 512.3
    }
}
```

---

## 3. 命令行参数化脚本

### 3.1 通用执行脚本 `run_jmeter.sh`

```bash
#!/bin/bash
set -e

# ===== 参数 =====
TEST_PLAN="${1:-tests/default.jmx}"
THREADS="${2:-50}"
DURATION="${3:-300}"
RAMPUP="${4:-30}"
ENV="${5:-dev}"
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
TEST_NAME=$(basename "$TEST_PLAN" .jmx)

# ===== 目录 =====
RESULTS_DIR="results/${ENV}/${TEST_NAME}"
REPORTS_DIR="reports/${ENV}/${TEST_NAME}"
mkdir -p "$RESULTS_DIR" "$REPORTS_DIR"

RESULT_FILE="${RESULTS_DIR}/${TEST_NAME}_${THREADS}vu_${DURATION}s_${TIMESTAMP}.csv"
REPORT_DIR="${REPORTS_DIR}/${TEST_NAME}_${THREADS}vu_${DURATION}s_${TIMESTAMP}"

# ===== JMeter 路径 =====
JMETER_HOME="${JMETER_HOME:-/opt/apache-jmeter-5.6.3}"
JMETER="${JMETER_HOME}/bin/jmeter"

echo "=========================================="
echo " JMeter 性能测试"
echo "=========================================="
echo " 测试计划: $TEST_PLAN"
echo " 线程数:   $THREADS"
echo " 持续时间: ${DURATION}s"
echo " 预热时间: ${RAMPUP}s"
echo " 环境:     $ENV"
echo " 结果文件: $RESULT_FILE"
echo " 报告目录: $REPORT_DIR"
echo "=========================================="

# ===== 执行 =====
$JMETER -n \
    -t "$TEST_PLAN" \
    -Jenv="$ENV" \
    -Jthreads="$THREADS" \
    -Jduration="$DURATION" \
    -Jrampup="$RAMPUP" \
    -l "$RESULT_FILE" \
    -e -o "$REPORT_DIR" \
    -j "logs/jmeter_${TEST_NAME}_${TIMESTAMP}.log"

# ===== 检查退出码 =====
EXIT_CODE=$?

# ===== 评估结果 =====
if [ -f "$REPORT_DIR/statistics.json" ]; then
    echo ""
    echo "=========================================="
    echo " 测试结果摘要"
    echo "=========================================="
    
    # 需要 jq 工具
    if command -v jq &> /dev/null; then
        echo "采样数:     $(jq '.Total.sampleCount' "$REPORT_DIR/statistics.json")"
        echo "错误率:     $(jq '.Total.errorPct' "$REPORT_DIR/statistics.json")%"
        echo "平均RT:     $(jq '.Total.meanResTime' "$REPORT_DIR/statistics.json")ms"
        echo "P90 RT:     $(jq '.Total.pct1ResTime' "$REPORT_DIR/statistics.json")ms"
        echo "P95 RT:     $(jq '.Total.pct2ResTime' "$REPORT_DIR/statistics.json")ms"
        echo "P99 RT:     $(jq '.Total.pct3ResTime' "$REPORT_DIR/statistics.json")ms"
        echo "吞吐量:     $(jq '.Total.throughput' "$REPORT_DIR/statistics.json")/sec"
        
        # SLA 判断
        ERROR_RATE=$(jq '.Total.errorPct' "$REPORT_DIR/statistics.json")
        AVG_RT=$(jq '.Total.meanResTime' "$REPORT_DIR/statistics.json")
        P99_RT=$(jq '.Total.pct3ResTime' "$REPORT_DIR/statistics.json")
        
        SLA_FAIL=false
        if (( $(echo "$ERROR_RATE > 1.0" | bc -l) )); then
            echo "❌ SLA 失败: 错误率 ${ERROR_RATE}% > 1%"
            SLA_FAIL=true
        fi
        if (( $(echo "$AVG_RT > 2000" | bc -l) )); then
            echo "❌ SLA 失败: 平均响应时间 ${AVG_RT}ms > 2000ms"
            SLA_FAIL=true
        fi
        if (( $(echo "$P99_RT > 5000" | bc -l) )); then
            echo "❌ SLA 失败: P99响应时间 ${P99_RT}ms > 5000ms"
            SLA_FAIL=true
        fi
        
        if [ "$SLA_FAIL" = false ]; then
            echo "✅ SLA 全部通过"
        fi
    else
        echo "（安装 jq 以查看详细结果: brew install jq / apt install jq）"
    fi
fi

exit $EXIT_CODE
```

### 3.2 使用方式

```bash
# 基础用法
./run_jmeter.sh tests/api_test.jmx 100 300 30 staging

# 使用默认值
./run_jmeter.sh tests/api_test.jmx

# CI 中使用
./run_jmeter.sh tests/smoke.jmx 10 60 5 staging || exit 1
```

---

## 4. Docker 容器化 JMeter

### 4.1 Dockerfile

```dockerfile
FROM alpine:3.18

ARG JMETER_VERSION=5.6.3
ARG JMETER_HOME=/opt/apache-jmeter-${JMETER_VERSION}

RUN apk add --no-cache openjdk11-jre curl bash

RUN curl -L https://dlcdn.apache.org/jmeter/binaries/apache-jmeter-${JMETER_VERSION}.tgz \
    | tar xz -C /opt

ENV JMETER_HOME=${JMETER_HOME}
ENV PATH=${PATH}:${JMETER_HOME}/bin

WORKDIR /jmeter

COPY tests/ /jmeter/tests/
COPY testdata/ /jmeter/testdata/
COPY run.sh /jmeter/run.sh

RUN chmod +x /jmeter/run.sh

ENTRYPOINT ["/jmeter/run.sh"]
```

### 4.2 run.sh

```bash
#!/bin/bash
jmeter -n \
    -t /jmeter/tests/${TEST_PLAN:-default.jmx} \
    -Jthreads=${THREADS:-50} \
    -Jduration=${DURATION:-300} \
    -Jenv=${ENV:-dev} \
    -l /jmeter/results/result.csv \
    -e -o /jmeter/results/report \
    -j /jmeter/results/jmeter.log
```

### 4.3 使用

```bash
# 构建
docker build -t jmeter-runner .

# 本地运行
docker run --rm \
    -v $(pwd)/results:/jmeter/results \
    -e TEST_PLAN=api_test.jmx \
    -e THREADS=100 \
    -e DURATION=300 \
    -e ENV=staging \
    jmeter-runner

# Kubernetes Job
kubectl create job jmeter-test \
    --image=jmeter-runner \
    -- env TEST_PLAN=api_test.jmx THREADS=200 DURATION=600 ENV=prod
```

### 4.4 Docker Compose（分布式）

```yaml
version: '3.8'
services:
  jmeter-master:
    image: jmeter-runner
    environment:
      - MODE=master
      - REMOTE_HOSTS=jmeter-slave-1,jmeter-slave-2
    volumes:
      - ./results:/jmeter/results
  
  jmeter-slave-1:
    image: jmeter-runner
    environment:
      - MODE=slave
  
  jmeter-slave-2:
    image: jmeter-runner
    environment:
      - MODE=slave
```

---

## 5. GitHub Actions 集成

```yaml
name: Performance Test

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]
  workflow_dispatch:
    inputs:
      threads:
        description: '并发线程数'
        default: '100'
      duration:
        description: '持续时间(秒)'
        default: '300'

jobs:
  perf-test:
    runs-on: ubuntu-latest
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Setup JMeter
        run: |
          wget -q https://dlcdn.apache.org/jmeter/binaries/apache-jmeter-5.6.3.tgz
          tar xzf apache-jmeter-5.6.3.tgz
          echo "JMETER_HOME=$PWD/apache-jmeter-5.6.3" >> $GITHUB_ENV
      
      - name: Run Smoke Test
        run: |
          ${{ env.JMETER_HOME }}/bin/jmeter -n \
            -t tests/smoke_test.jmx \
            -Jthreads=5 \
            -Jduration=30 \
            -l results/smoke.csv \
            -e -o reports/smoke
      
      - name: Run Performance Test
        if: github.event_name == 'workflow_dispatch' || github.ref == 'refs/heads/main'
        run: |
          ${{ env.JMETER_HOME }}/bin/jmeter -n \
            -t tests/full_load.jmx \
            -Jthreads=${{ github.event.inputs.threads || 100 }} \
            -Jduration=${{ github.event.inputs.duration || 300 }} \
            -l results/perf.csv \
            -e -o reports/perf
      
      - name: Check SLA
        run: |
          python3 scripts/check_sla.py reports/perf/statistics.json
      
      - name: Upload Report
        uses: actions/upload-artifact@v3
        if: always()
        with:
          name: jmeter-reports
          path: reports/
      
      - name: Deploy to GitHub Pages
        if: github.ref == 'refs/heads/main'
        uses: peaceiris/actions-gh-pages@v3
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./reports/perf
```

---

## 6. GitLab CI 集成

```yaml
# .gitlab-ci.yml
stages:
  - smoke
  - performance
  - report

variables:
  JMETER_VERSION: "5.6.3"
  JMETER_HOME: "/opt/jmeter"

.jmeter_setup: &jmeter_setup
  before_script:
    - |
      if [ ! -d "$JMETER_HOME" ]; then
        wget -q https://dlcdn.apache.org/jmeter/binaries/apache-jmeter-${JMETER_VERSION}.tgz
        tar xzf apache-jmeter-${JMETER_VERSION}.tgz -C /opt
        mv /opt/apache-jmeter-${JMETER_VERSION} $JMETER_HOME
      fi

smoke_test:
  stage: smoke
  <<: *jmeter_setup
  script:
    - mkdir -p results reports
    - $JMETER_HOME/bin/jmeter -n -t tests/smoke_test.jmx -Jthreads=5 -Jduration=30 -l results/smoke.csv -e -o reports/smoke
  artifacts:
    paths:
      - reports/smoke
    expire_in: 7 days

performance_test:
  stage: performance
  <<: *jmeter_setup
  only:
    - main
    - /^release\/.*$/
  script:
    - mkdir -p results reports
    - $JMETER_HOME/bin/jmeter -n -t tests/full_load.jmx -Jthreads=${THREADS:-100} -Jduration=${DURATION:-300} -l results/perf.csv -e -o reports/perf
  artifacts:
    paths:
      - results/
      - reports/
    expire_in: 30 days
  after_script:
    - python3 scripts/check_sla.py reports/perf/statistics.json

report:
  stage: report
  only:
    - main
  script:
    - echo "性能测试报告: ${CI_PROJECT_URL}/-/jobs/${CI_JOB_ID}/artifacts/browse/reports/perf/"
```

---

## 7. SLA 检查脚本 `check_sla.py`

```python
#!/usr/bin/env python3
import json
import sys
import os

def check_sla(stats_file):
    """检查 JMeter 统计结果是否满足 SLA"""
    
    if not os.path.exists(stats_file):
        print(f"错误: 统计文件不存在: {stats_file}")
        sys.exit(1)
    
    with open(stats_file) as f:
        stats = json.load(f)
    
    total = stats.get("Total", {})
    
    # SLA 阈值定义
    sla_rules = {
        "errorPct": {"value": total.get("errorPct", 0), "threshold": 1.0, "label": "错误率"},
        "meanResTime": {"value": total.get("meanResTime", 0), "threshold": 2000, "label": "平均响应时间(ms)"},
        "pct3ResTime": {"value": total.get("pct3ResTime", 0), "threshold": 5000, "label": "P99响应时间(ms)"},
        "throughput": {"value": total.get("throughput", 0), "threshold": 100, "label": "吞吐量(/sec)", "direction": "min"},
    }
    
    passed = True
    for key, rule in sla_rules.items():
        value = rule["value"]
        threshold = rule["threshold"]
        label = rule["label"]
        direction = rule.get("direction", "max")
        
        if direction == "max":
            fail = value > threshold
        else:
            fail = value < threshold
        
        status = "❌ 失败" if fail else "✅ 通过"
        direction_symbol = ">" if direction == "max" else "<"
        print(f"{status} | {label}: {value} {direction_symbol} {threshold}")
        
        if fail:
            passed = False
    
    print("")
    if passed:
        print("🎉 所有 SLA 检查通过！")
    else:
        print("⚠️  SLA 检查失败，请检查性能报告！")
        sys.exit(1)

if __name__ == "__main__":
    check_sla(sys.argv[1] if len(sys.argv) > 1 else "reports/perf/statistics.json")
```

---

## 8. 性能测试触发策略

| 策略 | 触发条件 | 测试类型 | 线程数 |
|------|----------|----------|--------|
| **每次提交** | Push/PR | Smoke Test | 5-10 |
| **合并到主分支** | Merge to main | 基准测试 | 50-100 |
| **发布前** | Release branch | 全量压测 | 200-500 |
| **定时任务** | 每日凌晨 | 稳定性测试 | 100 (30分钟) |
| **手动触发** | workflow_dispatch | 自定义 | 可配置 |

---

## 9. 常见问题排查

### 9.1 Jenkins 中 JMeter 找不到 Java

**解决：** 在 Pipeline 中显式设置 JAVA_HOME：

```groovy
environment {
    JAVA_HOME = '/usr/lib/jvm/java-11-openjdk-amd64'
}
```

### 9.2 Docker 容器内存不足

**解决：** 调整 JVM 参数：

```dockerfile
ENV JVM_ARGS="-Xms1g -Xmx4g -XX:MaxMetaspaceSize=256m"
```

### 9.3 CI 中 Dashboard 生成失败

**原因：** 结果文件太少（样本数不足）。

**解决：** 确保至少生成一定量的样本数据，或调整 `jmeter.reportgenerator.overall_granularity` 属性。

### 9.4 归档文件太大

**解决：** 仅归档报告 HTML，不归档原始 CSV：

```groovy
archiveArtifacts artifacts: "${REPORT_DIR}/**/*.html,${REPORT_DIR}/**/*.json"
```

---

> **下一章：** [09-性能基准与容量规划](./09-性能基准与容量规划.md) - 建立性能基准线和容量估算
