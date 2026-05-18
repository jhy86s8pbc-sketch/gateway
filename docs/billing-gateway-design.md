# 计费网关设计文档（公有云 / 私有云 SaaS 双模式）

## 1. 背景与目标

平台对外提供多种可计费能力（短信、LLM Token、语音、图像、知识库检索等），既要支持**公有云多租户 SaaS**，也要支持**私有云独立部署**（含完全断网/气隙环境）。

设计目标：

1. **统一计量、统一计费**：所有可计费动作最终归一到「**点数（Point）**」这一抽象单位。
2. **解耦计量与定价**：业务侧只上报"用了多少原始单位"，由计费网关负责"折算成多少点数"。
3. **公有云在线扣费、私有云离线核销**两种模式共用同一套数据模型，便于私转公或混合部署。
4. **不可篡改、可审计、可对账**：断网环境也能在恢复连接后完成完整对账。

---

## 2. 核心概念

| 概念 | 说明 |
|---|---|
| Point（点数） | 平台内部统一计价单位，1 点 = X 元（X 由套餐决定） |
| Meter（计量项） | 一种可计费的原子能力，例如 `sms.cn`、`llm.gpt4.input_token`、`tts.zh` |
| Pricing Plan（计费方案） | Meter → Point 的换算规则，可按租户/版本/时间维度生效 |
| Quota（额度） | 租户当前可用的点数余额（公有云）或 License 内剩余点数（私有云） |
| Usage Record（用量记录） | 一次调用产生的原始计量数据，签名后落账 |
| Settlement（结算单） | 一段周期内按点数汇总的账单 |

---

## 3. 总体架构

```
       ┌───────────────────────────────────────────────────────────┐
       │                       业务网关层                          │
       │   (API Gateway / SDK / OpenAPI - 短信、LLM、TTS 等)       │
       └──────────────────────┬────────────────────────────────────┘
                              │ 1. 预检 (Pre-check)
                              │ 2. 异步上报 (Async report)
                              ▼
       ┌───────────────────────────────────────────────────────────┐
       │                     计费网关 (Billing GW)                 │
       │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌───────────┐  │
       │  │ 配额服务 │  │ 计量服务 │  │ 计价引擎 │  │ 结算&账单 │  │
       │  │  Quota   │  │  Meter   │  │  Pricer  │  │ Settlement│  │
       │  └──────────┘  └──────────┘  └──────────┘  └───────────┘  │
       │                          │                                │
       │                          ▼                                │
       │                  ┌───────────────┐                        │
       │                  │  事件日志(WAL)│  ← 不可变追加           │
       │                  └───────────────┘                        │
       └──────────────────────┬────────────────────────────────────┘
                              │
              ┌───────────────┴───────────────┐
              ▼                               ▼
      公有云：实时余额扣减           私有云：License 离线核销
      (强一致 / Redis + DB)         (本地累计 + 周期上报/导出)
```

---

## 4. 点数（Point）统一计费模型

### 4.1 为什么必须用点数

- 业务侧的原始单位差异巨大：短信按"条"，LLM 按"Token"，语音按"秒"，图像按"张/分辨率"。
- 直接用"元"会因汇率、税率、套餐折扣等导致计费逻辑爆炸。
- 点数作为中间抽象单位：
  - **业务方只关心 Meter**；
  - **商务方只关心点数 ↔ 钱的换算**；
  - **客户只关心剩余点数**。

### 4.2 换算规则（Pricing Rule）

每个 Meter 定义一个换算函数，统一形式：

```
points = ceil( (raw_usage / unit) * rate * multiplier )
```

| 字段 | 含义 |
|---|---|
| `raw_usage` | 原始计量值（条数、token 数、秒数等） |
| `unit` | 计费基本单位（如 1000 tokens 为一个计费单元） |
| `rate` | 每单位消耗多少点数 |
| `multiplier` | 租户/版本/促销系数 |
| `ceil` | 取整策略（向上取整，避免零计费薅羊毛） |

#### 示例

