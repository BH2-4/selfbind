# Selfbind 契约规格（Contract Spec）v0

> **前后端交界的生死线。** 本文档是并行开发的唯一契约源：前端据此 mock，后端据此实现，裁决 prompt 据此产出结构化输出。语言中立——仅用 JSON Schema（draft 2020-12）、SQLite DDL、OpenAPI 风格描述，不绑定任何后端语言；技术选型后再生成各自绑定。
>
> **冻结纪律**：T0 中午前冻结本规格（含 §6 状态机转移表）。冻结后只许填内容、不许改结构；任何结构性变更需全队确认后方可合入。
>
> 核心命题：**立法在模型、裁决在模型、执法在状态机。** 模型承重的部分（语义裁决、意图条款）与状态机保证的部分（violate 必拦、解锁门、审计只追加）在本文档中逐条可指认。

---

## 1. 契约 JSON Schema v0

### 1.1 设计要点

- **双层结构**：结构化条款（`scope` / `forbidden` / `acceptance`）+ 自然语言意图条款（`intent_clauses`）。后者是 AI 承重层——「给按钮加丝滑转场动画」不含禁区词、路径落在白名单内，规则判不了，模型依据意图条款判得了。
- `path_hint` / `detector_hint` 只是给裁决器的**线索，不是正则白名单**。是否在意图内由模型判定（§2）。
- `version` 单调递增，唯一递增路径是 amendment 冷却期走完（§6.2）。
- `amendments[]` 只追加，承载冷却期状态机（pending → cooling → effective | revoked）。

