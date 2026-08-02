# Tokly 数据库 Schema（规范 DDL）

本文是 Tokly 本地仓库的**规范 schema**，M0.5 冻结门交付物（见 [路线图](路线图.md)）。契约语义见 [ADR-0002](adr/0002-数据架构.md)（收敛契约、成本所有权、水位线三类源语义）；配额快照语义见 [ADR-0008](adr/0008-配额窗口.md)。依据：[审查-2026-08-02-二轮](research/审查-2026-08-02-二轮.md) §2 schema 清单。

**全局类型约定（硬约束）**：

- 时间一律 `INTEGER` **epoch_ms**（UTC）；
- 金额一律 `INTEGER` **micro-USD**（10⁻⁶ 美元），禁止 REAL 存金额（聚合漂移）；
- token 一律非负 `INTEGER`，CHECK 强制；token 维度**不可得时写 0 并用 `token_coverage` 标注**——0 不冒充"已确认没有"；
- 文本一律 UTF-8；DDL 由迁移工具按 `schema_migrations` 顺序应用，禁止手工改库。

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;

-- ============================================================
-- schema_migrations：迁移记录。职责：schema_version 协商
-- （进程模型握手比对，版本漂移拒开）的权威来源。
-- ============================================================
CREATE TABLE schema_migrations (
  version     INTEGER PRIMARY KEY,        -- 单调递增迁移号
  name        TEXT NOT NULL,
  applied_at  INTEGER NOT NULL            -- epoch_ms
);

-- ============================================================
-- source_instances：源实例。职责：把 源类型 × 账号 × 机器 ×
-- 源数据根 映射为稳定实例 ID——事件身份 (source_instance_id,
-- event_key) 的一半；跨实例同 eventKey 不互相去重。
-- ============================================================
CREATE TABLE source_instances (
  id            TEXT PRIMARY KEY,         -- 稳定 ID = UNIQUE 四元组的确定性哈希（PK 兜底 UNIQUE 中 account 为 NULL 时的去重）
  source        TEXT NOT NULL CHECK (source IN
                  ('claude-code','codex','opencode','gemini-cli','copilot','cursor')),
  account       TEXT,                     -- 账号标识；NULL = 单账号/未知
  machine       TEXT NOT NULL,            -- 机器标识（安装期生成并持久化）
  data_root     TEXT NOT NULL,            -- 源数据根（规范化路径 / API 端点 base）
  format_version TEXT,                    -- 源格式代际（opencode Gen A/B、codex legacy/paginated…）
  first_seen_at INTEGER NOT NULL,
  last_seen_at  INTEGER NOT NULL,
  UNIQUE (source, account, machine, data_root)
);