| Meter | 原始单位 | 计费规则 | 说明 |
|---|---|---|---|
| `sms.cn.normal` | 1 条 | 1 点 / 条 | 国内普通短信 |
| `sms.cn.marketing` | 1 条 | 2 点 / 条 | 营销短信加价 |
| `sms.intl.<region>` | 1 条 | 5~30 点 / 条 | 按地区分档 |
| `llm.gpt4o.input` | 1 token | 0.3 点 / 1K tokens | 长 prompt 友好 |
| `llm.gpt4o.output` | 1 token | 1.2 点 / 1K tokens | 输出更贵 |
| `llm.claude.sonnet.input` | 1 token | 0.3 点 / 1K tokens | |
| `llm.claude.sonnet.output` | 1 token | 1.5 点 / 1K tokens | |
| `embedding.text` | 1 token | 0.02 点 / 1K tokens | |
| `tts.zh` | 1 秒 | 0.5 点 / 秒 | |

> **关键设计**：换算表通过 **Pricing Plan 版本号**管理，所有 Usage Record 都要带 `pricing_version`，保证历史账单可重算、可追溯。

### 4.3 一次调用的计费流水

```
1. 用户调用 /v1/chat/completions
2. 网关 Pre-check：估算 max_points（input_tokens 已知，output 按 max_tokens 上界估算）
                  → 检查 Quota.available >= max_points，否则 402
                  → 冻结 max_points (hold)
3. 业务执行，得到真实 output_tokens
4. 计算 actual_points = price(input) + price(output)
5. Release hold，扣减 actual_points
6. 写入 Usage Record（含签名）→ WAL
7. 异步聚合到 Settlement
```

---

## 5. 公有云模式

### 5.1 关键特性

- **强一致余额**：Redis（热）+ MySQL/PG（冷）双写，账户级行锁防并发超扣。
- **预扣 + 冲正**：高并发场景下用 hold/commit/cancel 三段式，避免回滚噪声。
- **实时充值**：对接支付/订单系统，充值后即时入账。
- **超额策略**：可配置硬拒绝、透支额度、欠费降级。

### 5.2 数据流

```
SDK ─► API GW ─► Billing GW.Quota (check + hold)
                       │
                       ▼
                  业务执行
                       │
                       ▼
              Billing GW.Meter.commit
                       │
              ┌────────┴────────┐
              ▼                 ▼
         Quota.deduct     UsageRecord(WAL) ─► Kafka ─► 数仓/账单
```

---

## 6. 私有云模式（重点：含完全断网场景）

私有云的核心矛盾：**没有中心服务可以实时扣费**，但仍要做到：
- 客户不能无限白嫖（必须有授权上限）；
- 厂商最终能拿到准确用量做结算；
- 客户能自审计、不被"算多了"。

### 6.1 三档部署形态

| 形态 | 网络 | 计费方式 |
|---|---|---|
| **A. 在线私有云** | 出站可达计费中心 | 周期性（如每分钟）异步上报，准实时扣费 |
| **B. 半离线私有云** | 仅允许定时白名单出站 | 每日/每周打包上传 Usage Bundle |
| **C. 气隙私有云** | 完全物理隔离 | License 预付点数 + 手工导出/导入对账包 |

### 6.2 License 文件（所有私有云形态的基石）

License 是一个**离线签名文件**，由计费中心用私钥签发，私有云用公钥校验：

```jsonc
{
  "license_id": "LIC-2026-0001",
  "tenant": "acme-corp",
  "issued_at": "2026-05-18T00:00:00Z",
  "valid_from": "2026-05-18",
  "valid_until": "2027-05-17",
  "edition": "enterprise",
  "quota": {
    "total_points": 10000000,           // 本期授权总点数
    "reset_policy": "none",             // none | monthly | yearly
    "overdraft_points": 0               // 允许透支
  },
  "meters_allowed": ["sms.*", "llm.*", "tts.zh"],
  "pricing_version": "v2026.05",        // 锁定的计价表版本
  "reporting": {
    "mode": "online | bundle | airgap",
    "endpoint": "https://billing.example.com/v1/usage",
    "interval_sec": 3600
  },
  "hardware_fingerprint": "sha256:...", // 防止 License 跨机迁移
  "signature": "base64(RSA/Ed25519)"
}
```

> **要点**：私有云内的计费网关**只信任 License 文件**，所有扣费都从 License 内的 `total_points` 余额扣减。

### 6.3 私有云内部数据流