### 1.2 Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://selfbind.dev/schemas/contract.v0.json",
  "title": "Contract v0",
  "type": "object",
  "required": ["contract_id", "version", "goal", "scope", "intent_clauses",
               "forbidden", "acceptance", "deadline", "status", "created_at"],
  "properties": {
    "contract_id": {
      "type": "string",
      "pattern": "^ct_[0-9a-z]{4,8}$",
      "description": "全局唯一，生成后不可变"
    },
    "version": {
      "type": "integer",
      "minimum": 1,
      "description": "修约单调递增，初始 1。仅 amendment effective 时 +1，无直接编辑通道"
    },
    "goal": {
      "type": "string",
      "minLength": 4,
      "maxLength": 120,
      "description": "裁决的意图锚点。所有语义判定以 goal 为参照系"
    },
    "scope": {
      "type": "array",
      "minItems": 1,
      "description": "白名单，至少 1 条；空数组语义为全禁，故 schema 层面禁止空",
      "items": { "$ref": "#/$defs/ScopeRule" }
    },
    "intent_clauses": {
      "type": "array",
      "minItems": 1,
      "description": "自然语言意图条款（AI 承重层），至少 1 条",
      "items": { "$ref": "#/$defs/IntentClause" }
    },
    "forbidden": {
      "type": "array",
      "description": "禁区，可为空数组",
      "items": { "$ref": "#/$defs/ForbidRule" }
    },
    "acceptance": {
      "type": "array",
      "minItems": 1,
      "description": "验收断言，至少 1 条。全部通过是 done 与停车场解锁的唯一门",
      "items": { "$ref": "#/$defs/Assertion" }
    },
    "deadline": {
      "type": "string",
      "format": "date-time",
      "description": "ISO 8601。惰性比对：now > deadline 时契约自动 expired（§6.1）"
    },
    "amendments": {
      "type": "array",
      "description": "只追加。每元素是一个冷却期状态机实例（§6.2）",
      "items": { "$ref": "#/$defs/Amendment" }
    },
    "status": {
      "enum": ["draft", "active", "done", "expired"],
      "description": "契约状态机（§6.1）"
    },
    "created_at": { "type": "string", "format": "date-time" }
  },
  "$defs": {
    "ScopeRule": {
      "required": ["id", "desc"],
      "properties": {
        "id": { "type": "string", "pattern": "^sc_[0-9]+$" },
        "desc": { "type": "string", "description": "允许做什么（意图描述，非正则）" },
        "path_hint": { "type": "string", "description": "可选线索，如 src/**,docs/**" },
        "action": { "enum": ["file_edit", "read_file", "run_cmd"],
                    "description": "可选。Agent 工具封顶 3 个，与之一一对应" }
      }
    },
    "IntentClause": {
      "required": ["id", "text"],
      "properties": {
        "id": { "type": "string", "pattern": "^ic_[0-9]+$" },
        "text": { "type": "string",
                  "description": "如「不做与 goal 无关的纯视觉炫技」「不为未失败的代码做优化」" }
      }
    },
    "ForbidRule": {
      "required": ["id", "desc"],
      "properties": {
        "id": { "type": "string", "pattern": "^fb_[0-9]+$" },
        "desc": { "type": "string", "description": "如「新增 npm 依赖」" },
        "detector_hint": { "type": "string",
                           "description": "可选线索，如「package.json 出现新增 dependencies」" }
      }
    },
    "Assertion": {
      "oneOf": [
        {
          "required": ["id", "type", "cmd", "expect"],
          "properties": {
            "id": { "type": "string", "pattern": "^acc_[0-9]+$" },
            "type": { "const": "auto" },
            "cmd": { "type": "string", "description": "如 pytest e2e/ -q" },
            "expect": { "type": "string", "description": "如 exit_code==0" }
          }
        },
        {
          "required": ["id", "type", "question"],
          "properties": {
            "id": { "type": "string", "pattern": "^acc_[0-9]+$" },
            "type": { "const": "manual" },
            "question": { "type": "string", "description": "如「演示视频成片≤3min」" },
            "confirm_by": { "const": "user", "default": "user" }
          }
        }
      ]
    },
    "Amendment": {
      "description": "修约冷却期状态机。同一契约至多一个非终态（pending/cooling）修约",
      "required": ["amendment_id", "to_v", "requested_at", "state", "diff"],
      "properties": {
        "amendment_id": { "type": "string", "pattern": "^am_[0-9a-z]{4,8}$" },
        "to_v": { "type": "integer", "minimum": 2,
                  "description": "生效后的目标版本，发起时锁定为 version+1" },
        "requested_at": { "type": "string", "format": "date-time" },
        "cooling_until": {
          "oneOf": [{ "type": "string", "format": "date-time" }, { "type": "null" }],
          "description": "冷却截止。pending 阶段为 null，confirm 后写入 = now + 冷却常数"
        },
        "state": {
          "enum": ["pending", "cooling", "effective", "revoked"],
          "description": "pending=已受理待二次确认；cooling=冷却中可 revoke；effective=冷却尽生效且 version+1；revoked=回心转意，永不生效"
        },
        "diff": { "type": "string",
                  "description": "人可读的变更描述，如 scope += docs/**。生效后由后端应用到契约条款" }
      }
    }
  }
}
```

### 1.3 示例实例

```json
{
  "contract_id": "ct_8f3k",
  "version": 1,
  "goal": "9/6 12:00 前交付 Selfbind 可演示版",
  "scope": [
    { "id": "sc_0", "desc": "编辑源码与文档", "path_hint": "src/**,docs/**", "action": "file_edit" }
  ],
  "intent_clauses": [
    { "id": "ic_0", "text": "不做与 goal 无关的纯视觉炫技" },
    { "id": "ic_1", "text": "不为未失败的代码做优化" }
  ],
  "forbidden": [
    { "id": "fb_0", "desc": "新增 npm 依赖", "detector_hint": "package.json 出现新增 dependencies" }
  ],
  "acceptance": [
    { "id": "acc_0", "type": "auto", "cmd": "pytest e2e/ -q", "expect": "exit_code==0" },
    { "id": "acc_1", "type": "manual", "question": "演示视频成片≤3min" }
  ],
  "deadline": "2026-09-06T12:00:00+08:00",
  "amendments": [],
  "status": "active",
  "created_at": "2026-09-05T10:30:00+08:00"
}
```

---

## 2. 裁决器输出 JSON Schema

### 2.1 设计要点

- `verdict` 三分类；`confidence < 0.70` **强制 unclear**（宁可交人不误拦）。
- `clauses` 只能引用契约实际存在的条款 ID（`sc_*` / `ic_*` / `fb_*`）。裁决 prompt 以 enum 结构化输出约束；**渲染层二次校验**：引用了不存在的条款 → 降级显示「依据意图条款」。引用幻觉不改变拦截结果（violate 就是 violate），只影响展示。
- `tag_suggestion` 是**开放元数据**：建议值 whim | beyond | avoidance | null，用户可改可删可自定义；词表放独立配置文件 `tags.json`，不进本 schema；**永不参与状态机与解锁条件**（不变量 I4）。
- 模型超时或输出不可解析 → 后端降级构造 `verdict=unclear` 且 `fallback=true` 的输出（不变量 I5）。

### 2.2 Schema

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "https://selfbind.dev/schemas/verdict.v0.json",
  "title": "Verdict v0",
  "type": "object",
  "required": ["verdict", "clauses", "reason", "tag_suggestion", "confidence"],
  "properties": {
    "verdict": { "enum": ["allow", "violate", "unclear"] },
    "clauses": {
      "type": "array",
      "items": { "type": "string", "pattern": "^(sc|ic|fb)_[0-9]+$" },
      "description": "引用的条款 ID。violate/unclear 必填非空；allow 应为空。合法性由渲染层对照契约校验"
    },
    "reason": {
      "type": "string", "maxLength": 50,
      "description": "人可读裁决理由（≤50 字），如「彩虹渐变属纯视觉炫技，违反 ic_0」"
    },
    "tag_suggestion": {
      "enum": ["whim", "beyond", "avoidance", null],
      "description": "仅 violate 时可有非空建议。词表定义在 tags.json，此处枚举仅为出厂默认值"
    },
    "confidence": {
      "type": "number", "minimum": 0, "maximum": 1,
      "description": "< 0.70 时后端强制改判 unclear（保留原值入审计）"
    }
  }
}
```

