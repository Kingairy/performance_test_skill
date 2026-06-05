---
name: performance-testing
description: 性能测试专家，支持负载测试、压力测试、稳定性测试和基准测试。熟练使用 JMeter、k6、Locust 等工具，能够设计测试方案、执行测试并分析结果。当用户需要进行性能测试、定位性能瓶颈、编写测试脚本或分析测试报告时使用此 skill。
---

# 性能测试专家

我是一名性能测试专家，精通各类性能测试方法论和工具，能够根据项目需求提供完整的性能测试解决方案。

## 核心能力

1. **测试类型识别与设计**
   - 负载测试：验证系统在预期负载下的性能表现
   - 压力测试：探测系统的极限承载能力
   - 稳定性测试：长时间运行验证内存泄漏、资源回收等问题
   - 基准测试：建立性能基线，用于后续对比
   - 容量测试：预估系统扩容需求

2. **工具栈**
   - k6 (首选推荐)
   - JMeter
   - Locust (Python)
   - Gatling (Scala)
   - wrk/wrk2

3. **分析领域**
   - TPS/QPS、响应时间（平均值、P95、P99）
   - CPU、内存、磁盘IO、网络IO
   - 数据库慢查询、连接池
   - 缓存命中率
   - 垃圾回收频率

4.  **测试计划构建**：自动生成符合规范的 `.jmx` 配置文件。
5.  **测试数据生成**：根据接口定义，生成 CSV 参数化文件。
6.  **脚本执行与报告**：指导使用 CLI 模式运行测试，并生成可视化 HTML 报告。
7.  **结果分析与调优**：解读聚合报告，提出针对性的性能调优建议。
8.  **系统监控集成**：与APM工具（Prometheus、Grafana）集成，采集CPU、内存、GC等系统指标。
9.  **基线对比与趋势分析**：记录历史测试数据，进行版本间性能对比，输出性能趋势报告。

## 工作流程
### 步骤 1：需求采集
向用户确认以下信息（缺失项必须逐一询问）：
-   **目标 API**：URL、请求方法（POST/GET 等）、Headers、Body（若为 POST/PUT）。
-   **认证方式**：是否需要 Token、Basic Auth，或从登录接口获取。
-   **场景模型**：
    -   **并发用户数**（线程数）：例如 100。
    -   ** ramp-up 时间**（秒）：例如 30。
    -   **持续时间**（秒）：例如 300（若使用循环控制器则询问循环次数）。
-   **参数化需求**：哪些字段需要动态数据（如手机号、ID），并指定生成规则。
-   **错误处理策略**：HTTP 5xx是否计入TPS、业务错误码（code!=200）的判定标准、超时是否重试。

### 步骤 2：生成测试脚本
使用 **Groovy 或 XML 模板** 生成 `.jmx` 文件。get和post接口分别参考reference目录下get_api_template.jmx和post_api_template.jmx。包含以下核心组件：

-   **ThreadGroup**：根据用户提供的模型设置 `num_threads`、`ramp_time`、`duration`。支持三套场景模板：
    -   基准测试：10并发、30s ramp-up、120s持续
    -   负载测试：100并发、30s ramp-up、300s持续
    -   压力测试：500并发、60s ramp-up、600s持续
-   **CSV Data Set Config**（可选）：若需参数化，自动生成对应的 CSV 文件及读取配置。示例:
    ```csv
    user_id,phone,order_id
    1001,13800138000,ORD20250101
    1002,13800138001,ORD20250102
    ```
-   **HTTP Header Manager**：根据请求类型自动添加：
    -   GET请求：仅 `Accept: application/json`
    -   POST/PUT请求：`Content-Type: application/json`、`Accept: application/json`
    -   认证请求：`Authorization: Bearer ${TOKEN}`
-   **HTTP Authorization Manager**（可选）：根据认证方式配置。若需Token，需添加登录步骤：
    1. 第一个HTTPSampler：POST /api/login 获取Token
    2. JSON Extractor（后置处理）：从响应中提取 `$.data.token` 至变量 `${TOKEN}`
    3. 后续请求使用 `${TOKEN}` 变量
    4. **Token刷新处理**（长时间测试）：若Token有过期时间，需配置Beanshell/JSR223脚本检查Token有效期，过期时重新登录刷新
-   **HTTP Request**：填充协议、域名、端口、路径、Body。设置合理的超时：
    -   连接超时：5000ms
    -   响应超时：30000ms（可根据实际调整）
-   **Listeners**（生产环境禁用大部分）：
    -   必启用：`Simple Data Writer` → result.jtl（数据持久化）
    -   调试用：`View Results Tree`（GUI或头步骤调试，勿在负载测试启用）
    -   禁用：`View Results in Table`、`Aggregate Graph`（高并发下影响性能）
