# 阶段二：安全审计检查表（A 数据库 → B 认证越权 → C 支付资金 → D API/Web）

原则：**先证伪再定性**。每条可疑点先找框架内置防护（ORM 参数化、全局校验、鉴权中间件），确认缺失后才写成缺陷；缺失本身也要给"全仓搜索无该中间件"的缺席证据。

## A. SQL 注入与数据库安全

### 检查点

| 项 | 怎么看 | 典型缺陷 |
|---|---|---|
| 原生 SQL | 搜索 `query(` `execute(` `raw(` `rawQuery` `createNativeQuery` `DB::select` | 拼接字符串：`"SELECT * FROM u WHERE id=" + id` |
| 动态拼接 | 搜索 `ORDER BY` `LIMIT` 附近的变量拼接 | 排序字段白名单缺失，`sort=id;drop` 类注入 |
| LIKE | 是否转义 `%` `_` | 通配符注入导致全表扫描/信息枚举 |
| IN 列表 | 是否 `?` 占位或逐个参数化 | 把数组 join 成字符串再拼 |
| 动态表名/列名 | 是否有白名单映射 | 直接拼表名 |
| ORM 逃逸 | `sequelize.literal` `whereRaw` `extra()` `$queryRawUnsafe` | 把用户输入塞进 literal |
| 参数化 | 是否使用占位符、命名参数 | `?` 与值顺序错位（逻辑漏洞） |
| 权限 | DB 账号是否 root/dbo、是否可 DDL/导出 | 应用账号权限过高 |
| 异常泄露 | 是否把 SQL/堆栈返回前端 | 报错含表名、字段名、路径 |
| 数据隔离 | 多租户/多门店/多用户是否有 where 条件兜底 | 只靠前端传 tenantId |

### 证据写法