### 2.3 示例输出

```json
{
  "verdict": "violate",
  "clauses": ["ic_0"],
  "reason": "彩虹渐变属纯视觉炫技，违反 ic_0",
  "tag_suggestion": "whim",
  "confidence": 0.86
}
```

### 2.4 tags.json（词表配置文件，独立于本 schema）

```json
{
  "whim":     { "label": "三分钟热度", "hint": "与 goal 无关的新念头" },
  "beyond":   { "label": "超出水平",   "hint": "当前能力承接不了的范围" },
  "avoidance":{ "label": "回避主线",   "hint": "用杂事推迟验收断言所指的正事" }
}
```

词表可增删改（用户自定义标签进卡片 `tag` 字段自由文本）；状态机与解锁条件的判定函数**不得读取** `tag` 与 `tag_suggestion`。

出厂默认为 `tags.json`（与本文档 JSON 系一致）；格式随技术选型可为 YAML 等，**外置与非硬编码是硬要求**（PRD v1.1 拍板②）。

---

## 3. 停车场卡片 SQLite DDL

```sql
CREATE TABLE parking_card (
  card_id      TEXT PRIMARY KEY,              -- pk_ 前缀 + 短随机串，如 pk_01H…
  contract_id  TEXT NOT NULL,                 -- 所属契约（卡片永不跨契约转移）
  request_raw  TEXT NOT NULL,                 -- 被拦截的请求原文
  tag          TEXT,                          -- 归档标签：tags.json 词表键或用户自定义；NULL = 无标签
  tag_reason   TEXT,                          -- 模型建议理由（可空），供用户改标签时参考
  created_at   TEXT NOT NULL,                 -- ISO 8601
  state        TEXT NOT NULL DEFAULT 'locked'
               CHECK (state IN ('locked', 'unlocked')),
  unlocked_by  INTEGER,                       -- 触发解锁的 acceptance 审计事件 seq（§4）
  CHECK (
    (state = 'locked'   AND unlocked_by IS NULL) OR
    (state = 'unlocked' AND unlocked_by IS NOT NULL)
  )
);

CREATE INDEX idx_card_contract ON parking_card (contract_id, state);
```

**状态转移硬规则**：

- `locked → unlocked` 的**唯一路径** = 所属契约的全部 `acceptance` 断言通过（不变量 I3）。
- 唯一实现位置：`POST /contracts/{id}/acceptance/run` 的事务内——`all_passed` 为真时，批量 `UPDATE ... SET state='unlocked', unlocked_by=<本次验收事件 seq> WHERE contract_id=? AND state='locked'`。
- `unlocked` 为终态，MVP 无反向转移。标签的增删改**只更新 `tag`/`tag_reason` 字段，触碰不到 `state`**。

---

## 4. audit.jsonl 事件 Schema

### 4.1 总则

- **只追加，不可改不可删**：进程内唯一写入口，无 update/delete API，UI 无删除入口。每行一个 JSON 对象，`seq` 全局单调递增且等于行号（1 起）。
- 事件公共头（所有事件必填）：

| 字段 | 类型 | 说明 |
|---|---|---|
| `seq` | integer | 全局单调递增，= 行号；交叉引用锚点 |
| `ts` | string (ISO 8601) | 事件时间戳 |
| `type` | string | 五类之一：`verdict` / `intercept` / `unclear_resolve` / `amend` / `acceptance`。**五类之外，写入器拒绝落盘**（不变量 I2） |
| `contract_id` | string | 所属契约 |
| `version` | integer | 事件发生时的契约版本（amend effective 事件记生效后的 `to_v`） |

### 4.2 五类事件

**verdict** —— 每次 `POST /requests` 裁决完成时追加（含 fallback 降级）：

```json
{ "seq": 142, "ts": "2026-09-05T23:41:02+08:00", "type": "verdict",
  "contract_id": "ct_8f3k", "version": 1,
  "request_id": "rq_01h", "req_hash": "sha256:9f2a…",
  "verdict": "violate", "clauses": ["ic_0"], "reason": "彩虹渐变属纯视觉炫技，违反 ic_0",
  "confidence": 0.86, "model": "glm-4.7", "latency_ms": 890, "fallback": false }
```