```
        ┌────────────────────────────────────────────────┐
        │             私有云本地 Billing GW              │
        │                                                │
        │  License Loader ──► LocalQuotaStore (BadgerDB) │
        │                          ▲                     │
        │  业务请求 ───► Meter ────┘                     │
        │                  │                             │
        │                  ▼                             │
        │           UsageRecord (本地 WAL，仅追加)       │
        │                  │                             │
        │                  ▼                             │
        │   ┌─────────────────────────────┐              │
        │   │  Reporter (按形态切换)      │              │
        │   │  - online: HTTPS 上报       │              │
        │   │  - bundle: 打包 + 签名导出  │              │
        │   │  - airgap: 手工导出 .ubp    │              │
        │   └─────────────────────────────┘              │
        └────────────────────────────────────────────────┘
```

### 6.4 不可篡改的本地账本

- **WAL 仅追加**：每条 UsageRecord 落盘后立即 fsync，按小时滚动文件。
- **Merkle Hash Chain**：每条记录 `hash_i = H(hash_{i-1} || record_i)`，每日生成 Daily Root Hash。
- **TPM/HSM 可选**：每日 Root Hash 用本地 TPM 签名，防止运维人员篡改账本。
- **客户可审计**：客户可独立运行 `audit-cli` 校验本地账本完整性。

### 6.5 气隙环境（C 类）对账流程

```
[私有云侧]
1. billing-cli export --from 2026-05-01 --to 2026-05-31 \
       --out usage-2026-05.ubp
   → 产物：Usage Bundle Package (.ubp)
       - 加密 (AES-256-GCM, 密钥来自 License)
       - 含 Merkle Root + TPM 签名
       - 含设备指纹

2. 客户用 U 盘把 .ubp 带出隔离区

[计费中心侧]
3. billing-admin import usage-2026-05.ubp
   → 校验签名/指纹/Merkle 链 → 入库
   → 生成结算单
   → 生成 License Refresh File（新一期点数 + 已确认的扣减游标）

4. 客户把 License Refresh File 带回隔离区

[私有云侧]
5. billing-cli license refresh license-refresh.lrf
   → 校验签名 → 更新 LocalQuotaStore
   → 标记已对账区间为"已结算"，从 WAL 截断
```

### 6.6 防作弊与时钟保护

- **设备指纹**：CPU / 主板 / 网卡 / 部署 UUID 组合，License 绑定后不可迁移。
- **单调时钟**：本地维护"曾经见过的最大时间戳"，防止运维人员回拨系统时间重复消费。
- **License 反重放**：每张 License 有 `nonce`，刷新文件按序号递增，旧文件无法重新激活。
- **业务侧 Pre-check 必须经过本地 Quota**：禁止业务模块绕过计费网关直连下游。

---

## 7. 数据模型（核心表）

```sql
-- 计量项注册表
CREATE TABLE meters (
  code           VARCHAR(64) PRIMARY KEY,    -- e.g. llm.gpt4o.input
  display_name   VARCHAR(128),
  raw_unit       VARCHAR(32),                -- token / sms / second
  category       VARCHAR(32)                 -- llm / sms / tts ...
);

-- 计价方案（版本化）
CREATE TABLE pricing_plans (
  id             BIGINT PRIMARY KEY,
  version        VARCHAR(32),                -- v2026.05
  meter_code     VARCHAR(64),
  unit_size      INT,                        -- 1000 (per 1K tokens)
  points_per_unit DECIMAL(18,6),
  effective_from TIMESTAMP,
  effective_to   TIMESTAMP NULL
);

-- 租户配额（公有云：动态变化；私有云：等同 License 镜像）
CREATE TABLE quotas (
  tenant_id      VARCHAR(64) PRIMARY KEY,
  total_points   DECIMAL(20,4),
  used_points    DECIMAL(20,4),
  held_points    DECIMAL(20,4),
  overdraft      DECIMAL(20,4),
  updated_at     TIMESTAMP
);

-- 用量流水（核心，最终都会同步到中心）
CREATE TABLE usage_records (
  id             UUID PRIMARY KEY,
  tenant_id      VARCHAR(64),
  meter_code     VARCHAR(64),
  raw_usage      DECIMAL(20,4),
  points         DECIMAL(20,4),
  pricing_version VARCHAR(32),
  request_id     VARCHAR(64),
  occurred_at    TIMESTAMP,
  prev_hash      CHAR(64),
  this_hash      CHAR(64),
  deployment_id  VARCHAR(64),                -- 私有云实例标识
  INDEX (tenant_id, occurred_at)
);
```

