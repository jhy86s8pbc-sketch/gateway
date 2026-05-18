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