专有字段：`request_id`、`req_hash`（请求原文哈希，原文全文在 intercept/卡片中）、`verdict`、`clauses`、`reason`、`confidence`、`model`、`latency_ms`、`fallback`（true = 模型超时/不可解析，后端降级 unclear）。

**intercept** —— violate 拦截并生成停车场卡片时追加：

```json
{ "seq": 143, "ts": "2026-09-05T23:41:02+08:00", "type": "intercept",
  "contract_id": "ct_8f3k", "version": 1,
  "request_id": "rq_01h", "card_id": "pk_01hX",
  "request_raw": "给按钮加彩虹渐变", "clauses": ["ic_0"],
  "tag": "whim", "tag_reason": "与 goal 无关的视觉炫技" }
```

专有字段：`request_id`、`card_id`、`request_raw`、`clauses`、`tag`（初值 = tag_suggestion，用户此后改标签**不回写**本事件——审计只追加）。

**unclear_resolve** —— 存疑请求得到人裁（或超时自动拒绝）时追加：

```json
{ "seq": 144, "ts": "2026-09-05T23:45:10+08:00", "type": "unclear_resolve",
  "contract_id": "ct_8f3k", "version": 1,
  "request_id": "rq_01j", "seq_ref": 141,
  "resolved_by": "user", "resolution": "allow_once",
  "tag_override": null, "amend_id": null }
```

专有字段：`request_id`、`seq_ref`（指向该请求 verdict 事件的 seq）、`resolved_by`（`user` | `timeout`，30 分钟未裁由后端惰性判定为 timeout）、`resolution`（`allow_once` | `deny`；timeout 时恒为 deny）、`tag_override`（deny 时用户改过的标签，可空）、`amend_id`（勾选「同时修约」时指向生成的 amendment，可空）。

**amend** —— 修约状态机每次转移各追加一条（pending/cooling/effective/revoked 四相流转全程留痕）：

```json
{ "seq": 145, "ts": "2026-09-06T00:02:00+08:00", "type": "amend",
  "contract_id": "ct_8f3k", "version": 1,
  "amendment_id": "am_02kq", "from_v": 1, "to_v": 2, "state": "cooling", "diff": "scope += docs/**" }
```

专有字段：`amendment_id`、`from_v`、`to_v`、`state`（本事件进入的状态）、`diff`。`effective` 事件的公共头 `version` 记 `to_v`。

**acceptance** —— 验收运行中每个断言出结果时各追加一条：

```json
{ "seq": 146, "ts": "2026-09-06T09:40:00+08:00", "type": "acceptance",
  "contract_id": "ct_8f3k", "version": 1,
  "run_id": "ar_03m", "acc_id": "acc_0", "acc_type": "auto", "passed": true,
  "detail": "exit_code==0 (pytest e2e/ -q, 2140ms)" }
```

专有字段：`run_id`（一次 acceptance/run 的标识）、`acc_id`、`acc_type`（`auto` | `manual`）、`passed`、`detail`（auto 记命令与退出码；manual 记确认时间）。全过时最后一条 acceptance 事件的 `seq` 即停车场卡片 `unlocked_by` 的取值。

---

## 5. REST API 草案

约定：Base URL `/api/v1`；时间一律 ISO 8601；错误响应统一 `{ "error": { "code": "…", "message": "…" } }`；业务结果（被拦/存疑）不占 HTTP 错误码，恒用 200 表达「请求已处理」；409 = 状态机拒绝，404 = 资源不存在，422 = 请求体校验失败。

### 5.1 POST /contracts —— 对话立约（立法）

自然语言对话 → 模型抽取双层结构化契约（draft）。schema 校验失败自动重生成，至多 2 次，仍失败返回 502。

请求：

```json
{ "messages": [
    { "role": "user",      "content": "周末冲刺签约跟踪 App。只做登录/列表/详情三页，不加任何动画库和新框架，三条用户旅程端到端要过，周日晚截止。" },
    { "role": "assistant", "content": "收到。确认两个边界：列表页的筛选动画算不算「动画库」？单元测试失败的修复算不算 scope？" },
    { "role": "user",      "content": "筛选动画用系统自带过渡可以做，引库不行；测试修复算 scope。" }
  ] }
```

响应 `201`：

```json
{ "contract": {
    "contract_id": "ct_8f3k", "version": 1, "status": "draft",
    "goal": "9/6 12:00 前交付 Selfbind 可演示版",
    "scope": [ { "id": "sc_0", "desc": "编辑源码与文档", "path_hint": "src/**,docs/**", "action": "file_edit" } ],
    "intent_clauses": [ { "id": "ic_0", "text": "不做与 goal 无关的纯视觉炫技" } ],
    "forbidden": [ { "id": "fb_0", "desc": "新增 npm 依赖", "detector_hint": "package.json 出现新增 dependencies" } ],
    "acceptance": [ { "id": "acc_0", "type": "auto", "cmd": "pytest e2e/ -q", "expect": "exit_code==0" } ],
    "deadline": "2026-09-06T12:00:00+08:00",
    "amendments": [], "created_at": "2026-09-05T10:30:00+08:00"
} }
```