---

## 8. 接口草案

### 8.1 业务侧上报（内部）

```http
POST /internal/v1/usage/commit
{
  "tenant_id": "acme",
  "request_id": "req-abc",
  "items": [
    {"meter": "llm.gpt4o.input",  "raw": 1532},
    {"meter": "llm.gpt4o.output", "raw":  481}
  ],
  "hold_id": "h-xxxx"           // pre-check 阶段拿到
}
→ 200 { "points_charged": 712, "balance": 9982341 }
```

### 8.2 私有云 ↔ 计费中心

```http
POST /v1/license/issue          # 签发 / 续期
POST /v1/usage/bundle           # 上传打包用量
GET  /v1/license/refresh        # 拉取刷新文件
```

气隙模式下，以上接口均通过 `.ubp` / `.lrf` 文件离线流转。

---

## 9. 失败处理与降级

| 场景 | 公有云策略 | 私有云策略 |
|---|---|---|
| 计费网关短暂不可达 | 业务侧本地缓存 hold，恢复后 commit；超时拒绝 | 本地 WAL 持续写入，不受中心可达性影响 |
| 余额不足 | 返回 402 + 升级提示 | 阻断 + License 即将耗尽预警 |
| 时钟漂移 > 阈值 | NTP 强制校准；记录但不阻断 | 阻断扣费，要求人工介入 |
| License 过期 | N/A | 进入 7 天宽限期（只读+预警），之后全量阻断 |
| WAL 损坏 | 从 binlog 回放 | 启动失败，需从最近 daily root 恢复，差量重算 |

---

## 10. 安全要点

1. **签名密钥**：License 与 Refresh 文件使用 Ed25519，私钥仅在计费中心 HSM 内。
2. **传输**：在线模式 mTLS，离线模式包内 AES-256-GCM + 完整性 MAC。
3. **最小权限**：私有云内 Billing GW 服务账户对 WAL 目录只增不删（chattr +a）。
4. **审计接口对客户开放**：客户可在私有云内独立验证账本，建立信任。
5. **隐私**：UsageRecord 默认不含业务原文，仅记录 meter + 数量 + request_id 摘要。

---

## 11. 后续工作 (TBD)

- [ ] Pricing Plan 的灰度发布与回滚机制
- [ ] 多币种结算与汇率快照
- [ ] 大客户私有云 → 公有云联邦计费（混合云用量合并账单）
- [ ] 与 OpenTelemetry Metrics 对接，提供消费可视化大盘
- [ ] License 自助续费门户（含离线签发 + 邮寄/U 盘交付 SOP）

---

# 附录 A：Hold / Commit / Cancel 状态机详解

## A.1 为什么需要三段式

LLM 类调用最难计费的地方在于：**调用开始时不知道最终消耗多少**（output_tokens 是结果决定的）。如果先调用后扣费，并发场景下就会出现"100 个请求同时只剩 50 点"的超扣。

三段式（Two-Phase Charging）解决：

```
Pre-check ──hold──► 业务执行 ──commit(actual)──► 真实扣费
                       │
                       └──cancel──► 释放预扣（失败/超时）
```

## A.2 状态机定义

```
              ┌─────────────┐
              │   INITIAL   │
              └──────┬──────┘
            hold(max_points)
                    │  余额不足 → REJECTED
                    ▼
              ┌─────────────┐
   ┌──────────┤    HELD     ├──────────┐
   │          └──────┬──────┘          │
commit(actual)       │ TTL 超时         cancel
   │            (e.g. 5 min)            │
   ▼                ▼                   ▼
┌─────────┐   ┌──────────┐        ┌──────────┐
│COMMITTED│   │  EXPIRED │        │ CANCELED │
└─────────┘   └─────┬────┘        └──────────┘
                    │ 自动 cancel
                    ▼
                ┌──────────┐
                │ CANCELED │
                └──────────┘
```