-- ============================================================
-- usage_events：归一化事件主表，current projection。
-- 职责：每个 (source_instance_id, event_key) 一行的当前状态；
-- observation 永久保留（本表行 + conflict_observations 败方），
-- 缺席 ≠ 删除，显式删除证据才 tombstoned。
-- ============================================================
CREATE TABLE usage_events (
  -- 事件身份
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  event_key          TEXT NOT NULL,
  identity_quality   TEXT NOT NULL DEFAULT 'strong'
                     CHECK (identity_quality IN ('strong','weak')),
  -- revision 三元组：仅同 (origin_stream, stream_generation) 内可比
  origin_stream      TEXT NOT NULL,
  stream_generation  INTEGER NOT NULL,
  source_position    TEXT NOT NULL,       -- kind 由各源 spec 定义（offset/rowid/(time_updated,id)/页序）
  payload_hash       TEXT NOT NULL,       -- 源 payload 的 sha256
  normalizer_version INTEGER NOT NULL,
  observed_at        INTEGER NOT NULL,    -- 首次观测时刻 epoch_ms
  updated_at         INTEGER NOT NULL,    -- projection 最近变更时刻
  -- 归属与时间
  session_key        TEXT,                -- 源内会话键；API 聚合事件可 NULL
  parent_session_key TEXT,
  project_key        TEXT NOT NULL,
  project_label      TEXT,
  timestamp          INTEGER NOT NULL,    -- 事件发生 epoch_ms
  duration_ms        INTEGER CHECK (duration_ms IS NULL OR duration_ms >= 0),
  model              TEXT,                -- 未知即 NULL，禁止魔法值
  agent              TEXT,
  mode               TEXT,
  git_branch         TEXT,
  cli_version        TEXT,
  status             TEXT NOT NULL DEFAULT 'counted'
                     CHECK (status IN ('counted','excluded','tombstoned')),
  exclude_reason     TEXT,
  channel            TEXT,
  granularity        TEXT NOT NULL DEFAULT 'event'
                     CHECK (granularity IN ('event','session','day','month')),
  period_end         INTEGER,
  service_tier       TEXT,
  -- token（净值口径，四者互斥；output 恒含 reasoning）
  tokens_input           INTEGER NOT NULL DEFAULT 0 CHECK (tokens_input >= 0),
  tokens_output          INTEGER NOT NULL DEFAULT 0 CHECK (tokens_output >= 0),
  tokens_cache_read      INTEGER NOT NULL DEFAULT 0 CHECK (tokens_cache_read >= 0),
  tokens_cache_write_5m  INTEGER NOT NULL DEFAULT 0 CHECK (tokens_cache_write_5m >= 0),
  tokens_cache_write_1h  INTEGER NOT NULL DEFAULT 0 CHECK (tokens_cache_write_1h >= 0),
  tokens_reasoning       INTEGER NOT NULL DEFAULT 0 CHECK (tokens_reasoning >= 0),
  token_quality          TEXT NOT NULL DEFAULT 'native'
                         CHECK (token_quality IN ('native','estimated','partial')),
  token_coverage         TEXT,            -- JSON 数组：不可得/低覆盖维度清单
  -- 成本：native 侧 adapter 独占；computed 侧 pricing worker 独占（重扫不覆盖）
  cost_native_micro_usd   INTEGER,
  native_cost_kind        TEXT CHECK (native_cost_kind IS NULL OR native_cost_kind IN
                            ('provider-bill','tool-computed','premium-requests','ai-credits')),
  native_cost_provider    TEXT,           -- 'github-billing' / 'opencode:models.dev' / …
  cost_computed_micro_usd INTEGER,
  computed_cost_status    TEXT NOT NULL DEFAULT 'pending'
                          CHECK (computed_cost_status IN ('pending','computed','unmatched','skipped')),
  pricing_version         TEXT,           -- 计价所用定价表版本
  pricing_eligibility     TEXT NOT NULL DEFAULT 'standard'
                          CHECK (pricing_eligibility IN ('standard','estimated-tokens','disabled')),
  display_policy          TEXT NOT NULL DEFAULT 'computed-only' CHECK (display_policy IN
                            ('native-preferred','computed-preferred','both',
                             'native-only','computed-only','estimate-only')),
  raw_usage               TEXT,           -- 仅有损映射时存
  raw_schema              TEXT,           -- 源格式版本标识
  PRIMARY KEY (source_instance_id, event_key),
  CHECK (granularity = 'event' OR period_end IS NOT NULL),
  CHECK (cost_native_micro_usd IS NULL OR native_cost_kind IS NOT NULL)
);

-- ============================================================
-- conflict_observations：冲突观测。职责：同坐标/跨流同键不同
-- payload 的显式留证（禁止静默 last-write-wins）；败方
-- observation 永久保留，fold 可据此重建 projection。
-- ============================================================
CREATE TABLE conflict_observations (
  id               INTEGER PRIMARY KEY AUTOINCREMENT,
  source_instance_id TEXT NOT NULL,
  event_key        TEXT NOT NULL,
  conflict_kind    TEXT NOT NULL CHECK (conflict_kind IN
                     ('same-position-diff-payload','cross-stream-diff-payload','unresolved-semantics')),
  existing_origin_stream      TEXT,
  existing_stream_generation  INTEGER,
  existing_source_position    TEXT,
  existing_payload_hash       TEXT,
  incoming_origin_stream      TEXT NOT NULL,
  incoming_stream_generation  INTEGER NOT NULL,
  incoming_source_position    TEXT NOT NULL,
  incoming_payload_hash       TEXT NOT NULL,
  detail           TEXT,                  -- JSON：差异字段与数值（只存指标，不存内容）
  resolution       TEXT NOT NULL CHECK (resolution IN
                     ('kept-existing','replaced','both-kept','excluded')),
  observed_at      INTEGER NOT NULL,
  FOREIGN KEY (source_instance_id, event_key)
    REFERENCES usage_events (source_instance_id, event_key)
);