### 5.2 POST /contracts/{id}/activate —— 确认激活

前端渲染 draft 供用户逐条确认，确认完成后调用。非 draft 状态返回 409；若惰性比对发现已过 deadline，契约置 expired 并返回 409 `contract_expired`。

请求：`{ "confirmed": true }`

响应 `200`：`{ "contract": { …, "status": "active" } }`（完整契约对象，下同）

### 5.3 POST /requests —— 执行请求（裁决 + 执行/拦截，同步）

同步阻塞，**8 秒超时**。超时或模型输出不可解析 → 降级 `unclear`（fallback），请求进入 pending 人裁，不执行不丢弃。裁决前置检查：契约非 active（含惰性过期）返回 409。

请求：

```json
{ "contract_id": "ct_8f3k",
  "request_raw": "给按钮加彩虹渐变",
  "tool": "file_edit",
  "payload": { "target": "src/Button.tsx" } }
```

响应 · allow（`200`）：

```json
{ "request_id": "rq_01g", "verdict": "allow", "executed": true,
  "clauses": ["sc_0"], "reason": "修复登录页空指针，在 sc_0 意图内",
  "confidence": 0.93,
  "execution": { "tool": "file_edit", "ok": true, "stdout": null, "stderr": null, "duration_ms": 412 } }
```

响应 · violate（`200`，同步返回停车场卡片）：

```json
{ "request_id": "rq_01h", "verdict": "violate", "executed": false,
  "clauses": ["ic_0"], "reason": "彩虹渐变属纯视觉炫技，违反 ic_0",
  "confidence": 0.86,
  "parking_card": {
    "card_id": "pk_01hX", "contract_id": "ct_8f3k",
    "request_raw": "给按钮加彩虹渐变", "tag": "whim", "tag_reason": "与 goal 无关的视觉炫技",
    "created_at": "2026-09-05T23:41:02+08:00", "state": "locked", "unlocked_by": null } }
```

响应 · unclear（`200`，等待人裁）：

```json
{ "request_id": "rq_01j", "verdict": "unclear", "executed": false,
  "clauses": [], "reason": "「换更安全的登录库」替换还是新增，契约未明写",
  "confidence": 0.61, "fallback": false,
  "pending": { "state": "pending", "pending_until": "2026-09-06T00:15:00+08:00",
               "resolution_endpoint": "/api/v1/requests/rq_01j/resolve" } }
```

### 5.4 POST /contracts/{id}/amendments —— 发起修约（进入 pending，二次确认后冷却）

契约必须 active；同一契约已存在非终态（pending/cooling）修约时返回 409。

请求：

```json
{ "diff": "scope += docs/**（允许补充文档写作）" }
```

响应 `201`：

```json
{ "amendment": {
    "amendment_id": "am_02kq", "contract_id": "ct_8f3k",
    "to_v": 2, "requested_at": "2026-09-06T00:01:30+08:00",
    "cooling_until": null, "state": "pending",
    "diff": "scope += docs/**（允许补充文档写作）" },
  "confirm_challenge": { "type": "contract_name", "hint": "输入契约名以确认修约" } }
```

### 5.5 POST /amendments/{id}/confirm —— 二次确认（pending → cooling）

落实「输入契约名二次确认」。契约名错误返回 422，amendment 保持 pending（不晋级、不删除，可重试）。

请求：`{ "contract_name": "愚者自缚" }`

响应 `200`：

```json
{ "amendment": { "amendment_id": "am_02kq", "to_v": 2,
    "state": "cooling",
    "cooling_until": "2026-09-06T00:07:30+08:00",
    "diff": "scope += docs/**（允许补充文档写作）" } }
```

（冷却常数为可配置项 `cooling_minutes`，演示版分钟级。）

### 5.6 POST /amendments/{id}/revoke —— 回心转意（pending/cooling → revoked）

请求：`{}`

响应 `200`：

```json
{ "amendment": { "amendment_id": "am_02kq", "state": "revoked",
    "diff": "scope += docs/**（允许补充文档写作）" } }
```

已 effective 的修约返回 409（生效即不可逆）。

### 5.7 POST /requests/{id}/resolve —— 存疑人裁（决策权在人）