| 状态 | 含义 | 影响余额字段 |
|---|---|---|
| `HELD` | 预扣中，可用余额减少但 used 未变 | `held_points += max` |
| `COMMITTED` | 真实扣费完成 | `held_points -= max; used_points += actual` |
| `CANCELED` | 主动取消或失败 | `held_points -= max` |
| `EXPIRED` | TTL 到期被回收 | 同 CANCELED + 触发告警 |
| `REJECTED` | hold 阶段就被拒（余额不足/Meter 未授权） | 无影响 |

## A.3 余额计算公式

```
available_points = total_points - used_points - held_points + overdraft_points
```

所有读取都走该公式，hold 阶段对客户表现为"已扣"，避免并发请求看到虚高余额。

## A.4 hold 配额估算

| Meter 类型 | max_points 估算策略 |
|---|---|
| SMS | 精确：条数 × 单价（无歧义） |
| LLM input | 精确：tokenizer 计算 prompt token |
| LLM output | 悲观：按请求中 `max_tokens` 上界计算 |
| TTS / 语音 | 按上游返回的预估时长 + 20% 缓冲 |
| 流式调用 | 按 chunk 累加多次小额 hold（见 A.7） |

## A.5 commit 阶段的差额处理

```
delta = actual - max_held
if delta > 0:          # 实际比预估多（极少见，需严格防护）
    if available_points < delta:
        → 写 OVERDRAFT_EVENT，按策略：
          (a) 透支允许：扣到负数，标记欠费
          (b) 透支禁止：返回 commit_partial，业务侧记录差额
else:                  # 实际小于预估，常见
    退还 (max_held - actual)
```

## A.6 接口规范

```http
# 1. 预扣
POST /internal/v1/hold
{
  "tenant_id": "acme",
  "request_id": "req-abc",
  "estimates": [
    {"meter": "llm.gpt4o.input",  "raw": 1500},
    {"meter": "llm.gpt4o.output", "raw": 4000}   // max_tokens
  ],
  "ttl_sec": 300
}
→ 200 {
  "hold_id": "h-9f3b...",
  "held_points": 5300,
  "expires_at": "2026-05-18T08:05:00Z"
}
→ 402 {"code": "INSUFFICIENT_BALANCE", "available": 200}

# 2. 真实提交
POST /internal/v1/commit
{
  "hold_id": "h-9f3b...",
  "actuals": [
    {"meter": "llm.gpt4o.input",  "raw": 1532},
    {"meter": "llm.gpt4o.output", "raw":  481}
  ]
}
→ 200 {"charged_points": 712, "refunded_points": 4588, "balance": 9982341}

# 3. 取消
POST /internal/v1/cancel
{ "hold_id": "h-9f3b...", "reason": "upstream_5xx" }
→ 200 {"released_points": 5300, "balance": 9987641}
```

## A.7 流式调用 (SSE / streaming) 适配

```
HELD(burst_1) ──commit──► HELD(burst_2) ──commit──► ... ──finalize
       │
   每生成 N 个 token 或每 K 秒滚动一次 hold/commit
```

策略：
- 首次 hold 取 `min(max_tokens, 512 token)` 这种小额度；
- 每流出一段后 `partial_commit` 并立即 `hold` 下一段；
- 连接断开时已 commit 的不退，正在 hold 的走 cancel。

## A.8 幂等与重试

- `hold_id` / `request_id` 均为幂等键，PG 唯一索引强约束。
- 重复 commit 同一 hold_id → 返回首次结果，不二次扣费。
- 业务侧因网络问题不确定 commit 是否成功 → 用 `GET /internal/v1/hold/{id}` 查询最终态再决定重放。

## A.9 并发与持久化

- 单租户 hold 用 Redis Lua 脚本保证 `check + decr + write` 原子。
- Lua 脚本对应记录同步 append 到 PG `hold_log` 表（异步双写）。
- Redis 故障兜底：降级到 PG 行锁，QPS 下降但不丢账。
- TTL 兜底：Redis ZSET 按 `expires_at` 排序，单独 reaper 协程每 10s 扫描自动 cancel。

---

# 附录 B：Merkle 哈希链校验细节（私有云不可篡改账本）

## B.1 目标