-- ============================================================
-- sessions：会话级聚合，派生表。职责：会话维度汇总展示；
-- 可从 usage_events 全量重建，dirty_since 持久标记驱动局部
-- 重算；空会话删除规则——重算后 counted 事件为 0 即 DELETE。
-- ============================================================
CREATE TABLE sessions (
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  session_key        TEXT NOT NULL,
  parent_session_key TEXT,
  project_key        TEXT NOT NULL,
  project_label      TEXT,
  started_at         INTEGER,
  ended_at           INTEGER,
  event_count        INTEGER NOT NULL DEFAULT 0,      -- 仅 status='counted'
  tokens_input           INTEGER NOT NULL DEFAULT 0,
  tokens_output          INTEGER NOT NULL DEFAULT 0,
  tokens_cache_read      INTEGER NOT NULL DEFAULT 0,
  tokens_cache_write_5m  INTEGER NOT NULL DEFAULT 0,
  tokens_cache_write_1h  INTEGER NOT NULL DEFAULT 0,
  tokens_reasoning       INTEGER NOT NULL DEFAULT 0,
  cost_native_micro_usd   INTEGER,    -- NULL = 无 native 数据
  cost_computed_micro_usd INTEGER,
  has_estimated      INTEGER NOT NULL DEFAULT 0,      -- 含估算事件，UI 降级样式
  dirty_since        INTEGER,         -- 非空 = 待局部重算（崩溃恢复的持久 dirty 队列）
  PRIMARY KEY (source_instance_id, session_key)
);

-- ============================================================
-- session_model_stats：会话 × 模型聚合，派生表。职责：会话内
-- 模型分布展示；与 sessions 同生命周期（同事务重算、可重建）。
-- ============================================================
CREATE TABLE session_model_stats (
  source_instance_id TEXT NOT NULL,
  session_key        TEXT NOT NULL,
  model              TEXT NOT NULL DEFAULT '',        -- 派生表内部键：'' = 模型未知
  event_count        INTEGER NOT NULL DEFAULT 0,
  tokens_input           INTEGER NOT NULL DEFAULT 0,
  tokens_output          INTEGER NOT NULL DEFAULT 0,
  tokens_cache_read      INTEGER NOT NULL DEFAULT 0,
  tokens_cache_write_5m  INTEGER NOT NULL DEFAULT 0,
  tokens_cache_write_1h  INTEGER NOT NULL DEFAULT 0,
  tokens_reasoning       INTEGER NOT NULL DEFAULT 0,
  cost_native_micro_usd   INTEGER,
  cost_computed_micro_usd INTEGER,
  PRIMARY KEY (source_instance_id, session_key, model),
  FOREIGN KEY (source_instance_id, session_key)
    REFERENCES sessions (source_instance_id, session_key) ON DELETE CASCADE
);

-- ============================================================
-- tool_calls：工具调用子表。职责：工具名/status/时长的索引化
-- 聚合；事件变更时整集合替换（同事务：旧集合删除 + 新集合插入）。
-- ============================================================
CREATE TABLE tool_calls (
  source_instance_id TEXT NOT NULL,
  event_key   TEXT NOT NULL,
  child_id    TEXT NOT NULL,              -- 源内调用 id（toolu_* / callID / 稳定序号）
  name        TEXT NOT NULL,
  status      TEXT,
  duration_ms INTEGER CHECK (duration_ms IS NULL OR duration_ms >= 0),
  detail      TEXT,                       -- JSON 摘要（指标级，不存内容）
  PRIMARY KEY (source_instance_id, event_key, child_id),
  FOREIGN KEY (source_instance_id, event_key)
    REFERENCES usage_events (source_instance_id, event_key) ON DELETE CASCADE
);

-- ============================================================
-- event_charges：非 token 计费子表。职责：web search / premium-
-- request 数 / AI-credit 等按次/按额度计费项的可索引聚合；
-- 不折算进成本列；整集合替换规则同 tool_calls。
-- ============================================================
CREATE TABLE event_charges (
  source_instance_id TEXT NOT NULL,
  event_key   TEXT NOT NULL,
  kind        TEXT NOT NULL,              -- 'web-search' / 'premium-requests' / 'ai-credits' / …
  quantity    REAL NOT NULL,              -- 数量（premium-request 数可为小数，如 3.6）
  unit        TEXT NOT NULL,              -- 'requests' / 'credits' / 'count'
  unit_price_micro_usd INTEGER,           -- 单价口径（可得时）
  detail      TEXT,
  PRIMARY KEY (source_instance_id, event_key, kind),
  FOREIGN KEY (source_instance_id, event_key)
    REFERENCES usage_events (source_instance_id, event_key) ON DELETE CASCADE
);

