# 阶段三：性能、并发、同步延迟与数据库性能

铁律：**没有测过的数字一律写"未测试"**。静态分析可以给"预期瓶颈"和"验证方法"，不能给 P95 数字，也不能由压测结果反推生产承载能力。

## 1. 指标口径（必须分开报）

| 层次 | 怎么取 | 说明 |
|---|---|---|
| 服务端处理 | 应用日志/APM 单请求耗时 | 不含网络，含 DB 与外部调用 |
| DB 执行 | 慢查询日志、`EXPLAIN ANALYZE`、ORM 日志计时 | 单条 SQL 耗时 |
| 网络传输 | 客户端计时 - 服务端计时 | 弱网差异大，需注明网络条件 |
| 端到端 | 小程序 `wx.request` 起止 / 浏览器 Performance | 含渲染与第三方 |
| 第三方等待 | 微信支付、短信、OSS 调用耗时 | 需单独统计占比 |

指标：P50 / P95 / P99、max、RPS（或 TPS）、错误率（4xx/5xx/超时分开）、响应体大小、CPU/内存、单请求 SQL 次数、缓存命中率。分接口给出，聚合平均没有意义。

## 2. 核心接口清单（按业务重要性排序）

登录/换取 token、首页聚合、列表（分页/搜索/筛选）、详情、写操作（新增/修改/取消）、管理端列表与统计、订单创建、支付状态查询、支付回调处理、文件上传/下载、导出。项目里没有的标"不适用"。

## 3. 并发与一致性分析

### 场景矩阵（先分析，后（授权后）测试）

| 场景 | 潜在问题 | 判定依据 | 验证方式 |
|---|---|---|---|
| 多人同时查询 | 连接池耗尽、慢查询放大 | 池大小 vs 并发 | 授权后压测 |
| 同一用户重复提交 | 重复订单/重复记录 | 有无幂等键、唯一约束 | 双发请求（隔离环境） |
| 并发更新同一行 | Lost Update | `UPDATE ... SET x=?` 无版本条件 | 两事务交叉执行 |
| 并发扣库存 | 超卖 | 是否 `stock >= n` 条件更新 | 并发下单（Mock 支付） |
| 并发核销券 | 一券多用 | 唯一约束/状态 CAS | 并发核销 |
| 同一订单多次回调 | 重复履约 | 幂等表 + 状态机 | 重放通知（沙箱） |
| 回调与取消并发 | 已取消却已支付/已支付却被取消 | 状态机是否 CAS | 构造交错 |
| 管理员与用户同时改 | 后写覆盖 | 无版本号/无锁 | 交叉写 |
| 批量任务与在线请求 | 锁等待、长事务 | 批大小、事务范围 | 观察锁等待 |

### 要判断的数据库现象（不要机械罗列）

- **Dirty Read**：只在 READ UNCOMMITTED 出现，先确认隔离级别（MySQL 默认 RR，PostgreSQL 默认 RC），不存在就写"不适用"。
- **Non-repeatable Read / Phantom Read**：RR 下 MySQL 用间隙锁抑制幻读但非全部场景；只有代码里"先查后判再写"的模式才真正受影响。
- **Lost Update**：读-改-写在应用层完成且无版本条件时成立，这是最常见的真实问题。
- **Deadlock**：多表更新顺序不一致、批量更新顺序随机时成立；看是否有统一加锁顺序。
- **长事务**：事务内调用微信 API、OSS、发短信、sleep、大批量循环 → 锁持有时间放大。

### 判定要点

- 只有一处写、或 DB 层原子自增，不构成竞态。
- 唯一索引存在时，"重复"退化为报错而非脏数据——这仍算缺陷（体验/事务），但级别不同。
- 乐观锁要连"检查影响行数"一起看，只 `WHERE version=?` 不判 `affectedRows` 等于没做。

## 4. 同步与阻塞

| 栈 | 阻塞迹象 | 搜索 |
|---|---|---|
| Node | async 里用 `*Sync`、CPU 密集未分片 | `readFileSync|writeFileSync|execSync|pbkdf2Sync|bcrypt.hashSync` |
| Node | 未 await 的 Promise（吞异常） | 缺失 `await`、`forEach(async` |
| Python | `async def` 内调用同步 ORM/requests | `requests.get` in async、`Session()` 同步 |
| Java | 线程池过小、`synchronized` 跨请求、阻塞 Future.get | 线程池配置、`@Async` 默认池 |
| 通用 | 外部 API 无超时（默认无限） | `timeout` 缺失、`connectTimeout` 缺省 |
| 通用 | 串行可并发（循环里 await 单条查询） | `for ... await` 包住 DB 调用 |
| 通用 | MQ 消费无幂等、无重试上限、无死信 | 消费者入口 |