目标请求必须处于 pending。`resolution=allow_once` → 本次执行（不修契约）；`resolution=deny` → 维持拦截并生成停车场卡片。可附 `tag_override`（用户改标签）；可附 `amend.diff`（勾选「同时修约」：创建 amendment 走标准冷却，见附录 A-4）。超过 `pending_until` 由后端惰性判定自动 deny（`resolved_by=timeout`），此接口随后返回 409。

请求：

```json
{ "resolution": "deny", "tag_override": "beyond", "amend": null }
```

响应 · deny（`200`）：

```json
{ "request_id": "rq_01j", "resolution": "deny",
  "parking_card": {
    "card_id": "pk_01jZ", "contract_id": "ct_8f3k",
    "request_raw": "换一个更安全的登录库", "tag": "beyond", "tag_reason": null,
    "created_at": "2026-09-06T00:12:00+08:00", "state": "locked", "unlocked_by": null } }
```

响应 · allow_once（`200`）：`{ "request_id": "rq_01j", "resolution": "allow_once", "executed": true, "execution": { …同 5.3… } }`

### 5.8 POST /contracts/{id}/acceptance/run —— 验收（解锁门）

运行全部断言：auto 断言执行 `cmd`（实现自由度：宿主直跑或降级人工标记，见附录 A-8）；manual 断言取请求体确认。契约须 active。副作用（同一事务）：全过 → 契约置 done + 该契约全部 locked 卡解锁（`unlocked_by` = 本 run 最后一条 acceptance 事件 seq）；任一失败 → 契约保持 active、卡片保持 locked。运行前先惰性处理：deadline 过期（→ 409 expired）与到点修约生效（version+1）。

请求：

```json
{ "confirmations": { "acc_1": true } }
```

响应 `200`：

```json
{ "run": {
    "run_id": "ar_03m",
    "results": [
      { "acc_id": "acc_0", "type": "auto",   "passed": true,  "detail": "exit_code==0 (2140ms)" },
      { "acc_id": "acc_1", "type": "manual", "passed": true,  "detail": "user confirmed 2026-09-06T09:40:05+08:00" }
    ],
    "all_passed": true },
  "contract": { "contract_id": "ct_8f3k", "version": 1, "status": "done" },
  "unlocked_cards": ["pk_01hX", "pk_01jZ"] }
```

### 5.9 GET /parking-cards —— 停车场列表

查询参数：`state=locked|unlocked`（可缺省=全部）、`contract_id`（可缺省）。

响应 `200`：

```json
{ "cards": [
    { "card_id": "pk_01hX", "contract_id": "ct_8f3k",
      "request_raw": "给按钮加彩虹渐变", "tag": "whim", "tag_reason": "与 goal 无关的视觉炫技",
      "created_at": "2026-09-05T23:41:02+08:00", "state": "locked", "unlocked_by": null },
    { "card_id": "pk_01jZ", "contract_id": "ct_8f3k",
      "request_raw": "换一个更安全的登录库", "tag": "beyond", "tag_reason": null,
      "created_at": "2026-09-06T00:12:00+08:00", "state": "unlocked", "unlocked_by": 147 }
  ] }
```

### 5.10 GET /audit —— 审计游标流式读取

查询参数：`after_seq`（缺省 0，返回 seq > after_seq 的事件）、`limit`（缺省 100，上限 1000）。

响应 `200`：

```json
{ "events": [ { "seq": 148, "ts": "…", "type": "acceptance", "contract_id": "ct_8f3k", "version": 1, "run_id": "ar_03m", "acc_id": "acc_1", "acc_type": "manual", "passed": true, "detail": "…" } ],
  "next_cursor": 148,
  "has_more": false }
```

### 5.11 GET /contracts/{id} —— 契约读取（惰性判定触发点）

响应 `200`：`{ "contract": { …完整契约对象，含 amendments… } }`

读取时后端惰性处理：deadline 到点 → expired；cooling 修约到点 → effective（version+1，记 amend 事件）。冷却进度、版本变化均以此接口轮询。

### 5.12 消融对照旁路（「证据」标签页，PRD v1.1 拍板⑤）

演示用三条**实时同源**管线并行跑同一请求：无裁决放行 / 纯规则引擎（glob + 关键词）/ 模型双层裁决（主管线，即 5.3）。对照实现约束（红线）：

- 两条对照管线**只返回「假设裁决结果」，永不进入执行分支**——「无裁决放行」栏绝不允许真的执行请求，否则破坏不变量 I1；
- 对照管线不产生审计事件、不生成停车场卡片、不影响契约与卡片状态机；
- 请求/响应形状由前端与后端协商（如 `POST /ablation/run`），不进入本规格冻结范围；唯一冻结点是前述两条约束。

---

## 6. 状态机转移表

> **硬编码转移表 + 单测钉死。变更需全队确认。** 三个状态机（契约 / 修约 / 停车场卡片）在代码中表现为显式转移表常量，任何不在表中的转移请求一律拒绝并抛断言错误；每个状态机的全部合法/非法转移必须有对应单测。