-- ============================================================
-- pending_messages：未定稿消息队列。职责：流式/中断轮次的
-- 定稿状态机（finish 非空 → 定稿；error 非空 → 定稿；15 分钟
-- 无更新 → 陈旧定稿）；水位线不得越过任何 pending id；
-- 定稿入库后删除行（本表即"未决队列"）。
-- ============================================================
CREATE TABLE pending_messages (
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  pending_key TEXT NOT NULL,              -- 源内消息 id（如 opencode message.id）
  first_seen_at  INTEGER NOT NULL,
  last_seen_at   INTEGER NOT NULL,
  last_payload_hash TEXT,
  state       TEXT NOT NULL DEFAULT 'open'
              CHECK (state IN ('open','finalized','errored','stale')),
  PRIMARY KEY (source_instance_id, pending_key)
);

-- ============================================================
-- ingest_errors：摄取错误。职责：坏行/跳过行可见性（30 天删
-- 源下静默丢弃 = 永久丢弃）；只存错误码/位置/hash/长度/计数，
-- 永不存内容。
-- ============================================================
CREATE TABLE ingest_errors (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  stream_key   TEXT NOT NULL,             -- 出错流（同 watermarks.stream_key 口径）
  source_position TEXT NOT NULL DEFAULT '',  -- 出错位置（offset/行号等）；'' = 位置不适用（NOT NULL 保证 UNIQUE 去重计数生效）
  error_code   TEXT NOT NULL,             -- 'json-parse' / 'schema-mismatch' / 'unresolved-mapping' / …
  payload_hash TEXT,                      -- 出错字节的 sha256
  payload_len  INTEGER,
  occurrence_count INTEGER NOT NULL DEFAULT 1,
  first_seen_at INTEGER NOT NULL,
  last_seen_at  INTEGER NOT NULL,
  UNIQUE (source_instance_id, stream_key, source_position, error_code)
  -- 重扫遇同位置同错误：ON CONFLICT 计数 +1、刷新 last_seen_at
);

-- ============================================================
-- watermarks：增量摄取游标。职责：PK 四元组 + 行内
-- cursor_version/generation 构成逻辑游标六元组，position 只在
-- 同六元组内可比；身份断裂 generation++ 并归零 position。
-- 与事件写入同一事务推进。
-- ============================================================
CREATE TABLE watermarks (
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  channel      TEXT NOT NULL DEFAULT 'default',
  stream_key   TEXT NOT NULL,             -- 文件相对路径 / 库内表 / API 端点+参数
  kind         TEXT NOT NULL CHECK (kind IN ('jsonl-offset','sqlite-position','api-period')),
  cursor_version INTEGER NOT NULL DEFAULT 1,   -- 游标格式版本，演进时 ++
  generation   INTEGER NOT NULL DEFAULT 0,     -- 流身份代际，断裂 ++
  position     TEXT,                      -- 游标本体（kind 解释）
  file_id      TEXT,                      -- 文件身份（inode/创建时间等；库文件为 NULL）
  header_hash  TEXT,                      -- 流头指纹（首行/首页采样哈希）
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (source_instance_id, channel, stream_key, kind)
);

-- ============================================================
-- quota_snapshots：配额窗口观测（ADR-0008）。职责：与
-- UsageEvent 正交的配额时间序列；used_pct 允许 NULL——本地
-- 重建只承诺 activity + burn rate，官方端点或用户配置容量时
-- 才出百分比。
-- ============================================================
CREATE TABLE quota_snapshots (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  limit_key    TEXT NOT NULL,             -- 稳定限额键（'codex:primary' / 'claude:5h' / 源原生 limit_id 规范化）
  observed_at  INTEGER NOT NULL,
  used_pct     REAL CHECK (used_pct IS NULL OR (used_pct >= 0 AND used_pct <= 100)),
  used_amount  REAL,                      -- 观测用量（unit 计量；本地重建用）
  capacity     REAL,                      -- 窗口容量；未知/动态为 NULL
  unit         TEXT,                      -- 'tokens' / 'requests' / 'credits' / 'pct'
  window_minutes INTEGER,
  resets_at    INTEGER,
  provenance   TEXT NOT NULL CHECK (provenance IN ('official','local-estimate','user-configured')),
  rule_version TEXT,                      -- 窗口规则版本（本地重建规则硬编码带版本）
  confidence   TEXT CHECK (confidence IS NULL OR confidence IN ('high','medium','low')),
  raw          TEXT,                      -- 原始快照 JSON（溯源与字段演进兜底）
  UNIQUE (source_instance_id, limit_key, observed_at)
);