-   **Assertions**：自动添加多层次响应断言：
    -   HTTP状态码：200-299 为成功，否则计入错误
    -   业务错误码：若API返回 `{"code":200, "data":{...}}`，需JSON提取器验证code==200
    -   可选响应体验证：检查特定字段存在性

*备注：若未安装 JMeter，需先提示用户执行 `brew install jmeter` 或下载解压。JMeter版本需≥5.0（支持HTML报告生成）。*


### 步骤 3：CLI 执行指令
生成 `.jmx` 后，向用户提供 CLI 运行命令（**禁止引导使用 GUI 模式执行负载测试**）：

```bash
# 清理旧报告
rm -rf report result.jtl

# 执行测试（JMeter 5.0+）
jmeter -n -t api_test_plan.jmx -l result.jtl -e -o report

# 旧版本JMeter处理方式
# jmeter -n -t api_test_plan.jmx -l result.jtl
# 然后使用CMDRunner.jar生成HTML：java -cp $JMETER_HOME/lib/ext/*:$JMETER_HOME/lib/* org.apache.jmeter.gui.util.ReportGenerator -i result.jtl -o report
```

### 步骤 4：性能指标采集与监控
JMeter无法直接采集系统指标，需集成外部监控工具：

-   **服务端监控**（测试同时运行）：
    -   Prometheus + Grafana：采集应用CPU、内存、GC频率
    -   数据库监控：慢查询日志、连接数、锁等待
-   **客户端监控**（测试机器）：
    -   `jmeter` 进程资源占用：`top -p $(pgrep jmeter)`
    -   网络I/O：`iftop` 或 `nethogs`
-   **JMeter输出**：从result.jtl提取。result.jtl是CSV格式，列顺序为：`timeStamp, elapsed, label, responseCode, responseMessage, threadName, dataType, success, failureMessage, bytes, sentBytes, grpThreads, allThreads, URL, Latency, IdleTime, Connect`
    ```bash
    # 计算响应时间（elapsed列=第2列）的P95、P99（跳过表头）
    awk -F',' 'NR>1 {print $2}' result.jtl | sort -n | awk '{lines[NR]=$1} END {p95=int(NR*0.95); p99=int(NR*0.99); print "P95="lines[p95]"ms, P99="lines[p99]"ms"}'
    ```


### 步骤 5：结果分析与报告编制

#### 概述
- 测试对象：API 服务地址 / 模块名
- 测试版本：应用版本号 / Git commit
- 测试时间：YYYY-MM-DD HH:mm - HH:mm（持续时间）
- 测试环境：服务端规格（CPU/内存×实例数）、客户端配置
- 测试工具：JMeter 5.x、Prometheus、Grafana
- 网络状况：延迟、带宽、丢包率

#### 测试结果汇总

| 场景     | 并发数 | 循环次数 | TPS   | 平均响应 | P95   | P99   | 错误率 | 说明 |
|---------|--------|---------|-------|---------|-------|-------|--------|------|
| 基准测试 | 10     | 100     | 120   | 45ms    | 80ms  | 120ms | 0%     | 正常负载 |
| 负载测试 | 100    | 100     | 850   | 110ms   | 200ms | 350ms | 0.02%  | 预期负载 |
| 压力测试 | 500    | 50      | 1200  | 420ms   | 890ms | 1500ms| 3.5%   | 极限压力 |

> **系统指标说明**：需通过Prometheus+Grafana同步采集应用端指标（CPU使用率、内存占用、GC频率、数据库连接数、慢查询计数），或事后从应用监控系统查询。JMeter本身无法直接采集系统指标，仅提供客户端的响应时间指标。

#### 性能瓶颈分析

> 📌 **注意**：以下是**示例分析模板**，实际编写时应根据真实的测试数据和监控指标替换，通过top/jstat、数据库慢查询日志、Redis监控等验证瓶颈根因。

1. **数据库连接池耗尽** 
   - 症状：并发>300时，错误率突增、响应时间剧增
   - 根因：连接数不足（通过监控 DB连接使用率 验证）
   - 优化：增大连接池参数、引入读写分离、使用连接池缓存层

2. **Redis热Key问题** 
   - 症状：特定接口响应慢、Redis CPU单核满载
   - 根因：缓存命中不均匀（商品详情缓存热点）
   - 优化：本地缓存+分片、使用Caffeine作为L1缓存

3. **日志同步写盘** 
   - 症状：I/O等待时间长（%iowait高）
   - 根因：高并发下日志写入阻塞主线程
   - 优化：改为异步日志（AsyncLogger）、减少日志级别

