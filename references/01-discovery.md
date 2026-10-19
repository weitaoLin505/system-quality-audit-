# 阶段一：项目全貌、API 资产与调用链

目标：在任何判断之前，先建立可复核的资产底账。本阶段只读，产出的清单是后续所有结论的坐标系。

## 1. 技术栈指纹（先定框架，再定提取方法）

读这些文件即可定栈，不要靠猜：

- 根目录 `package.json` / `pyproject.toml` / `requirements.txt` / `composer.json` / `pom.xml` / `build.gradle` / `go.mod`
- 入口文件：`server.js` `app.js` `main.ts` `manage.py` `index.php` `Application.java`
- ORM/迁移：`prisma/schema.prisma`、`migrations/`、`alembic.ini`、`*.sql`、`models/`、`entity/`
- 小程序：`app.json`、`pages.json`、`project.config.json`（判断是原生还是 uni-app/Taro）
- 后台：`vite.config.*`、`vue.config.js`、`src/router/`、`src/api/`
- 部署：`Dockerfile`、`docker-compose.yml`、`nginx.conf`、CI 配置

记录：框架与版本、运行时版本、数据库类型与版本、缓存/MQ/对象存储、部署形态（单机/容器/反代）。

## 2. 路由提取（以代码为准，不以文档为准）

先用 glob 找路由文件，再按框架特征正则清点。命令用 `rg`（Windows 下注意引号转义）。

| 栈 | 路由声明特征 | 提取命令示例 |
|---|---|---|
| Express/Koa | `router.get(` `router.post(` `app.use(` | `rg -n "router\.(get|post|put|patch|delete)\(" src` |
| NestJS | `@Controller(` `@Get(` `@Post(` | `rg -n "@(Controller|Get|Post|Put|Patch|Delete)\(" src` |
| FastAPI | `@app.get(` `@router.post(` | `rg -n "@(app|router)\.(get|post|put|patch|delete)\(" .` |
| Flask | `@app.route(` | `rg -n "@app\.route\(" .` |
| Django | `path(` `re_path(` `urlpatterns` | `rg -n "urlpatterns|re_path\(" --glob "urls.py"` |
| Spring | `@RequestMapping` `@GetMapping` | `rg -n "@(Request|Get|Post|Put|Delete)Mapping" src` |
| ThinkPHP | `Route::rule(` `->rule(` | `rg -n "Route::(rule|get|post)\(|->rule\(" route app` |
| Laravel | `Route::get(` | `rg -n "Route::(get|post|put|patch|delete)\(" routes` |
| Gin/Echo | `r.GET(` `e.POST(` | `rg -n "\.(GET|POST|PUT|PATCH|DELETE)\(" .` |
| 原生 PHP | 入口文件 + `$_GET/$_POST` | `rg -n "\\\$_(GET|POST|REQUEST)" .` |

小程序侧调用点：`rg -n "wx\.request\(|wx\.uploadFile\(|uni\.request\(" --glob "*.js" --glob "*.ts" --glob "*.vue"`。
后台侧调用点：`rg -n "axios\.(get|post|put|delete)|fetch\(" src`。

**文档与代码冲突时以代码为准，并在报告写"文档缺失/过期"这一条。**

## 3. API 资产台账（JSON 供后续机器复核）

`API资产.json` 建议结构：

```json
{
  "generatedAt": "2026-01-01T00:00:00+08:00",
  "server": {"framework": "Express 4.x", "runtime": "Node 18", "db": "MySQL 8.0"},
  "endpoints": [
    {
      "id": "API-001",
      "method": "POST",
      "path": "/api/order/create",
      "auth": "user-token | admin-token | none",
      "roles": ["user"],
      "params": [{"name": "skuId", "in": "body", "type": "int", "validated": true}],
      "response": {"shape": "{code,data,msg}", "leaksInternals": false},
      "service": "OrderService.create",
      "tables": ["orders", "order_items", "skus"],
      "source": {"file": "src/routes/order.js", "line": 42},
      "risk": "high",
      "notes": "金额由客户端传入"
    }
  ],
  "summary": {"total": 0, "noAuth": 0, "adminOnly": 0, "unusedByFrontend": 0}
}
```

盘点时同步标注：无鉴权接口数、仅管理端可用接口数、前端从未调用的"孤儿接口"（可能是遗留后门入口，值得单独列）。

## 4. 调用链重建

对每条关键链路逐跳写清：

```
路由 → 中间件（鉴权/限流/校验）→ Controller → Service → Repository/ORM → 表
                                        ↘ 外部依赖（微信 API、OSS、MQ、Redis）
```

必须重建的三条：

1. 小程序 → API → Service → DB（至少覆盖：登录、首页/列表、详情、下单、支付状态查询、个人中心写操作）
2. 管理后台 → API → Service → DB（至少覆盖：列表查询、编辑/上下架、订单处理、退款、统计）
3. 支付链路：创建订单 → 服务端计价 → 创建支付单 → 调微信下单 → 返回 prepay → 小程序拉起支付 → 异步通知 → 验签解密 → 校验金额/商户号 → 更新订单 → 履约（发货/发券/加积分/开通权益）

每跳记录：是否有事务边界、是否跨外部 API、是否有锁、是否有缓存读写。**事务里调用外部 API 是要单独记账的风险点。**

## 5. 高风险模块识别（用于确定审计深度）

按"资金 / 权限 / 数据面"三个维度打分，命中的进深审：

- 涉及金额、余额、积分、优惠券、退款、库存扣减
- 涉及登录、注册、绑定手机号、角色变更、后台登录
- 涉及文件上传/下载、导出、导入、批量操作
- 涉及动态查询条件（搜索、排序、筛选、分页参数进入 SQL）
- 涉及 JSON 字段、多态关联、软删除、状态机流转
- 涉及定时任务、MQ 消费、回调入口（外部可触达）
- 涉及管理端写操作与越权面（最容易被忽视）

输出 Top5 高风险模块与理由，作为第二阶段取证顺序。

## 6. 既有测试与运维资产

- 测试目录、测试框架、fixture、seed、Mock（能否复用，别重复造）
- CI 流水线里的检查项（lint / 单测 / 构建），缺什么
- 日志与监控：日志落盘位置、是否结构化、有无请求 ID、有无慢查询日志
- 备份与回滚：有无备份策略、迁移是否可回退

这些直接决定第五阶段"能做什么动态测试"，缺什么就在报告里写"环境不具备"。

## 7. 本阶段完成判据

- [ ] 栈与版本已确认并写明来源
- [ ] API 数量与清单齐全，含鉴权方式与源码位置
- [ ] 三条调用链可画成图，支付链路每跳都有文件证据
- [ ] 表清单与外键/逻辑关联已列出
- [ ] 高风险 Top5 已定，且每条有文件依据
- [ ] `01_项目全貌与API清单.md` 与 `API资产.json` 已落盘