-- ============================================================
-- pricing_history：带生效期的价格规则。职责：按事件时点计价
-- ——reprice 用 event.timestamp 匹配 [effective_from,
-- effective_to) 取价，禁止"最新价回算历史"。
-- ============================================================
CREATE TABLE pricing_history (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  model_key    TEXT NOT NULL,             -- 归一化模型键
  service_tier TEXT NOT NULL DEFAULT '',  -- 计费率档；'' = 默认档
  context_tier TEXT NOT NULL DEFAULT '',  -- 上下文档位预留；'' = 无
  price_kind   TEXT NOT NULL CHECK (price_kind IN
                 ('input','output','cache-read','cache-write-5m','cache-write-1h','request')),
  unit_price_micro_usd INTEGER NOT NULL,  -- 每 per_unit 数量单价
  per_unit     INTEGER NOT NULL DEFAULT 1000000,   -- token 价默认每 1M；request 价用 1
  effective_from INTEGER NOT NULL,
  effective_to   INTEGER,                 -- NULL = 至今有效
  price_source TEXT NOT NULL,             -- 'litellm:<fetched_at|etag>' / 'bundled:<pkgVersion>' / 'user-override'
  UNIQUE (model_key, service_tier, context_tier, price_kind, effective_from)
);

-- ------------------------------------------------------------
-- daily_usage_rollups：日聚合（分层保留的永久层，ADR-0002 分层保留节）
-- raw 事件 90 天清理前生成；热力图/趋势/Wrapped 永久查这张表
-- ------------------------------------------------------------
CREATE TABLE daily_usage_rollups (
  day                    TEXT NOT NULL,      -- YYYY-MM-DD，本地时区日界
  source_instance_id     TEXT NOT NULL REFERENCES source_instances(id),
  model                  TEXT,               -- NULL = 无模型事件（tool 行已排除在聚合外）
  project_key            TEXT NOT NULL DEFAULT '',
  tokens_input           INTEGER NOT NULL DEFAULT 0,
  tokens_output          INTEGER NOT NULL DEFAULT 0,
  tokens_cache_read      INTEGER NOT NULL DEFAULT 0,
  tokens_cache_write_5m  INTEGER NOT NULL DEFAULT 0,
  tokens_cache_write_1h  INTEGER NOT NULL DEFAULT 0,
  tokens_reasoning       INTEGER NOT NULL DEFAULT 0,
  native_cost_micro_usd   INTEGER,
  computed_cost_micro_usd INTEGER,
  event_count            INTEGER NOT NULL DEFAULT 0,
  PRIMARY KEY (day, source_instance_id, model, project_key)
);

-- ============================================================
-- 索引清单（二轮审查 §2）
-- ============================================================
CREATE INDEX idx_events_status_ts  ON usage_events (status, timestamp);
CREATE INDEX idx_events_source_ts  ON usage_events (source_instance_id, timestamp);
CREATE INDEX idx_events_session    ON usage_events (source_instance_id, session_key, timestamp);
CREATE INDEX idx_events_project_ts ON usage_events (project_key, timestamp);
CREATE INDEX idx_events_model_ts   ON usage_events (model, timestamp);
CREATE INDEX idx_events_repricing  ON usage_events (pricing_eligibility, computed_cost_status);
CREATE INDEX idx_conflict_event    ON conflict_observations (source_instance_id, event_key);
CREATE INDEX idx_sessions_dirty    ON sessions (dirty_since) WHERE dirty_since IS NOT NULL;
CREATE INDEX idx_quota_limit_obs   ON quota_snapshots (limit_key, observed_at);
```

## 备注

- **集合替换规则**：`tool_calls` / `event_charges` 不做行级 upsert——事件变更时在同事务内「按事件身份整集合删除 → 插入新集合」（ADR-0002 事务边界）。
- **派生表重建**：`sessions` / `session_model_stats` 任何时候可由 `usage_events`（`status='counted'`）全量重建；`dirty_since` 是持久 dirty 队列，崩溃恢复先清队列再服务查询。空会话（重算后 `event_count=0`）直接 DELETE。
- **观察层 = `usage_events` 当前行 + `conflict_observations` 败方**：fold 规则见 ADR-0002「收敛契约」；重建场景（property test 之一）即清空 projection/子表/派生表后按 fold 重放。
- **迁移**：schema 演进只追加新迁移号，不改已应用迁移；`schema_migrations` 最新 version 即进程模型握手用的 `schema_version`。