让客户能够**独立验证**：
1. 任意一条 UsageRecord 没有被增删改；
2. 上报给计费中心的 daily root，与本地账本的 daily root 完全一致；
3. 即便 root 系统管理员，也无法静默删除某条记录而不被发现。

## B.2 数据结构

### 链式哈希（Hash Chain）

每条记录在 WAL 中带 `prev_hash` 与 `this_hash`：

```
record_i = {
  ...usage fields,
  prev_hash: hash_{i-1},
  this_hash: SHA256(canonical_json(usage_fields) || prev_hash)
}
```

初始记录 `prev_hash = SHA256("genesis|" || deployment_id || license_id)`。

> **canonical_json**：键名按字典序排序、数字定型号、UTF-8 NFC 归一，消除序列化歧义。

### 当日 Merkle 树

每个 UTC 日凌晨，将当日所有记录的 `this_hash` 作为叶子，构建标准二叉 Merkle Tree：

```
              Root_d (DailyRoot)
             /         \
          H(L||R)    H(L||R)
          /   \       /    \
        h1   h2     h3     h4   ← this_hash of each usage record
```

奇数叶子用 `H(last || last)` 填充（避免引入空值）。

### Daily Anchor 文件

每日生成 `anchor-YYYYMMDD.json`：

```jsonc
{
  "deployment_id": "...",
  "date": "2026-05-18",
  "record_count": 184213,
  "first_record_id": "uuid",
  "last_record_id":  "uuid",
  "merkle_root": "sha256:abc...",
  "prev_anchor_hash": "sha256:def...",   // 跨日链，形成 anchor-of-anchors
  "signed_by_tpm": "base64(sig)",         // 可选：TPM 远程认证
  "signed_by_billing_key": "base64(sig)"  // 必选：License 内派生密钥
}
```

## B.3 写入流程（保证不可篡改）

```
business commit
     │
     ▼
1. 序列化 record（canonical_json）
2. 在写锁内读取 last_hash → 计算 this_hash
3. append 到 WAL 段文件 (open with O_APPEND | O_DSYNC)
4. fsync(fd)
5. 更新内存 last_hash
6. 返回成功
```

**关键约束**：
- WAL 文件使用 `chattr +a`（ext4 append-only），即便 root 也只能追加；
- WAL 段按小时滚动，旧段 `chattr +i`（immutable）；
- 每日生成 anchor 后，当日所有段文件归档到 `archive/`，权限改为 0400。

## B.4 客户独立审计 CLI

```bash
$ billing-cli audit verify --from 2026-05-01 --to 2026-05-18

[OK] 段文件完整性: 432/432 segments, no gap
[OK] 哈希链连续性: 18,452,317 records, no break
[OK] Daily anchors:  18 days, all signatures valid
[OK] 与中心对账: 已结算 12 天，root 完全一致；未结算 6 天，本地 pending
```

底层执行的事：
1. 顺序读所有 WAL 段，重算 `this_hash`，与磁盘记录比对；
2. 重建每日 Merkle Tree，比对 `anchor.merkle_root`；
3. 验证 anchor 跨日链：`H(prev_anchor) == anchor.prev_anchor_hash`；
4. （在线/半离线模式）拉取中心侧已确认的 root 列表做交叉验证。

## B.5 Merkle Proof（针对单条记录）

客户对某条具体扣费有疑问时：

```bash
$ billing-cli audit proof --record-id uuid-xxx
{
  "record": {...},
  "merkle_proof": [
    {"position": "right", "hash": "..."},
    {"position": "left",  "hash": "..."},
    ...
  ],
  "merkle_root": "sha256:abc...",
  "anchor_signature": "base64(...)"
}
```

第三方拿到该 proof，无需访问私有云本地数据，即可证明该记录确实存在于某日账本中。

## B.6 篡改检测场景

| 攻击行为 | 检测点 |
|---|---|
| 删除某条已写入记录 | 下一条 prev_hash 校验失败 |
| 修改 raw_usage 数值 | this_hash 不匹配 |
| 偷偷重排序段文件 | segment 编号 + 起止 record_id 比对失败 |
| 全部重算哈希链 | 与 anchor 已签名的 root 对不上 |
| 篡改 anchor 文件 | TPM/License-key 签名验证失败 |
| 回滚整个 WAL 目录到昨天 | anchor 跨日链断裂；中心侧已收到的 root 对不上 |