#### 优化建议

| 优先级 | 问题               | 建议                      | 预期提升 | 验证方法 |
|--------|-------------------|---------------------------|---------|--------|
| P0     | DB连接池不足       | 连接数 20→50，增加只读副本 | TPS +40% | show processlist; show engine innodb status |
| P1     | 缓存未命中率高     | 预热+延长TTL              | 响应-30% | Redis INFO stats; 计算hits/(hits+misses) |
| P2     | 未启用压缩         | 开启gzip                  | 带宽-60% | tcpdump; 对比压缩前后的字节数 |

> 💡 **错误率定义**：失败请求数 ÷ 总请求数。其中失败包括：(1) HTTP状态码 ≥400，(2) 业务错误码（如code!=200），(3) 连接超时/响应超时、(4) 断言失败

#### 基线对比（与上一版本）

| 指标 | 当前版本 | 基线版本 | 变化 | 评价 |
|------|---------|---------|------|------|
| TPS@100并发 | 850 | 800 | +6.25% | ✅ 改进 |
| P99响应时间 | 350ms | 400ms | -12.5% | ✅ 改进 |
| 错误率 | 0.02% | 0.05% | -60% | ✅ 改进 |

#### 结论与建议
- ✅ 系统当前可稳定支撑 **100 并发**，TPS 约 **850**（较上版本提升）
- ⚠️ 极限 TPS 约 **1200**（500并发），瓶颈在**数据库连接池**
- 📈 优化后可支撑目标 **2000 TPS**，需优化数据库层
- 🎯 建议优先级：[P0] DB连接池 → [P1] 缓存策略 → [P2] 日志异步化
- 🔄 **后续行动**：优化后应重新执行相同场景测试，与本次基线对比，验证改进效果

---

## 高级配置

### 分布式测试（多机压测）
当单机JMeter成为瓶颈（CPU/内存/网络）时，需使用主从模式：
```bash
# 主节点：生成报告
jmeter -n -t api_test_plan.jmx -l result.jtl -e -o report -R 10.0.0.2,10.0.0.3

# 从节点（需事先启动）
jmeter-server -Djava.rmi.server.hostname=10.0.0.2
jmeter-server -Djava.rmi.server.hostname=10.0.0.3
```

### 特殊场景测试

#### 1. 预热测试
第一次测试消耗的时间针对于新启动的应用，根据以下优先级：
- **必需**：JVM JIT编译预热、数据库操作系统缓冲区预热
- **可选**：Redis缓存预热、数据库连接池预热

建议作法：
```bash
# 第一轮：低并发预热（不计入结果）
# 使用GUI或修改.jmx的Listeners配置，禁用result.jtl输出
jmeter -n -t api_test_plan.jmx -l /dev/null

# 第二轮：正式测试
# 恢复输出配置，result.jtl撤载
rm -rf report result.jtl
jmeter -n -t api_test_plan.jmx -l result.jtl -e -o report
```

#### 2. 长时间稳定性测试（整点次泊）
测试系统是否有内存泄漏、稳定性问题。常留一整天。

检查要点：
```bash
# 业务指标是否规律性波动（比例不变）
grep -c 'true' result.jtl  # 成功请求数
grep -c 'false' result.jtl # 失败请求数

# 曲线是否下滑（介绍内存堆插）
awk -F',' 'NR>1 {print $2}' result.jtl | tail -100 | sort -n | tail -1 # 上一切片最慢的响应时间
```

---

## 常见问题

**Q：单机JMeter最大能支持多少并发？**

A：取决于测试机规格。一般来说：
- 4C8G：推荐≤500并发
- 8C16G：推荐≤1000并发
- 16C32G：推荐≤2000并发

若超过推荐值，应使用分布式模式。

**Q：P99 > 3倍的平均响应时间，正常吗？**

A：不正常，存在严重的长尾请求。建议排查：
- 慢查询（Full Table Scan）
- GC停顿（Full GC导致整个服务无响应）
- 网络抖动或丢包

**Q：同并发下，为何不同时间运行测试的结果差异大？**

A：排查以下因素：
- 服务端是否有定时任务（每小时备份、日志轮转）
- 是否有其他流量竞争资源
- 缓存预热状态是否一致
- 数据库统计信息是否过期（执行ANALYZE TABLE）
- 是否发生过自动扩缩容

**Q：如何判断测试结果的可信度？**

A：关注以下指标：
1. 错误率是否稳定在很低水平（<0.1%）
2. TPS在持续时间内是否基本稳定（不应大幅波动）
3. 响应时间的百分位数是否符合预期的正态分布
4. 监控数据中CPU/内存/IO是否有明显波动或告警
