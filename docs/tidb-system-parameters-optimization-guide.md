# TiDB 8.5.5 系统参数最佳优化方案
# TiDB 8.5.5 System Parameters Optimization Guide

本文档为TiDB 8.5.5版本提供最佳系统参数优化方案，帮助用户根据不同场景获得最佳性能。

This document provides the best system parameter optimization plan for TiDB 8.5.5 to help users achieve optimal performance in different scenarios.

## 目录 / Table of Contents

1. [服务器配置参数 / Server Configuration Parameters](#server-configuration)
2. [性能相关系统变量 / Performance System Variables](#performance-variables)
3. [内存管理参数 / Memory Management Parameters](#memory-management)
4. [并发和并行参数 / Concurrency and Parallelism Parameters](#concurrency-parallelism)
5. [存储和事务参数 / Storage and Transaction Parameters](#storage-transaction)
6. [查询优化器参数 / Query Optimizer Parameters](#query-optimizer)
7. [场景化配置方案 / Scenario-based Configuration](#scenarios)

---

## <a name="server-configuration"></a>1. 服务器配置参数 / Server Configuration Parameters

### 1.1 基础服务配置 / Basic Service Configuration

```toml
# TiDB Server 端口配置
[server]
host = "0.0.0.0"
port = 4000
status-port = 10080

# 套接字配置 - 生产环境推荐配置
# Socket Configuration - Recommended for Production
socket = ""
```

**优化建议 / Optimization Recommendations:**
- **端口 (Port)**: 使用默认端口4000，除非有特殊网络要求
- **状态端口 (Status Port)**: 保持10080用于监控和诊断
- **主机绑定 (Host Binding)**: 生产环境建议使用具体IP而非0.0.0.0以提高安全性

### 1.2 日志配置 / Log Configuration

```toml
[log]
# 日志级别: debug, info, warn, error, fatal
# Log Level: debug, info, warn, error, fatal
level = "info"

# 日志格式: text, json
# Log Format: text, json  
format = "text"

# 慢查询日志
# Slow Query Log
[log.slow-query]
threshold = 300  # 毫秒 / milliseconds
```

**优化建议 / Optimization Recommendations:**
- **日志级别 (Log Level)**: 
  - 生产环境使用 "info" 或 "warn" 以减少磁盘I/O
  - 调试问题时临时切换到 "debug"
- **慢查询阈值 (Slow Query Threshold)**: 
  - OLTP场景: 100-300ms
  - OLAP场景: 1000-5000ms

---

## <a name="performance-variables"></a>2. 性能相关系统变量 / Performance System Variables

### 2.1 执行器相关 / Executor Related

```sql
-- 分布式执行框架 (推荐开启)
-- Distributed Execution Framework (Recommended to enable)
SET GLOBAL tidb_enable_dist_task = ON;

-- Prepare Plan Cache (生产环境推荐开启)
-- Prepare Plan Cache (Recommended for production)
SET GLOBAL tidb_enable_prepared_plan_cache = ON;
SET GLOBAL tidb_prepared_plan_cache_size = 1000;

-- Non-Prepare Plan Cache (8.0+新特性，推荐开启)
-- Non-Prepare Plan Cache (New in 8.0+, recommended)
SET GLOBAL tidb_enable_non_prepared_plan_cache = ON;
SET GLOBAL tidb_session_plan_cache_size = 100;

-- 索引合并优化 (推荐开启)
-- Index Merge Optimization (Recommended)
SET GLOBAL tidb_enable_index_merge = ON;
```

**优化建议 / Optimization Recommendations:**
- **Plan Cache大小**: 根据应用SQL模式数量调整，建议500-2000
- **索引合并**: 对于有多个索引的查询场景显著提升性能

### 2.2 统计信息配置 / Statistics Configuration

```sql
-- 统计信息版本 (推荐使用Version 2)
-- Statistics Version (Recommend Version 2)
SET GLOBAL tidb_analyze_version = 2;

-- 自动统计信息更新
-- Auto Statistics Update
SET GLOBAL tidb_auto_analyze_ratio = 0.5;
SET GLOBAL tidb_auto_analyze_start_time = '00:00 +0000';
SET GLOBAL tidb_auto_analyze_end_time = '06:00 +0000';

-- 统计信息采样率
-- Statistics Sampling Rate
SET GLOBAL tidb_enable_fast_analyze = OFF;  # 生产环境不推荐快速分析
```

**优化建议 / Optimization Recommendations:**
- **Analyze Version 2**: 提供更准确的统计信息，特别适合有索引的列
- **自动分析比例**: 0.5表示当表数据变化超过50%时自动更新统计信息
- **自动分析时间窗口**: 设置在业务低峰期执行，避免影响在线业务

---

## <a name="memory-management"></a>3. 内存管理参数 / Memory Management Parameters

### 3.1 TiDB Server内存限制 / TiDB Server Memory Limits

```sql
-- 单个查询的内存限制 (推荐1GB-4GB)
-- Memory Limit per Query (Recommend 1GB-4GB)
SET GLOBAL tidb_mem_quota_query = 1073741824;  -- 1GB

-- 整个TiDB Server的内存限制 (推荐设置为系统内存的70-80%)
-- Total TiDB Server Memory Limit (Recommend 70-80% of system memory)
SET GLOBAL tidb_server_memory_limit = "80%";

-- 内存使用告警比例
-- Memory Usage Alarm Ratio
SET GLOBAL tidb_memory_usage_alarm_ratio = 0.8;

-- OOM Action (内存溢出处理策略)
-- OOM Action (Out of Memory handling)
SET GLOBAL tidb_mem_oom_action = "CANCEL";  -- 推荐: CANCEL 或 LOG
```

**优化建议 / Optimization Recommendations:**
- **查询内存配额 (tidb_mem_quota_query)**:
  - OLTP场景: 512MB - 1GB
  - OLAP场景: 2GB - 8GB
  - 复杂分析查询: 8GB+
- **Server内存限制**: 留出20-30%给操作系统和其他进程
- **OOM Action**: 
  - CANCEL: 自动取消超限查询（推荐）
  - LOG: 仅记录日志，可能导致OOM

### 3.2 临时对象内存 / Temporary Object Memory

```sql
-- 临时表内存限制
-- Temporary Table Memory Limit
SET GLOBAL tidb_tmp_table_max_size = 67108864;  -- 64MB

-- Chunk大小 (影响内存使用和性能)
-- Chunk Size (Affects memory usage and performance)
SET GLOBAL tidb_max_chunk_size = 1024;  -- 推荐保持默认
```

---

## <a name="concurrency-parallelism"></a>4. 并发和并行参数 / Concurrency and Parallelism Parameters

### 4.1 查询并行度 / Query Parallelism

```sql
-- 执行器并发度配置
-- Executor Concurrency Configuration

-- Index Lookup并发度 (推荐4-8)
SET GLOBAL tidb_index_lookup_concurrency = 4;

-- Index Lookup Join并发度 (推荐4-8)
SET GLOBAL tidb_index_lookup_join_concurrency = 4;

-- Hash Join并发度 (推荐4-8)
SET GLOBAL tidb_hash_join_concurrency = 5;

-- Projection并发度 (推荐4-8)
SET GLOBAL tidb_projection_concurrency = 4;

-- HashAgg并发度
-- HashAgg Concurrency
SET GLOBAL tidb_hashagg_partial_concurrency = 4;
SET GLOBAL tidb_hashagg_final_concurrency = 4;

-- Window函数并发度
-- Window Function Concurrency
SET GLOBAL tidb_window_concurrency = 4;

-- Distsql扫描并发度 (推荐15-25，取决于TiKV实例数)
-- Distsql Scan Concurrency (Recommend 15-25, depends on TiKV instances)
SET GLOBAL tidb_distsql_scan_concurrency = 15;
```

**优化建议 / Optimization Recommendations:**
- **总体原则**: 总并发度不应超过 CPU核心数 * 2
- **OLTP场景**: 较低并发度(4-8)，减少上下文切换
- **OLAP场景**: 较高并发度(8-16)，提高吞吐量
- **混合场景**: 中等并发度(4-8)，平衡性能和资源

### 4.2 DDL并发 / DDL Concurrency

```sql
-- DDL Worker线程数 (推荐2-4)
-- DDL Worker Thread Count (Recommend 2-4)
SET GLOBAL tidb_ddl_reorg_worker_cnt = 4;

-- DDL Reorg批量大小
-- DDL Reorg Batch Size
SET GLOBAL tidb_ddl_reorg_batch_size = 256;

-- DDL并发执行优先级
-- DDL Concurrent Execution Priority
SET GLOBAL tidb_ddl_reorg_priority = 'PRIORITY_LOW';
```

### 4.3 事务并发 / Transaction Concurrency

```sql
-- 事务重试限制
-- Transaction Retry Limit
SET GLOBAL tidb_retry_limit = 10;

-- 悲观事务锁等待超时 (秒)
-- Pessimistic Transaction Lock Wait Timeout (seconds)
SET GLOBAL innodb_lock_wait_timeout = 50;

-- Commit重试
-- Commit Retry
SET GLOBAL tidb_disable_txn_auto_retry = OFF;
```

---

## <a name="storage-transaction"></a>5. 存储和事务参数 / Storage and Transaction Parameters

### 5.1 事务配置 / Transaction Configuration

```sql
-- 事务大小限制 (推荐100MB)
-- Transaction Size Limit (Recommend 100MB)
SET GLOBAL tidb_txn_total_size_limit = 104857600;  -- 100MB

-- 事务条目大小限制
-- Transaction Entry Size Limit
SET GLOBAL tidb_txn_entry_size_limit = 6291456;  -- 6MB

-- 悲观事务模式 (8.0+推荐默认使用)
-- Pessimistic Transaction Mode (Recommend as default in 8.0+)
SET GLOBAL tidb_txn_mode = 'pessimistic';

-- Pipelined DML (8.0+新特性，提升写入性能)
-- Pipelined DML (New in 8.0+, improves write performance)
SET GLOBAL tidb_enable_pipelined_dml = ON;
```

**优化建议 / Optimization Recommendations:**
- **悲观事务**: 适合高冲突场景，减少重试
- **乐观事务**: 适合低冲突场景，提高并发
- **Pipelined DML**: 批量写入场景性能提升20-30%

### 5.2 存储优化 / Storage Optimization

```sql
-- 使用异步提交 (推荐开启)
-- Use Async Commit (Recommend to enable)
SET GLOBAL tidb_enable_async_commit = ON;

-- 使用1PC优化 (推荐开启)
-- Use 1PC Optimization (Recommend to enable)
SET GLOBAL tidb_enable_1pc = ON;

-- 批量DML优化
-- Batch DML Optimization
SET GLOBAL tidb_batch_commit = ON;
SET GLOBAL tidb_batch_delete = ON;
SET GLOBAL tidb_batch_insert = ON;

-- DML批量大小
-- DML Batch Size
SET GLOBAL tidb_dml_batch_size = 20000;
```

---

## <a name="query-optimizer"></a>6. 查询优化器参数 / Query Optimizer Parameters

### 6.1 优化器配置 / Optimizer Configuration

```sql
-- Cost Model版本 (推荐Version 2)
-- Cost Model Version (Recommend Version 2)
SET GLOBAL tidb_cost_model_version = 2;

-- 启用新的行格式
-- Enable New Row Format
SET GLOBAL tidb_enable_new_row_format = ON;

-- 优化器修正控制
-- Optimizer Fix Control
SET GLOBAL tidb_enable_pseudo_for_outdated_stats = OFF;

-- Join重排序算法
-- Join Reorder Algorithm
SET GLOBAL tidb_opt_join_reorder_threshold = 6;

-- 启用外连接重排序
-- Enable Outer Join Reorder
SET GLOBAL tidb_enable_outer_join_reorder = ON;
```

### 6.2 谓词下推 / Predicate Pushdown

```sql
-- 启用聚合下推
-- Enable Aggregate Pushdown
SET GLOBAL tidb_opt_agg_push_down = ON;

-- 启用投影消除
-- Enable Projection Elimination
SET GLOBAL tidb_projection_concurrency = 4;

-- 启用谓词下推到TiKV
-- Enable Predicate Pushdown to TiKV
SET GLOBAL tidb_allow_mpp = ON;
```

### 6.3 MPP执行引擎 / MPP Execution Engine

```sql
-- 启用MPP模式 (推荐TiFlash场景)
-- Enable MPP Mode (Recommend for TiFlash scenarios)
SET GLOBAL tidb_allow_mpp = ON;

-- 强制MPP执行
-- Enforce MPP Execution
SET GLOBAL tidb_enforce_mpp = OFF;  -- 按需开启

-- MPP执行模式
-- MPP Execution Mode
SET GLOBAL tidb_broadcast_join_threshold_size = 104857600;  -- 100MB
SET GLOBAL tidb_broadcast_join_threshold_count = 10240;

-- MPP版本
-- MPP Version
SET GLOBAL mpp_version = 'UNSPECIFIED';  -- 自动选择最优版本
```

---

## <a name="scenarios"></a>7. 场景化配置方案 / Scenario-based Configuration

### 7.1 OLTP场景 (高并发事务处理) / OLTP Scenario (High Concurrency Transaction Processing)

适用于：电商、支付、社交等高并发在线交易系统

**核心配置:**

```sql
-- 内存配置
SET GLOBAL tidb_mem_quota_query = 536870912;  -- 512MB
SET GLOBAL tidb_server_memory_limit = "75%";

-- 并发配置 (保守策略)
SET GLOBAL tidb_index_lookup_concurrency = 4;
SET GLOBAL tidb_hash_join_concurrency = 4;
SET GLOBAL tidb_distsql_scan_concurrency = 15;

-- 事务配置
SET GLOBAL tidb_txn_mode = 'pessimistic';
SET GLOBAL tidb_enable_async_commit = ON;
SET GLOBAL tidb_enable_1pc = ON;

-- 缓存配置
SET GLOBAL tidb_enable_prepared_plan_cache = ON;
SET GLOBAL tidb_prepared_plan_cache_size = 1500;

-- 统计信息
SET GLOBAL tidb_analyze_version = 2;
SET GLOBAL tidb_auto_analyze_ratio = 0.3;  -- 更频繁的更新

-- 优化器
SET GLOBAL tidb_cost_model_version = 2;
```

**性能指标预期:**
- QPS: 10,000 - 100,000+
- 延迟: P95 < 50ms, P99 < 100ms
- 并发连接: 1,000 - 10,000

### 7.2 OLAP场景 (复杂分析查询) / OLAP Scenario (Complex Analytical Queries)

适用于：数据仓库、BI报表、大数据分析

**核心配置:**

```sql
-- 内存配置 (更大的内存配额)
SET GLOBAL tidb_mem_quota_query = 8589934592;  -- 8GB
SET GLOBAL tidb_server_memory_limit = "80%";

-- 并发配置 (激进策略)
SET GLOBAL tidb_index_lookup_concurrency = 8;
SET GLOBAL tidb_hash_join_concurrency = 8;
SET GLOBAL tidb_distsql_scan_concurrency = 25;
SET GLOBAL tidb_hashagg_partial_concurrency = 8;
SET GLOBAL tidb_hashagg_final_concurrency = 8;

-- MPP配置 (启用TiFlash)
SET GLOBAL tidb_allow_mpp = ON;
SET GLOBAL tidb_enforce_mpp = ON;

-- 事务配置
SET GLOBAL tidb_txn_mode = 'optimistic';

-- 优化器
SET GLOBAL tidb_cost_model_version = 2;
SET GLOBAL tidb_opt_agg_push_down = ON;
SET GLOBAL tidb_enable_outer_join_reorder = ON;

-- 统计信息
SET GLOBAL tidb_analyze_version = 2;
SET GLOBAL tidb_auto_analyze_ratio = 0.7;  -- 较低频率
```

**性能指标预期:**
- 查询响应时间: 秒级到分钟级
- 吞吐量: TB级数据扫描能力
- 并发查询: 10 - 100

### 7.3 HTAP混合场景 / HTAP Mixed Scenario

适用于：需要同时支持在线交易和实时分析

**核心配置:**

```sql
-- 内存配置 (平衡策略)
SET GLOBAL tidb_mem_quota_query = 2147483648;  -- 2GB
SET GLOBAL tidb_server_memory_limit = "75%";

-- 并发配置 (中等策略)
SET GLOBAL tidb_index_lookup_concurrency = 6;
SET GLOBAL tidb_hash_join_concurrency = 6;
SET GLOBAL tidb_distsql_scan_concurrency = 20;

-- MPP配置 (选择性启用)
SET GLOBAL tidb_allow_mpp = ON;
SET GLOBAL tidb_enforce_mpp = OFF;  -- 让优化器自动选择

-- 事务配置
SET GLOBAL tidb_txn_mode = 'pessimistic';
SET GLOBAL tidb_enable_async_commit = ON;
SET GLOBAL tidb_enable_1pc = ON;

-- 缓存配置
SET GLOBAL tidb_enable_prepared_plan_cache = ON;
SET GLOBAL tidb_enable_non_prepared_plan_cache = ON;
SET GLOBAL tidb_prepared_plan_cache_size = 1000;
SET GLOBAL tidb_session_plan_cache_size = 100;

-- 优化器
SET GLOBAL tidb_cost_model_version = 2;

-- 统计信息
SET GLOBAL tidb_analyze_version = 2;
SET GLOBAL tidb_auto_analyze_ratio = 0.5;
```

**性能指标预期:**
- OLTP性能: P95延迟 < 100ms
- OLAP性能: 复杂查询秒级响应
- 资源隔离: 通过Resource Control实现

### 7.4 小规模部署 / Small-scale Deployment

适用于：开发测试环境、小型应用

**核心配置:**

```sql
-- 内存配置 (保守策略)
SET GLOBAL tidb_mem_quota_query = 268435456;  -- 256MB
SET GLOBAL tidb_server_memory_limit = "60%";

-- 并发配置 (最小化)
SET GLOBAL tidb_index_lookup_concurrency = 2;
SET GLOBAL tidb_hash_join_concurrency = 2;
SET GLOBAL tidb_distsql_scan_concurrency = 10;

-- DDL配置
SET GLOBAL tidb_ddl_reorg_worker_cnt = 2;

-- 缓存配置 (较小)
SET GLOBAL tidb_prepared_plan_cache_size = 500;

-- 统计信息
SET GLOBAL tidb_analyze_version = 2;
SET GLOBAL tidb_auto_analyze_ratio = 0.5;
```

---

## 8. 监控和调优建议 / Monitoring and Tuning Recommendations

### 8.1 关键性能指标 / Key Performance Metrics

**监控项目:**
1. **查询延迟 (Query Latency)**: 
   - P99延迟 < 100ms (OLTP)
   - 平均延迟 < 20ms (OLTP)

2. **吞吐量 (Throughput)**:
   - QPS (Queries Per Second)
   - TPS (Transactions Per Second)

3. **资源使用 (Resource Usage)**:
   - CPU使用率 < 70%
   - 内存使用率 < 80%
   - 网络带宽使用

4. **连接数 (Connections)**:
   - 活跃连接数
   - 总连接数

### 8.2 调优流程 / Tuning Process

1. **建立性能基线 (Establish Performance Baseline)**
   - 记录当前系统配置和性能指标
   - 识别性能瓶颈

2. **识别瓶颈 (Identify Bottlenecks)**
   - CPU瓶颈: 增加并发度或优化SQL
   - 内存瓶颈: 调整内存配额或优化查询
   - I/O瓶颈: 优化索引或扩容存储
   - 网络瓶颈: 优化数据传输或升级网络

3. **逐步调整 (Incremental Tuning)**
   - 每次只调整一个参数
   - 观察调整后的性能变化
   - 记录调整前后的对比数据

4. **持续监控 (Continuous Monitoring)**
   - 使用Grafana+Prometheus监控集群
   - 定期审查慢查询日志
   - 关注系统告警

### 8.3 最佳实践 / Best Practices

1. **参数调整原则**:
   - 先易后难：从影响大的参数开始
   - 循序渐进：避免激进的大幅度调整
   - 测试验证：在测试环境先验证效果

2. **容量规划**:
   - 预留30%的资源余量
   - 根据业务增长定期评估
   - 提前规划扩容策略

3. **高可用配置**:
   - 至少3个TiDB节点
   - 至少3个PD节点  
   - TiKV副本数设置为3

4. **备份策略**:
   - 定期全量备份（每日）
   - 增量备份（每小时）
   - 定期演练恢复流程

---

## 9. 配置模板 / Configuration Templates

### 9.1 生产环境配置文件模板 / Production Configuration Template

```toml
# TiDB配置文件 - 生产环境推荐配置
# TiDB Configuration - Production Recommended

# 服务器配置
[server]
host = "0.0.0.0"
port = 4000
status-port = 10080
max-server-connections = 4000
grpc-keepalive-time = 10
grpc-keepalive-timeout = 3

# 日志配置
[log]
level = "info"
format = "text"
enable-timestamp = true
enable-slow-log = true

[log.slow-query]
threshold = 300
enable-slow-log = true

[log.file]
max-size = 300
max-days = 3
max-backups = 3

# 性能配置
[performance]
max-procs = 0  # 0表示使用所有CPU核心
server-memory-quota = 0  # 通过系统变量设置
txn-local-latches.enabled = false
tcp-keep-alive = true
tcp-no-delay = true
cross-join = true
stats-lease = "3s"
pseudo-estimate-ratio = 0.8
force-priority = "NO_PRIORITY"

# Prepared Plan Cache
[prepared-plan-cache]
enabled = true
capacity = 1000
memory-guard-ratio = 0.1

# OpenTracing配置
[opentracing]
enable = false

# 监控配置
[status]
report-status = true
record-db-qps = true

# 事务配置
[pessimistic-txn]
max-retry-count = 256
deadlock-history-capacity = 10

# 资源控制
[instance]
tidb_enable_resource_control = true
```

### 9.2 初始化SQL脚本 / Initialization SQL Script

```sql
-- TiDB 8.5.5 优化参数初始化脚本
-- TiDB 8.5.5 Optimization Parameters Initialization Script

-- ========================================
-- 性能优化参数
-- Performance Optimization Parameters
-- ========================================

-- Plan Cache
SET GLOBAL tidb_enable_prepared_plan_cache = ON;
SET GLOBAL tidb_prepared_plan_cache_size = 1000;
SET GLOBAL tidb_enable_non_prepared_plan_cache = ON;
SET GLOBAL tidb_session_plan_cache_size = 100;

-- 内存管理
SET GLOBAL tidb_mem_quota_query = 1073741824;  -- 1GB
SET GLOBAL tidb_server_memory_limit = '75%';
SET GLOBAL tidb_memory_usage_alarm_ratio = 0.8;
SET GLOBAL tidb_mem_oom_action = 'CANCEL';

-- 并发配置
SET GLOBAL tidb_index_lookup_concurrency = 4;
SET GLOBAL tidb_hash_join_concurrency = 5;
SET GLOBAL tidb_distsql_scan_concurrency = 15;
SET GLOBAL tidb_hashagg_partial_concurrency = 4;
SET GLOBAL tidb_hashagg_final_concurrency = 4;

-- 事务优化
SET GLOBAL tidb_txn_mode = 'pessimistic';
SET GLOBAL tidb_enable_async_commit = ON;
SET GLOBAL tidb_enable_1pc = ON;
SET GLOBAL tidb_enable_pipelined_dml = ON;

-- 统计信息
SET GLOBAL tidb_analyze_version = 2;
SET GLOBAL tidb_auto_analyze_ratio = 0.5;
SET GLOBAL tidb_auto_analyze_start_time = '00:00 +0000';
SET GLOBAL tidb_auto_analyze_end_time = '06:00 +0000';

-- 优化器配置
SET GLOBAL tidb_cost_model_version = 2;
SET GLOBAL tidb_enable_outer_join_reorder = ON;
SET GLOBAL tidb_opt_agg_push_down = ON;
SET GLOBAL tidb_enable_index_merge = ON;

-- DDL配置
SET GLOBAL tidb_ddl_reorg_worker_cnt = 4;
SET GLOBAL tidb_ddl_reorg_batch_size = 256;

-- MPP配置 (如果使用TiFlash)
SET GLOBAL tidb_allow_mpp = ON;
SET GLOBAL tidb_broadcast_join_threshold_size = 104857600;

-- ========================================
-- 验证配置
-- Verify Configuration
-- ========================================

SELECT 
    'tidb_enable_prepared_plan_cache' as variable_name,
    @@global.tidb_enable_prepared_plan_cache as value
UNION ALL
SELECT 'tidb_mem_quota_query', @@global.tidb_mem_quota_query
UNION ALL
SELECT 'tidb_txn_mode', @@global.tidb_txn_mode
UNION ALL
SELECT 'tidb_analyze_version', @@global.tidb_analyze_version
UNION ALL
SELECT 'tidb_cost_model_version', @@global.tidb_cost_model_version;

-- 检查当前配置
SHOW VARIABLES LIKE 'tidb%';
```

---

## 10. 故障排查和常见问题 / Troubleshooting and FAQs

### 10.1 常见性能问题 / Common Performance Issues

**问题1: 查询慢**
- 检查是否缺少索引
- 查看是否需要更新统计信息: `ANALYZE TABLE table_name;`
- 检查慢查询日志
- 考虑增加 `tidb_distsql_scan_concurrency`

**问题2: 内存溢出 (OOM)**
- 降低 `tidb_mem_quota_query`
- 优化查询，减少内存使用
- 增加TiDB Server内存
- 检查是否有大事务

**问题3: 写入性能差**
- 检查是否开启了异步提交: `tidb_enable_async_commit`
- 启用1PC优化: `tidb_enable_1pc`
- 考虑批量写入
- 检查TiKV磁盘性能

**问题4: 连接数过多**
- 调整 `max-server-connections`
- 使用连接池
- 检查是否有连接泄露
- 考虑部署负载均衡

### 10.2 参数调整建议 / Parameter Tuning Recommendations

| 场景 | 建议操作 | 预期效果 |
|------|---------|---------|
| 点查询多 | 启用Plan Cache | 降低延迟30-50% |
| 大表Join | 增加hash_join_concurrency | 提升30-100% |
| 写入多 | 启用async_commit + 1PC | 提升20-40% |
| 分析查询 | 启用MPP + TiFlash | 提升5-10倍 |
| 内存不足 | 降低mem_quota_query | 避免OOM |

---

## 11. 版本升级注意事项 / Version Upgrade Notes

### 从早期版本升级到8.5.5的配置变更:

1. **8.0新特性**:
   - Non-Prepared Plan Cache (建议启用)
   - Pipelined DML (建议启用)
   - 新的Cost Model Version 2

2. **废弃的参数**:
   - 某些早期版本的参数可能已废弃
   - 建议查看官方升级文档

3. **兼容性检查**:
   - 升级前备份配置文件
   - 在测试环境验证配置
   - 逐步应用到生产环境

---

## 12. 参考资源 / References

- [TiDB官方文档](https://docs.pingcap.com/tidb/stable)
- [TiDB系统变量参考](https://docs.pingcap.com/tidb/stable/system-variables)
- [TiDB配置文件参考](https://docs.pingcap.com/tidb/stable/tidb-configuration-file)
- [TiDB性能调优指南](https://docs.pingcap.com/tidb/stable/performance-tuning-overview)

---

## 总结 / Summary

本文档提供了TiDB 8.5.5版本的系统参数优化最佳实践，包括：

This document provides best practices for TiDB 8.5.5 system parameter optimization, including:

1. **通用优化方案**: 适用于大多数场景的推荐配置
2. **场景化配置**: 针对OLTP、OLAP、HTAP不同场景的专门优化
3. **监控和调优**: 持续优化的方法和建议
4. **故障排查**: 常见问题的解决方案

建议用户根据实际业务场景，参考本文档的配置建议，结合监控数据，逐步优化系统参数，以获得最佳性能。

**重要提醒**: 
- 所有配置调整应先在测试环境验证
- 逐步应用到生产环境
- 持续监控系统性能指标
- 定期审查和优化配置

---

**文档版本**: 1.0  
**适用TiDB版本**: 8.5.5 及相近版本  
**最后更新**: 2026-02-07