## B.7 恢复策略

- **段文件磁盘损坏**：从最近一份未签名归档 + 中心侧已收到的 anchor 起点，差量重算到该段起点；缺失区间标记为"不可恢复"，触发人工对账。
- **anchor 文件丢失**：可由 WAL 重算重建，但**不可签名**——需要计费中心配合补签一份"补发 anchor"，记录在案。
- **License 密钥轮换**：每张 License 都有自己的 anchor 签名密钥，新旧 License 期间的 anchor 各自独立验证。

---

# 附录 C：License 续期与刷新 SOP

## C.1 License 生命周期

```
[未签发] ──issue──► [有效期内]
                       │
                       │ 即将到期 (T-30d)
                       ▼
                  [续期提醒]
                       │
            ┌──────────┴──────────┐
            ▼                     ▼
        正常续期               未续期
            │                     │
            ▼                     ▼
       [新 License 生效]    [过期] ──T+7d──► [全量阻断]
                                  ▲
                                  └── 7 天宽限期：只读+预警
```

## C.2 角色与产物

| 角色 | 职责 | 产物 |
|---|---|---|
| 计费中心 (Issuer) | 持有 HSM 私钥，签发文件 | `*.license`, `*.lrf` |
| 客户管理员 | 触发续期、运送介质、导入文件 | 提交 `*.ubp` |
| 私有云 Billing GW | 校验/装载/汇报 | 状态指标、告警 |
| 商务 / 销售 | 合同对齐、定价方案确认 | 续期订单 |

## C.3 在线模式续期 SOP（A 类）

```
T-30d  系统自动检测 License 剩余期限 < 30 天 → 推送续期工单
T-15d  客户支付 / 商务对齐 pricing_version
T-7d   计费中心生成新 License，自动推送到私有云 → 双 License 并存
T0     旧 License 到期，自动切换；监控验证流量平稳
T+1d   归档旧 License，写入审计日志
```

技术细节：私有云 Billing GW 启动时加载**所有**未过期 License，按 `valid_from` 排序使用，自然过渡无需重启。

## C.4 气隙模式续期 SOP（C 类）

最严格场景，全程人工 + 介质：

```
┌──────────── 客户侧 ────────────┐    ┌──────────── 厂商侧 ────────────┐
                                                                       
T-30d  系统预警 License 余量/期限                                       
                                                                       
T-25d  客户管理员执行：                                                 
       billing-cli export usage \                                       
         --since last-settled \                                         
         --out usage-2026-05.ubp                                        
       (包含本期所有用量 + Merkle root + TPM 签名)                      
                                                                       
T-24d  U 盘运送 ─────────────────►   T-24d  接收 .ubp                   
                                            校验签名/指纹/Merkle 链     
                                            导入计费系统                
                                            生成结算单 → 客户付款        
                                                                       
                                     T-20d  生成 License Refresh:        
                                            - 续期 / 加点 / 调价        
                                            - 含本期已确认扣减游标       
                                            - HSM 签名                  
                                                                       
T-15d  接收 .lrf ◄───────────────── 邮寄 / 派送                         
                                                                       
T-14d  客户管理员执行：                                                 
       billing-cli license refresh \                                    
         license-refresh-2026-06.lrf                                    
       → 校验 → 更新 LocalQuotaStore                                    
       → 截断已结算的 WAL（归档到只读区）                                
       → 新 License 立即可用                                            
                                                                       
T-13d  执行 billing-cli audit verify 全量校验                           
       上传校验报告确认                                                 
└────────────────────────────────┘    └────────────────────────────────┘
```

## C.5 License Refresh 文件格式（`.lrf`）

```jsonc
{
  "lrf_id": "LRF-2026-0612-0001",
  "license_id": "LIC-2026-0001",
  "seq": 7,                         // 严格递增，防重放
  "issued_at": "2026-06-12T03:00:00Z",
  "actions": [
    {
      "type": "ack_usage",
      "until_anchor": "sha256:abc...",   // 中心已确认到这一天的 anchor
      "settled_points": 8429311          // 中心算出的已结算点数
    },
    {
      "type": "extend_validity",
      "new_valid_until": "2027-06-11"
    },
    {
      "type": "topup",
      "add_points": 5000000              // 新增点数
    },
    {
      "type": "update_pricing",
      "new_pricing_version": "v2026.06"  // 价格调整，下一秒生效
    }
  ],
  "previous_lrf_hash": "sha256:...",  // 形成续期事件链
  "signature": "base64(Ed25519)"
}
```

