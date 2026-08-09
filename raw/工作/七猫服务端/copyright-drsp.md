```
1. 一个请求的流程
```
## 数据库连接
文件在这里：configs/config.yaml
数据库名称：qimao_drsp

可以先挑一个真实接口顺着读，不要一开始试图看懂整个项目。以本次改动最核心的文字分成总览接口为例：

```http
GET /api/profit/overview?start_month=202501&end_month=202503&platform=-99999999&status=-99999999&source_ids=1001&source_ids=1002&import_content_types=1&import_content_types=3
```

它表示：查询 2025 年 1～3 月、指定合作方、指定引入内容类型的分成总览。

## 一、先建立整体心智模型

这个项目的请求路径大致是：

- `api/*.proto`：接口的“合同”，定义 URL、请求字段、响应字段。
- `api/*_http.pb.go`：由 proto 自动生成，把 HTTP 请求转换成 Go 方法调用。
- `internal/app/service`：应用层，负责校验/归一化参数和编排调用。
- `internal/app/domain`：核心业务规则，例如按月汇总、计算总计。
- `internal/app/domain/*/repo`：数据库查询和 SQL。
- `cmd/drsp`、`internal/server`、`internal/middleware`：启动、注册路由和鉴权。

这里通常没有传统 MVC 里的 Controller：生成的 HTTP handler 直接调用 `ProfitService`。

## 二、这个接口的完整调用链

```text
浏览器发起 GET /api/profit/overview
  │
  ├─ Gin 全局中间件
  │    ├─ IAM 鉴权
  │    ├─ Request ID / 日志
  │    ├─ 错误恢复
  │    └─ 统一响应包装
  │
  ├─ 生成的 HTTP Handler
  │    api/profit/profit_http.pb.go
  │    将 query 参数绑定成 GetProfitOverviewRequest
  │
  ├─ 应用 Service
  │    internal/app/service/profit.go
  │    ProfitService.GetProfitOverview
  │    ├─ 校验月份权限
  │    ├─ 兼容新旧月份/合作方参数
  │    └─ 先筛选可用合作方
  │
  ├─ 领域 Service
  │    internal/app/domain/profit/overview_range.go
  │    BuildProfitOverview
  │    ├─ 展开月份区间
  │    ├─ 跨月汇总数据
  │    └─ 计算列表项和合计
  │
  ├─ Repository
  │    ├─ partner/repo/unitive_fiction_source.go
  │    │    查符合合作方/状态/引入内容类型的 source_id
  │    └─ profit/repo/fiction_profit_repo.go
  │         查 monthly_partner_profit 并按月份、合作方聚合
  │
  └─ 返回 JSON
```

## 三、推荐阅读顺序

### 第 1 步：接口契约

先看 `api/profit/profit.proto`。

重点找：

- `rpc GetProfitOverview`
- URL：`/api/profit/overview`
- `GetProfitOverviewRequest`
- `GetProfitOverviewReply`

这一步要回答：

1. 前端可以传什么字段？
2. 每个字段是什么类型、含义是什么？
3. 哪些是本次新增字段？
4. 响应长什么样？

本次筛选重点字段是：

- `start_month`、`end_month`：月份区间
- `source_ids`：多合作方
- `import_content_types`：引入内容类型多选
- `platform`：平台
- `status`：合作方结算状态

`profit.pb.go`、`profit_http.pb.go` 都是生成文件，阅读理解可以，但通常不要手改；真正改接口先改 `.proto`，再执行项目规定的生成命令。

### 第 2 步：HTTP 入口

看 `api/profit/profit_http.pb.go`：

- `RegisterProfitHTTPServerController`
- `_Profit_GetProfitOverview0_HTTP_Handler`

你不需要逐行研究生成代码，只需确认：

```text
GET /api/profit/overview
→ GetProfitOverviewRequest
→ ProfitService.GetProfitOverview
```

尤其要知道：GET 请求中的 query 参数如何被绑定。例如：

```text
?start_month=202501&end_month=202503
```

会进入 `request.StartMonth` 和 `request.EndMonth`。

### 第 3 步：中间件

看：

- `cmd/drsp/http.go`
- `internal/middleware/middleware.go`

重点关注 `IamFilter`：`/api/profit/overview` 不在白名单中，生产环境会经过 IAM 权限校验。

同时，这个接口还会根据当前登录用户的角色限制可查询月份。也就是说，代码评审时不能只检查“传进来的月份是否合法”，还要检查“该角色是否允许看这个月”。

### 第 4 步：路由注册

看 `internal/server/http.go`。

重点确认：

```text
ProfitService
→ RegisterProfitHTTPServerController
```

这是你从“接口声明”找到“谁真正实现它”的连接点。

### 第 5 步：应用 Service

重点看 `internal/app/service/profit.go` 的：

```go
ProfitService.GetProfitOverview
```

这是你最应该先精读的手写代码。它负责：

1. 设置响应表头；
2. 获取当前用户角色；
3. 计算该用户可查询的月份边界；
4. 归一化请求参数；
5. 查询符合筛选条件的合作方；
6. 调用领域层完成汇总。

然后继续看：

- `internal/app/service/profit_month.go`

这里是本次“筛选优化”的关键之一：

- 新参数 `start_month + end_month` 优先；
- 老参数 `month` 仍兼容；
- 只传开始月或只传结束月应报错；
- `source_ids` 优先，回退兼容旧 `source_id`；
- 去重、过滤非法合作方 ID；
- 校验开始月份不晚于结束月份；
- 校验月份不能超过当前用户权限。