### 6.1 契约状态机

| 当前状态 | 触发 | 下一状态 | 守卫条件 |
|---|---|---|---|
| （无） | `POST /contracts` 生成 | `draft` | schema 校验通过（失败自动重生成 ≤2 次） |
| `draft` | `POST /contracts/{id}/activate` | `active` | 用户逐条确认 `confirmed=true`；`now < deadline` |
| `draft` | 用户放弃确认 | （契约不存在） | draft 可静默删除，系统无约束，不进审计 |
| `draft` | 惰性比对（activate 时） | `expired` | `now > deadline`：拒绝激活，置 expired |
| `active` | amendment `effective`（惰性） | `active`（`version+1`） | `now ≥ cooling_until` 且该修约未被 revoke；状态不变，版本进位 |
| `active` | `POST …/acceptance/run` 全过 | `done` | 全部 acceptance 断言 `passed=true` |
| `active` | 惰性比对（任意 API 触碰） | `expired` | `now > deadline`；此后一切执行请求 409 |
| `done` | — | （终态） | 无出边：请求/修约/验收均 409 |
| `expired` | — | （终态） | 无出边：仅可查看与导出 |

deadline 为**惰性执法**：无后台定时器，任何 API 触碰契约时比对 `now > deadline` 则转移。

### 6.2 修约（amendment）状态机

| 当前状态 | 触发 | 下一状态 | 守卫条件 |
|---|---|---|---|
| （无） | `POST /contracts/{id}/amendments` | `pending` | 契约 active；无其他非终态修约 |
| `pending` | `POST /amendments/{id}/confirm` | `cooling` | 契约名正确（错误则 422 原地不动） |
| `pending` | `POST /amendments/{id}/revoke` | `revoked` | — |
| `cooling` | 惰性比对（`now ≥ cooling_until`） | `effective` | 未被 revoke；触发契约 `version → to_v`、diff 应用、记 amend 事件 |
| `cooling` | `POST /amendments/{id}/revoke`（冷却内任意时刻，「回心转意」） | `revoked` | `now < cooling_until` |
| `effective` | — | （终态） | 不可逆 |
| `revoked` | — | （终态） | 永不生效，留档可查 |

### 6.3 停车场卡片状态机

| 当前状态 | 触发 | 下一状态 | 守卫条件 |
|---|---|---|---|
| `locked` | 所属契约 acceptance 全过（`acceptance/run` 事务内） | `unlocked` | **唯一路径**（不变量 I3）；`unlocked_by` = 本次最后一条 acceptance 事件 seq |
| `unlocked` | — | （终态） | 无反向转移 |

---

## 7. 不变量（红线）

以下条款为系统不可妥协的性质，任何实现必须可指认其 enforcement 代码位置，并以单测钉死：

- **I1 violate 必拦**：执行路由是代码硬分支——只有 `verdict=allow` 或 `unclear_resolve.resolution=allow_once` 两个入口能进入执行；裁决 JSON 中不存在任何可触发执行的字段，模型无放行通道。violate 的请求同步返回停车场卡片，永不下发执行。消融对照旁路（§5.12）同受此约束：对照管线永不执行。
- **I2 审计只追加**：audit.jsonl 由进程内唯一写入器追加；无 update/delete API，UI 无删除入口；事件 `type` 不在五类枚举内（verdict / intercept / unclear_resolve / amend / acceptance）时，写入器在落盘前拒绝。
- **I3 解锁门唯一**：`locked → unlocked` 唯一路径 = 所属契约全部验收断言通过，唯一实现点在 `acceptance/run` 事务内；不存在任何其他 UPDATE 卡片 state 的代码路径。
- **I4 标签不进状态机**：`tag` / `tag_suggestion` 是开放元数据（词表外置 tags.json，用户可改可删可自定义），三个状态机的全部转移与解锁条件的判定函数不得读取标签字段；单测以「删除全部标签数据后状态机行为逐转移不变」钉死。
- **I5 宁交人不误拦**：`confidence < 0.70` 强制 unclear；模型超时（>8s）或输出不可解析时降级 unclear（`fallback=true`）；unclear 不直接执行、也不直接生成停车场卡片——必须人裁（allow_once / deny）或 30 分钟超时自动 deny。
- **I6 version 单调递增**：`version+1` 仅由 amendment `effective` 一条路径触发；不存在直接编辑契约条款的 API。
- **I7 deadline 惰性执法**：过期后一切执行请求拒绝（409 `contract_expired`），契约自动进 expired 终态。
- **I8 引用幻觉不改变结果**：`clauses` 引用不存在的条款 ID 时，仅渲染层降级显示「依据意图条款」；拦截/放行结果不受影响，后端不得因引用无效而重裁决。