## 5. 跨端同步延迟

分三段量化，别混成一句"有延迟"：

1. **落库延迟**：接口返回时是否已提交（有无异步写、有无缓存先写后落库）
2. **缓存传播**：Redis key 何时失效、是否有双删、是否 TTL 兜底、是否有本地缓存
3. **页面可见**：轮询间隔、WebSocket 推送、进入页面刷新、手动下拉

判断标准：资金/库存/订单状态需强一致（读主库或版本校验）；展示类信息（销量、评价数、公告）最终一致可接受。写出"哪个字段属于哪类"。

## 6. 数据库性能

### 静态检查

- N+1：ORM 关联懒加载、循环内单查、列表接口每行再查一次；对比 `include/join/preload` 使用情况
- 缺索引：`WHERE`、`ORDER BY`、`JOIN ON`、`GROUP BY` 列；检查迁移与 `SHOW INDEX`
- 冗余/低效索引：前缀重复、选择性极低（性别、状态）单列索引
- 组合索引顺序：等值列在前、范围列在后；是否覆盖查询列
- 分页：`LIMIT 100000, 20` 大偏移；建议游标分页（`WHERE id > ?`）
- 高成本 COUNT：分页总数用 `COUNT(*)` 全表；可估算或用近似
- 全表扫描：`LIKE '%x%'`、函数包裹列（`DATE(created_at)`）、隐式类型转换
- 事务：范围过大、事务内外部 IO、循环单条 insert（应批量）
- 连接池：max 过小/过大、泄漏（未 release）、连接未复用
- JSON 字段：在 JSON 上过滤排序（无法用索引）

### 验证方法（只读）

```sql
SHOW INDEX FROM orders;
SHOW TABLE STATUS LIKE 'orders';
EXPLAIN SELECT ... ;              -- 看 type/rows/Extra（Using filesort / Using temporary）
EXPLAIN ANALYZE SELECT ... ;      -- MySQL 8.0.18+ / PostgreSQL
SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = DATABASE();
```

行数用 `information_schema.tables.table_rows`（近似）或抽样 `COUNT`，不要在千万级表上直接 `COUNT(*)`。

### 建议写法

每条索引建议写成：`表(列...)` + 支撑的查询 + 依据（EXPLAIN 现状）+ 预期收益 + 写入放大代价 + 回归风险。没有 EXPLAIN 或数据量的"建议加索引"不算结论。

## 7. 分级负载测试计划（默认只出计划）

| 阶段 | 目标 | 并发/速率 | 时长 | 停止条件 |
|---|---|---|---|---|
| 基线 | 单用户功能正确性 + 单请求耗时 | 1 | 每接口 ≥30 次 | 出现 5xx 或耗时异常 |
| 典型负载 | 混合业务比例下稳定运行 | 按测试环境 CPU/池容量推算 | 15–30 min | 错误率 >1% 或 P95 超目标 |
| 并发爬坡 | 找拐点 | 阶梯递增 | 每阶 3–5 min | P95 翻倍或错误率 >2% |
| 突发 | 抗瞬时冲击 | 短时 3–5 倍基线 | 1–2 min | 服务不可用或不恢复 |
| 稳定性 | 内存泄漏、连接泄漏 | 70% 拐点负载 | 2–4 h | 内存持续增长不回落 |
| 故障恢复 | 依赖宕机恢复 | 停 Redis/DB/下游 | 单次 | 不自动恢复即失败 |

前置条件：测试环境独立、数据可重置、支付走 Mock、限流临时放宽需记录、有监控采集（CPU/内存/池/慢查询/QPS）。

k6 骨架（仅示例，参数必须按环境实测调整）：

```js
import http from 'k6/http';
import { check, sleep } from 'k6';

export const options = {
  stages: [
    { duration: '2m', target: 10 },   // 爬坡
    { duration: '10m', target: 10 },  // 稳态
    { duration: '2m', target: 0 },    // 收尾
  ],
  thresholds: { http_req_failed: ['rate<0.01'], http_req_duration: ['p(95)<500'] },
};

export default function () {
  const res = http.get(`${__ENV.BASE_URL}/api/products?page=1&size=20`);
  check(res, { 'status 200': (r) => r.status === 200 });
  sleep(1);
}
```

数据必须由脚本内生成或测试库专用，禁止打生产、禁止用真实手机号/真实订单。

## 8. 完成判据

- [ ] 每个核心接口有"未测试"或实测数据，二选一并注明环境
- [ ] 并发场景逐条给出"成立/不成立/待验证"及依据
- [ ] SQL 风险有 EXPLAIN 或 SQL 结构证据
- [ ] 同步延迟分三段描述，且指出哪些必须强一致
- [ ] 压测计划有停止条件与回滚方式，明确"未执行"