### 第 6 步：领域 Service

重点看：

- `internal/app/domain/profit/overview_range.go`
  - `BuildProfitOverview`
  - `loadProfitMonthlySummaries`

这里是核心业务汇总，不是简单的查库转发。

阅读时围绕三个问题：

1. 多个月份怎么展开和合并？
2. 为什么查询范围会从 `start_month` 的上个月开始？
3. 单月、多月展示字段有什么区别？

第二点尤其值得注意：单月场景需要计算“上月广告收入”，所以会多查询一个月的数据；多月范围则不展示这个单月专属概念。

### 第 7 步：实体定义

看这些数据结构即可：

- `internal/app/domain/profit/entity/vo/profit_overview.go`
- `internal/app/domain/profit/repo/po/monthly_partner_profit_total.go`
- `internal/app/domain/profit/repo/po/monthly_partner_profit.go`

区分概念：

- `Request / Reply`：接口输入输出；
- `VO`：业务层整理后的视图数据；
- `PO`：数据库行或 SQL 聚合结果；
- Entity：领域对象。

新手常见误区是看到一个结构体就默认它等于数据库表。这个项目里并不总是如此。

### 第 8 步：数据库层

最后看两个仓库，因为筛选条件被拆在了两处。

1. `internal/app/domain/partner/repo/unitive_fiction_source.go`

负责先筛合作方：

- `status`
- `source_ids`
- `import_content_types`

多个 `import_content_types` 用多个 `FIND_IN_SET` 条件组合，语义是“同时满足所有所选类型”（AND），不是“任意一个即可”（OR）。

2. `internal/app/domain/profit/repo/fiction_profit_repo.go`

重点看：

```go
ListPartnerProfitMonthlySums
```

负责查询 `monthly_partner_profit`：

- 按 `month`、`source_id` 汇总；
- `platform` 在这里过滤；
- 传入的合作方 ID 为空时直接返回空，避免无筛选的全表查询。

## 四、哪些重点看、哪些可以跳过

重点看：

- `api/profit/profit.proto`
- `cmd/drsp/http.go`
- `internal/middleware/middleware.go`
- `internal/server/http.go`
- `internal/app/service/profit.go`
- `internal/app/service/profit_month.go`
- `internal/app/domain/profit/overview_range.go`
- `internal/app/domain/partner/repo/unitive_fiction_source.go`
- `internal/app/domain/profit/repo/fiction_profit_repo.go`
- 前端参数拼装：`copyright-front/.../overview-shared.ts`

可先跳过：

- `*.pb.go`、`*.pb.validate.go`：生成代码，不适合逐行精读；
- `cmd/drsp/wire_gen.go`：Wire 自动生成；
- `internal/app/job/*`：定时任务，与请求读链路无关；
- 同一 repo 中其他扣款、结算、同步相关方法；
- `profit_grpc.pb.go`：这个请求是 HTTP 路径，不经过 gRPC；
- 有声分成代码：理解文字分成后，再对照 `album_profit.go` 阅读。

## 五、CR 时的重点检查项

针对这次筛选优化，建议按数据流检查：

1. Proto 是否声明了字段，前端是否实际传了同名参数。
2. 新旧参数是否兼容，优先级是否明确。
3. 空值是否有安全语义，尤其 `status`、`platform` 的默认值。
4. 月份范围是否校验完整：缺一端、起止倒置、越权、未来月份。
5. 多选筛选究竟是 AND 还是 OR，是否和产品语义一致。
6. 筛选条件是否落到了正确的表：
   - 合作方属性筛选应在 partner repo；
   - 分成记录属性筛选应在 profit repo。
7. SQL 是否可能在空筛选条件下扫描大量数据。
8. 单月和多月的返回字段是否保持一致的产品语义。
9. 旧客户端传 `month`、`source_id` 时是否仍能正常工作。
10. 生成文件是否由 `.proto` 重新生成，而非直接修改。

有一个值得重点核实的风险：`status` 未传时 Go 默认是 `0`，但“全部”使用 `-99999999`。如果前端遗漏 `status=-99999999`，后端可能会误筛选状态为 `0` 的合作方。

## 六、以后自己追任意接口的方法

固定按照这条线搜索：

```text
URL
→ proto rpc
→ 生成 HTTP handler
→ Service 方法
→ Domain 方法
→ Repo 方法
→ SQL / 表
```

例如你看到 `/api/profit/overview`，先全局搜索：

```text
GetProfitOverview
```

再依次看“谁调用它、它调用谁”。每读完一层，都用一句自己的话写下来：

```text
这一层输入什么？
做了什么转换/校验？
调用了谁？
输出什么？
```

只要能回答这四个问题，你就不是“看过”代码，而是真的追清了调用链。

### assembly
`assembly` 是项目内部的“组装/转换”工具包（英文 assembly 即“装配”）。

这里的：

assembly.CreateSelectOptions(partnerMeta.Import_Content_Type_Map, false)

作用是把 `Import_Content_Type_Map` 这种枚举映射，转换成接口返回给前端的下拉选项列表，例如每项通常包含 `value` 和 `label`。

因此这行是在给“引入内容类型”筛选框提供可选项；`false` 一般表示不额外插入“全部”选项。


profitDomainService.BuildProfitOverview

profitDomainService.BuildProfitMonthlyItems

为什么要新增这两个方法呢？而不是在原来的方法上改造