私有云接收逻辑：

```python
def apply_refresh(lrf):
    verify_signature(lrf, license.issuer_pubkey)
    assert lrf.license_id == license.id
    assert lrf.seq == license.last_lrf_seq + 1     # 严格连续
    assert lrf.previous_lrf_hash == sha256(license.last_lrf)

    for action in lrf.actions:
        if action.type == "ack_usage":
            # 关键：必须先核对本地 anchor 与中心一致才能截断 WAL
            local_root = compute_anchor_up_to(action.until_anchor_date)
            assert local_root == action.until_anchor
            mark_settled(action.until_anchor)
            truncate_wal_before(action.until_anchor)
            assert local_settled_points == action.settled_points
        elif action.type == "extend_validity":
            license.valid_until = action.new_valid_until
        elif action.type == "topup":
            license.total_points += action.add_points
        elif action.type == "update_pricing":
            pricing_store.load(action.new_pricing_version)

    license.last_lrf = lrf
    license.last_lrf_seq = lrf.seq
    persist(license)
```

## C.6 异常 SOP

| 异常 | 处理 |
|---|---|
| `.ubp` 在路途丢失 | 客户重新导出（同 seq 内容确定），无副作用 |
| `.lrf` 在路途丢失 | 计费中心可重新签发**同 seq** lrf；旧 lrf 一旦使用即作废 |
| `.lrf` seq 不连续 | 私有云拒绝装载，要求补发缺失序号的 lrf |
| 本地 root 与中心 root 不一致 | 阻断 ack_usage 动作，进入"账目争议"流程：双方人工对账，可能由 anchor 维度逐日定位差异 |
| 客户长期不导出（>90 天） | 计费中心标记"失联"，触发商务介入；私有云本身仍能消费到 License 余量耗尽 |
| 客户主动退订 | 签发"终止 lrf"：禁止新增扣费，仅允许导出最后一份 ubp 完成清算 |

## C.7 度量与告警（私有云本地）

| 指标 | 阈值 | 告警 |
|---|---|---|
| `license.remaining_days` | < 30 / < 7 | warning / critical |
| `license.remaining_points / daily_avg_use` | < 30 天用量 | warning |
| `wal.unsettled_days` | > 14 | warning（提醒导出） |
| `wal.unsettled_days` | > 60 | critical |
| `lrf.seq_gap` | > 0 | critical（缺失续期文件） |
| `audit.verify_failed` | 任意 | critical（立刻通知客户与厂商） |

---

# 附录 D：参考时序图

## D.1 公有云一次 LLM 调用完整链路

```
Client ─► APIGW ─► LLM Service ─► Billing GW                Redis     DB
   │                                  │                       │       │
   │── POST /chat ────────────────►   │                       │       │
   │                                  │── hold(max=5300) ──►  │ Lua   │
   │                                  │                       │ atom  │
   │                                  │◄── hold_id ──────────│        │
   │                                  │                       │── async ─►│
   │                                  │── invoke upstream ──► (LLM)
   │                                  │◄── result (tokens=481) ────────
   │                                  │── commit(712) ────►   │ Lua   │
   │                                  │◄── balance ──────────│        │
   │◄── 200 OK + Usage ───────────────│                       │── async ─►│
                                                                       (WAL)
```

## D.2 气隙私有云续期完整时序

```
私有云 BGW             U盘介质            计费中心
   │                     │                   │
   │ T-25: export ──►    │                   │
   │                     │ ──物理运送──►     │
   │                     │                   │ T-24: import + verify
   │                     │                   │ T-22: generate settlement
   │                     │                   │ T-20: sign LRF
   │                     │ ◄──物理运送──     │
   │                     │                   │
   │ T-15: refresh ◄──   │                   │
   │ verify + apply      │                   │
   │ truncate WAL        │                   │
   │ continue serving    │                   │
   │                     │                   │
```

