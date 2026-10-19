# system-quality-audit

一个只读的**系统质量审计**技能（DSH / Claude-style `SKILL.md` 格式）：面向「微信小程序 + 后端 API + Web 管理后台 + 关系型数据库」类产品，覆盖支付/退款等资金链路、认证与越权、并发一致性、数据库性能与业务闭环，产出带证据分级与 P0–P3 优先级的可落地审计报告。

A read-only **system quality audit** skill for products built around a WeChat Mini Program, a backend API, an admin console and a relational database. It covers money flows (payment, refund, coupons), authentication/authorization, concurrency, database performance and end-to-end business closure, and produces an actionable report with evidence levels and P0–P3 priorities.

---

## 为什么是"技能"而不是一段提示词

把上千行的审计提示词塞进一次对话，会带来三个问题：上下文被占满、流程执行到一半就丢、结论容易脱离证据。这个技能把它们拆开：

- **主文件只放流程**：硬约束、六阶段、完成判据、常见失败模式，控制在 8192 字符渲染上限内。
- **细节按需加载**：`references/` 下按阶段放置检查表、SQL、模板、命令，读到时才进上下文。
- **用机制代替自觉**：证据三类标签、动态测试门禁、误报自查表、交付前自查清单，让"不得伪造""只读"变成可执行判据。

## 安装

技能发现规则：扫描根目录下的直接子目录 `<name>/SKILL.md`（或平铺 `<name>.md`），读取 YAML frontmatter 的 `name` 与 `description`。

| 范围 | 路径 | 适用 |
|---|---|---|
| 用户级（所有项目） | `~/.dsh/skills/system-quality-audit/` | 个人常用 |
| 项目级 | `<repo>/.dsh/skills/system-quality-audit/` | 随仓库走、团队共享 |
| 兼容目录 | `<repo>/.agents/skills/system-quality-audit/` | 其他兼容 agent 配置 |

```bash
# macOS / Linux
git clone https://github.com/weitaoLin505/system-quality-audit-.git \
  ~/.dsh/skills/system-quality-audit

# Windows PowerShell
git clone https://github.com/weitaoLin505/system-quality-audit-.git `
  "$env:USERPROFILE\.dsh\skills\system-quality-audit"
```

安装后无需重启：技能目录被监视，新增即出现在会话目录中。

## 使用

直接提出审计需求即可触发，例如：

- 「用 system-quality-audit 审计这个项目，先做阶段 1 和阶段 2 的静态部分」
- 「只跑支付链路专项：下单计价、回调验签、重复履约、退款」
- 「先出隔离环境验证方案，不要执行」

也可以显式调用：`skill: system-quality-audit`。

## 目录结构

```
system-quality-audit/
├── SKILL.md                  # 主流程：硬约束、六阶段、判据、失败模式
└── references/
    ├── 01-discovery.md       # 资产盘点、各框架路由提取命令、API 资产 JSON、调用链
    ├── 02-security.md        # SQLi / 认证越权 / 微信支付 v2+v3 / 退款对账 / OWASP API Top10
    ├── 03-performance.md     # 指标口径、并发矩阵、同步延迟、EXPLAIN、分级压测计划
    ├── 04-functional.md      # 功能矩阵、API 边界用例、状态机、闭环断点、完整性 SQL
    ├── 05-evidence.md        # 用例字段、动态测试门禁、证据命名、误报自查、工具矩阵
    └── 06-report.md          # 十节报告模板、P0–P3 判据、Top10 模板、发布结论
```

## 工作流概览

| 阶段 | 产出 |
|---|---|
| 0 立项与门禁 | 范围声明、授权状态、动态测试可行性 |
| 1 资产与调用链 | API 全量清单、三条调用链、表清单、高风险 Top5 |
| 2 安全审计 | 注入 → 越权 → 支付资金 → API/Web 四线台账 |
| 3 性能与并发 | 指标（未测写"未测试"）、并发场景判定、同步延迟三段、SQL 风险 |
| 4 功能闭环 | 功能矩阵、边界用例、状态机（含非法迁移）、闭环断点 |
| 5 证据与用例 | 用例表、证据文件、门禁记录、实测/代码可证/待验证分类 |
| 6 报告 | 十节报告 + Top10 优先修复 + 路线图 + 上线结论 |

交付物默认写入工作区的 `审计交付_<YYYYMMDD>/`，含 `evidence/` 证据目录。

## 三条硬约束

1. **只读**：不改业务代码、配置、数据库结构与生产数据；不跑迁移、不清缓存。
2. **资金零风险**：不触发真实扣款、退款、划转；仅限官方沙箱、Mock 或授权隔离环境。
3. **不虚构**：未实现的功能标"未发现实现"，没测的指标写"未测试"，静态推断不得表述为"已复现漏洞"。

每条结论必须标注证据等级：

- `[实测]` 授权隔离环境跑过，有输出或日志
- `[代码]` 静态可证，附 `file:line`
- `[待验证]` 依赖运行时或环境，需进一步确认

## 格式约定

- `name` 必须为 kebab-case；`description` 必填，用于会话目录召回。
- 可选 `whenToUse`、`metadata`、`user-invocable`、`disable-model-invocation`。
- 主文件正文渲染后建议控制在 8192 字符内，超出的细节放 `references/`。

## 贡献

欢迎提交新的检查项、框架路由提取规则、支付渠道差异（如支付宝、银联）或报告模板改进。请保持"每条检查项可判定、每条结论可归证"的写法。

## License

未指定。使用前请与仓库所有者确认授权范围。