```markdown
F-07 / P1：订单搜索的排序字段拼接进 SQL
证据：src/service/order.ts:118 `const sql = \`... ORDER BY ${req.query.sort}\``；全仓无白名单函数（rg "SORT_WHITELIST" 无结果）
可利用性：order 参数进入 ORDER BY，可布尔盲注/延时注入（需隔离环境实测确认）
修复：白名单映射 + 参数化 LIMIT
```

未授权环境不做注入实测，只写"静态可证 + 待验证"，并给出隔离环境验证步骤。

## B. 登录认证与访问控制

### 登录流程（微信）

- [ ] `code` 是否**服务端**调 `code2Session` 换 openid，而非前端传 openid/userId
- [ ] AppSecret 是否只在服务端（小程序包内出现即高危，包体可解）
- [ ] session_key 是否泄露给前端、是否落库明文
- [ ] 首次登录建号是否有竞态（并发同 openid 建两条用户）
- [ ] 手机号解密/绑定是否校验归属
- [ ] unionid 场景是否校验 appid 匹配

### Token / 会话

- [ ] 生成是否用安全随机（不是自增 ID、不是时间戳、不是可预测 UUIDv1）
- [ ] 存储：服务端 session 还是 JWT；JWT 是否校验 `alg`（禁 `none`）、是否校验 `exp`/`iat`、密钥是否硬编码/默认值
- [ ] 过期与刷新策略；登出是否真的失效（JWT 无黑名单则登出无效）
- [ ] 是否支持多端互踢、密码/角色变更后旧 token 是否失效
- [ ] token 是否出现在 URL、日志、Referer

### 授权（重点）

| 测试 | 方法 | 判定 |
|---|---|---|
| 普通用户打管理端 | 用 user token 请求 admin 路由 | 应 401/403；返回数据即越权 |
| BOLA/IDOR | 换 userId/orderId/addressId 访问他人资源 | 应以登录身份过滤，而非仅按入参查 |
| 角色伪造 | body 里加 `role: "admin"`、`isAdmin: true` | 不应被 Mass Assignment 吸收 |
| 字段级越权 | 更新接口能否改 `status`/`balance`/`price`/`ownerId` | 只允许白名单字段 |
| 管理端菜单绕过 | 直接调 API（跳过前端路由守卫） | 服务端必须独立校验 |
| 越权链 | 用户改自己订单状态为已支付/已退款 | 状态迁移只能服务端驱动 |

**缺席证据**：`rg -n "requireAdmin|isAdmin|role.*admin" src` 无结果，或中间件挂在路由之后（`app.use(auth)` 写在 `app.use('/api', routes)` 之后）。

## C. 微信支付与资金链路专项

### 1. 计价与下单

- [ ] 金额是否服务端按 DB 价格计算；客户端传 `price/total/amount` 是否被忽略
- [ ] 优惠券：归属、有效期、适用范围、门槛、使用次数、是否原子核销
- [ ] 精度：以"分"为整数，禁用 float；检查 `toFixed` 后仍用浮点比较
- [ ] 零元/负数：`price <= 0` 是否直接置为已支付
- [ ] 订单归属：创建后能否被他人支付/查询/退款
- [ ] 重复下单：同一业务是否生成多张支付单

### 2. 支付通知（最高危入口）

**APIv2（XML + MD5/HMAC-SHA256）**
- [ ] 验签：按字典序拼接 + `key=`，校验 `sign`；是否可被 `sign_type` 绕过
- [ ] 校验 `return_code`、`result_code`、`mch_id`、`appid`、`out_trade_no`、`total_fee`、`cash_fee`、`time_end`
- [ ] 回包必须 `<xml><return_code>SUCCESS</return_code></xml>`，否则微信重试

**APIv3（JSON + 平台证书 + AEAD_AES_256_GCM）**
- [ ] 校验头 `Wechatpay-Timestamp`（防重放窗口）、`Wechatpay-Nonce`、`Wechatpay-Serial`、`Wechatpay-Signature`
- [ ] 用平台证书验签 `timestamp\nnonce\nbody\n`
- [ ] `resource` 用 APIv3 密钥 AES-256-GCM 解密（`associated_data`、`nonce` 正确）
- [ ] 平台证书是否定期更新、序列号匹配
- [ ] 校验 `mchid`、`appid`、`out_trade_no`、`amount.total`、`amount.currency`、`trade_state == SUCCESS`
- [ ] 5 秒内响应；重试 8 次指数退避 → 处理必须**幂等且快速**（落库 + 入队，重活异步）

**通用缺陷**
- [ ] 是否存在"信任前端返回支付成功"的分支（`wx.requestPayment` success 即改单）
- [ ] 通知处理是否先查当前状态再更新（`UPDATE ... WHERE status='PENDING'` 或行锁）
- [ ] 幂等键：`out_trade_no` 唯一约束 / 通知表去重
- [ ] 并发通知：两个线程同时履约（发券、加积分、减库存）→ 需唯一约束或状态机 CAS
- [ ] 通知乱序：先到 `SUCCESS` 后到别的状态，是否回退
- [ ] 金额与订单金额不一致时是否告警并拒绝
- [ ] 测试/沙箱开关是否可能在生产开启（`if (env.dev) return true` 类短路）

### 3. 竞态与重复

- [ ] 订单状态更新是否 `WHERE status IN (...)` 乐观锁，或 `SELECT ... FOR UPDATE`
- [ ] 库存扣减是否 `UPDATE stock = stock - n WHERE stock >= n`（检查影响行数）
- [ ] 权益发放是否有唯一约束（`user_id + coupon_id` / `order_id`）
- [ ] 同一用户重复提交是否有幂等键或防重令牌
- [ ] 分布式锁：Redis 锁是否设置过期、是否校验持有者、是否误删他人锁

### 4. 退款

- [ ] 退款接口鉴权（仅管理端 + 二次确认）
- [ ] 额度：`refund_amount <= paid_amount - refunded_amount`，含并发校验
- [ ] 重复退款：退款单号唯一、状态机约束
- [ ] 多次部分退款累计与明细
- [ ] 退款结果以渠道回调/查单为准，不以本地请求成功为准
- [ ] 退款失败可重试、可人工干预；成功后库存/券/权益只回滚一次
- [ ] 退款后订单状态与实际资金一致

### 5. 对账与恢复

- [ ] 有主动查单（`/v3/pay/transactions/out-trade-no/`）与定时补偿
- [ ] 有支付流水表（渠道流水号、通知原文、处理结果、重试次数）
- [ ] 有每日对账（本地 vs 渠道），差异处理流程
- [ ] 宕机/超时后订单可恢复，不产生"已付款未履约"

## D. API / Web 安全

| 项 | 检查点 |
|---|---|
| 参数校验 | 类型、长度、枚举、范围；是否只有前端校验 |
| Mass Assignment | `Object.assign(entity, req.body)`、`Model.create(req.body)`、DTO 未过滤 |
| XSS | 后台富文本/昵称渲染是否转义；`v-html`、`dangerouslySetInnerHTML` |
| CSRF | Cookie 会话是否有 SameSite/Synchronizer Token；纯 token header 可豁免 |
| SSRF | 用户可传 URL 被服务端请求（图片抓取、回调地址、webhook） |
| 上传 | 扩展名/内容类型/大小限制、存储是否可执行、是否随机重命名、是否在 webroot |
| 路径穿越 | 文件下载 `../`、`path.join(userInput)` |
| CORS | `origin: *` + `credentials: true`；是否反射任意 Origin |
| 敏感信息 | 接口是否返回他人手机号/地址/身份证、是否返回 password/salt/session_key |
| 日志 | 是否记录 token、密码、支付回调原文（含密钥）、用户隐私 |
| 限流/防爆破 | 登录、短信、支付、导出是否限流；是否仅前端限制 |
| 错误响应 | 是否返回堆栈、SQL、内部路径、框架版本 |
| 管理端 | 危险操作二次确认、操作审计、批量操作上限 |

## E. OWASP API Security Top 10 映射（自检用）

| 类别 | 本项目对应检查 |
|---|---|
| API1 BOLA | B 节对象级越权 |
| API2 认证失效 | B 节 token/登录 |
| API3 属性级授权 | Mass Assignment、字段白名单 |
| API4 资源消耗 | 限流、分页上限、上传大小 |
| API5 功能级授权 | 普通用户打管理端 |
| API6 业务流滥用 | 重复下单、刷券、刷积分、批量导出 |
| API7 SSRF | D 节 |
| API8 配置错误 | CORS、调试开关、默认密钥、错误泄露 |
| API9 资产管理 | 旧版本接口、测试接口、文档暴露 |
| API10 不安全消费 | 第三方回调/响应未校验（支付、短信、OSS） |

## F. 完成判据

- [ ] 四线检查表逐项有结论（有/无/不适用/待验证），不是空白
- [ ] 每条缺陷有 file:line 或"全仓搜索无 X"的缺席证据
- [ ] 资金链路每个入口（建单/回调/查单/退款/对账）都有结论
- [ ] 严重度按可利用性与影响定级，不按修复难度
- [ ] 敏感信息已脱敏