---

## 附录 A：与 PRD 的差异清单

### A.1 与 PRD v1 §2 的差异

本规格以作战版 PRD（synthesize:prd v1）§2 为基准起草，以下为主动修正或补充之处，逐条经发起人过目：

1. **amendment 结构升级**：PRD §2.1 的 `amendments[{to_v, at, diff, operator}]` 是简单日志；本规格按冷却期需求重构为状态机实例 `{amendment_id, to_v, requested_at, cooling_until, state, diff}`，PRD §2.5 的「冷却期计数器」落成显式四态机（§6.2），operator 字段因单用户 MVP 冗余而移除。
2. **新增 `POST /amendments/{id}/confirm`**：PRD §2.5「需输入契约名二次确认」没有对应 API；本规格补上，并为 `pending` 态提供真实语义（已受理、未二次确认）。
3. **新增 `GET /contracts/{id}`**：deadline 与冷却期均为惰性判定，需要一个读取型触发点；前端查询冷却进度与版本变化也依赖它。
4. **「同时修约」改为走标准冷却**：PRD §2.6 存疑通道勾选「同时修约」原文为「放行并把该类请求写入 scope、version+1」（立即生效），与 §2.5 修约冷却期自相矛盾；本规格统一走 pending→cooling→effective 标准流程（一致性优先）。代价：冷却窗口内同类请求可能再次 unclear，MVP 接受。
5. **审计公共头扩展**：PRD 早期稿公共字段仅 `seq/ts/type`；本规格按任务要求加入 `contract_id/version`，使审计流可按契约与版本切片；`seq` 保留并定义为行号，作为卡片 `unlocked_by` 与 `unclear_resolve.seq_ref` 的交叉引用锚点。
6. **amend 事件细化为转移流**：PRD 的 amend 事件是单条 `{from_v, to_v}`；本规格按修约状态机每次转移各记一条（pending/cooling/effective/revoked），「发起→回心转意/生效」全程可审计，支撑 Q&A 演示叙事。
7. **tag_suggestion 词表外置**（tags.json）：PRD §2.3 未指定词表存放位置；外置后用户自定义标签不动 schema。同时以不变量 I4 从机制上钉死「标签永不参与状态机」——这是对内部评审「三标签机制是死的、且与『AI 不做道德诊断』铁律相冲突」的回应：标签权完全归用户，AI 只给建议。
8. **auto 断言执行方式留为实现自由度**：PRD §3.1 Won't 明确不做真实沙箱，则 acceptance 的 auto `cmd` 在 MVP 中由后端宿主直跑（限时）或降级为人工标记通过；契约 schema 保留 auto/manual 两型不变，规格不锁死实现。
9. **存疑 30 分钟超时定为惰性触发**：PRD §2.6 定义了「30min 超时默认拒绝」但未定义由谁触发；本规格定为后端惰性判定（触碰该请求或相关资源时），落审计为 `unclear_resolve{resolved_by:"timeout", resolution:"deny"}`，同样生成停车场卡片。
10. **verdict 事件增加 `fallback` 字段**：区分「模型判 unclear」与「模型不可用降级 unclear」（PRD §2.2 有此语义、无对应字段），供演示与一致性测试分别统计。

### A.2 与 PRD v1.1 五项拍板的对照核验

本规格起草于 PRD v1，定稿时 repo 已合入 PRD v1.1（五项拍板）。逐项核验：

| 拍板 | 本规格落点 | 结论 |
|---|---|---|
| ① 主叙事口径收窄（范围蔓延治理） | 纯叙事层，不涉及契约/接口结构 | 无冲突 |
| ② 标签开放元数据：词表独立配置文件、永不参与状态机 | §2.4 词表外置（出厂 tags.json，格式随选型）+ 不变量 I4 | 完全一致 |
| ③ 改约冷却期：契约名二次确认→冷却→回心转意→生效 version+1，演示 15-30s | §6.2 四态机（pending→cooling→effective/revoked）+ §5.4/5.5/5.6 三个端点 + `cooling_minutes` 可配置 | 完全一致（本规格的 pending 态即「二次确认前」） |
| ④ 魔搭通道（Docker 先行、裁决器 base_url 兼容开关） | 部署层；verdict 事件 `model` 字段可承载多供应商 | 无冲突 |
| ⑤ 消融对照页三栏实时同源 | 新增 §5.12：对照管线只返回假设结果、**永不执行**、不进审计（并入不变量 I1） | 本规格补充关键红线 |

---

*版本：v0.1 · 2026-09-05 · 冻结前唯一契约源 · 变更需全队确认*

---

*版本：v0 · 2026-09-05 · 冻结前唯一契约源 · 变更需全队确认*
