# Tokly 数据库 Schema（规范 DDL）

本文是 Tokly 本地仓库的**规范 schema**，M0.5 冻结门交付物（见 [路线图](路线图.md)）。契约语义见 [ADR-0002](adr/0002-数据架构.md)（收敛契约、成本所有权、水位线三类源语义）；四层寿命与可重建性见 [ADR-0011](adr/0011-存储生命周期.md)；计价精度见 [ADR-0015](adr/0015-计价精度.md)；配额语义见 [ADR-0008](adr/0008-配额窗口.md)。

**2026-08-03 四轮修订**（依据 [审查-2026-08-03](research/审查-2026-08-03.md)）：全表转 `STRICT`；补 `usage_observations` 观测层与 `daily_tool_rollups`；解开 `conflict_observations` 的外键死锁；日聚合修复主键并记录时区；`event_charges.quantity` 去 REAL。逐项理由见文末「本次修订的问题清单」。

## 全局类型约定（硬约束）

- 时间一律 `INTEGER` **epoch_ms**（UTC）；
- 金额一律 `INTEGER` **micro-USD**（10⁻⁶ 美元）；**任何会被求和的数量都不得用 REAL**（聚合漂移）——计数类用整数倍数加单位倍率表达；
- token 一律非负 `INTEGER`；token 维度**不可得时写 0 并用 `token_coverage` 标注**——0 不冒充"已确认没有"；
- **全表 `STRICT`**：SQLite 默认动态类型，往 INTEGER 列写字符串会成功入库而 CHECK 拦不住。STRICT 是让"由 DDL 固化，不靠代码自觉"真正成立的唯一手段，附带使 PRIMARY KEY 列隐含 NOT NULL；
- 文本一律 UTF-8；DDL 由迁移工具按 `schema_migrations` 顺序应用，禁止手工改库。

## 连接 PRAGMA（固化集合）

```sql
PRAGMA journal_mode = WAL;
PRAGMA foreign_keys = ON;
PRAGMA synchronous  = NORMAL;      -- WAL 标配：fsync 降到 checkpoint。本库可从源重扫重建，
                                   -- 断电丢失最后若干事务的代价远低于每次 commit 一次 fsync 的摄取开销
PRAGMA busy_timeout = 5000;        -- 三端并存时读端不应直接吃 SQLITE_BUSY
PRAGMA temp_store   = MEMORY;
PRAGMA mmap_size    = 268435456;   -- 256 MB
PRAGMA cache_size   = -16000;      -- 16 MB。注：4–5 倍提升的实测出自 Bun + bun:sqlite 栈，
                                   -- 须在 M0.5 spike ① 上按 rusqlite 重测并重新打戳（ADR-0001）
```

## DDL

```sql
-- ============================================================
-- schema_migrations：迁移记录。职责：schema_version 协商
-- （握手比对，版本漂移拒开）的权威来源。
-- ============================================================
CREATE TABLE schema_migrations (
  version     INTEGER PRIMARY KEY,        -- 单调递增迁移号
  name        TEXT NOT NULL,
  applied_at  INTEGER NOT NULL            -- epoch_ms
) STRICT;

-- ============================================================
-- source_instances：源实例。职责：把 源类型 × 账号 × 机器 ×
-- 源数据根 映射为稳定实例 ID——事件身份 (source_instance_id,
-- event_key) 的一半；跨实例同 eventKey 不互相去重。
-- machine 由 <data_dir>/tokly/machine 持久化（ADR-0010）。
-- ============================================================
CREATE TABLE source_instances (
  id            TEXT PRIMARY KEY,         -- 稳定 ID = UNIQUE 四元组的确定性哈希
                                          -- （兜底 UNIQUE 中 account 为 NULL 时不去重的 SQL 语义）
  source        TEXT NOT NULL CHECK (source IN
                  ('claude-code','codex','opencode','gemini-cli','copilot','cursor')),
  account       TEXT,                     -- 账号标识；NULL = 单账号/未知
  machine       TEXT NOT NULL,
  data_root     TEXT NOT NULL,            -- 源数据根（规范化路径 / API 端点 base）
  format_version TEXT,                    -- 源格式代际（opencode Gen A/B、codex legacy/paginated…）
  first_seen_at INTEGER NOT NULL,
  last_seen_at  INTEGER NOT NULL,
  UNIQUE (source, account, machine, data_root)
) STRICT;

-- ============================================================
-- usage_observations：观测层，只追加。职责：fold 的输入，
-- 「重建」property test 的实际依据（ADR-0011）。
-- normalized 是归一化后 UsageEvent 的 JSON（含 tool_calls /
-- event_charges 两个子集合）——只存指标，永不存对话内容。
-- 保留期 retention.observations_days，默认 90 天。
-- ============================================================
CREATE TABLE usage_observations (
  id                 INTEGER PRIMARY KEY AUTOINCREMENT,
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  event_key          TEXT NOT NULL,
  origin_stream      TEXT NOT NULL,
  stream_generation  INTEGER NOT NULL,
  source_position    TEXT NOT NULL,
  payload_hash       TEXT NOT NULL,       -- 源 payload 的 sha256
  normalizer_version INTEGER NOT NULL,
  event_timestamp    INTEGER NOT NULL,    -- 事件发生 epoch_ms，供按时间剪枝
  observed_at        INTEGER NOT NULL,    -- 观测时刻 epoch_ms
  normalized         TEXT NOT NULL,       -- JSON：归一化事件 + 子集合
  UNIQUE (source_instance_id, event_key, origin_stream, stream_generation, source_position, payload_hash)
) STRICT;

-- ============================================================
-- usage_events：归一化事件主表，current projection。
-- 职责：每个 (source_instance_id, event_key) 一行的当前状态。
-- 缺席 ≠ 删除，只有显式删除证据才 tombstoned。
-- 保留期 retention.events_days，默认 180 天（ADR-0011）。
-- ============================================================
CREATE TABLE usage_events (
  -- 事件身份
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  event_key          TEXT NOT NULL,
  identity_quality   TEXT NOT NULL DEFAULT 'strong'
                     CHECK (identity_quality IN ('strong','weak')),
  -- 胜出观测的 revision 三元组：仅同 (origin_stream, stream_generation) 内可比
  origin_stream      TEXT NOT NULL,
  stream_generation  INTEGER NOT NULL,
  source_position    TEXT NOT NULL,
  payload_hash       TEXT NOT NULL,
  normalizer_version INTEGER NOT NULL,
  observed_at        INTEGER NOT NULL,    -- 首次观测时刻
  updated_at         INTEGER NOT NULL,    -- projection 最近变更时刻
  has_conflict       INTEGER NOT NULL DEFAULT 0
                     CHECK (has_conflict IN (0,1)),   -- 冲突明细随观测层过期，本标志永久保留
  observations_pruned_at INTEGER,         -- 非空 = 支撑观测已清理，fold 不再对本行生效
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
  exclude_reason     TEXT,                -- 'unverified-schema'（ADR-0014）/ 'non-authoritative-channel'（R3）/ …
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
  native_cost_provider    TEXT,
  cost_computed_micro_usd INTEGER,
  computed_cost_status    TEXT NOT NULL DEFAULT 'pending'
                          CHECK (computed_cost_status IN
                            ('pending','computed','unmatched','skipped','priced-at-snapshot')),
  pricing_version         TEXT,
  pricing_eligibility     TEXT NOT NULL DEFAULT 'standard'
                          CHECK (pricing_eligibility IN ('standard','estimated-tokens','disabled')),
  display_policy          TEXT NOT NULL DEFAULT 'computed-only' CHECK (display_policy IN
                            ('native-preferred','computed-preferred','both',
                             'native-only','computed-only','estimate-only')),
  raw_usage               TEXT,           -- 仅有损映射时存
  raw_schema              TEXT,           -- 源格式版本标识
  PRIMARY KEY (source_instance_id, event_key),
  CHECK (granularity = 'event' OR period_end IS NOT NULL),
  CHECK (cost_native_micro_usd IS NULL OR native_cost_kind IS NOT NULL),
  CHECK (tokens_reasoning <= tokens_output)   -- 口径不变量 2 由 DDL 固化，不只靠 Rust 校验
) STRICT;

-- ============================================================
-- conflict_observations：冲突观测。职责：同坐标/跨流同键不同
-- payload 的显式留证，禁止静默 last-write-wins。
-- 属观测层：与 usage_observations 同寿命，**不外键引用投影层**
-- ——旧设计的外键使 compact 与「重建」在 foreign_keys=ON 下
-- 直接跑不通（ADR-0011）。
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
  observed_at      INTEGER NOT NULL
) STRICT;

-- ============================================================
-- sessions：会话级聚合，派生表，**永久保留**。
-- 投影窗口内可由 usage_events 重建；窗口外 frozen_at 置位后
-- 定格，不再重建（ADR-0011）。
-- 空会话删除规则：重算后 counted 事件为 0 即 DELETE。
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
  cost_native_micro_usd   INTEGER,        -- NULL = 该会话无任何 native 数据
  cost_computed_micro_usd INTEGER,
  has_estimated      INTEGER NOT NULL DEFAULT 0 CHECK (has_estimated IN (0,1)),
  has_partial_cost   INTEGER NOT NULL DEFAULT 0 CHECK (has_partial_cost IN (0,1)),
                                          -- 1 = 只有部分事件有该成本口径，汇总不可当完整账目读
  dirty_since        INTEGER,             -- 非空 = 待局部重算（崩溃恢复的持久 dirty 队列）
  frozen_at          INTEGER,             -- 非空 = 支撑事件已清理，本行定格
  PRIMARY KEY (source_instance_id, session_key)
) STRICT;

-- ============================================================
-- session_model_stats：会话 × 模型聚合，派生表，随 sessions。
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
) STRICT;

-- ============================================================
-- tool_calls：工具调用子表，随投影层寿命。
-- 事件变更时整集合替换（同事务：旧集合删除 + 新集合插入）。
-- 永久层由 daily_tool_rollups 承担。
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
) STRICT;

-- ============================================================
-- event_charges：非 token 计费子表。整集合替换规则同 tool_calls。
-- quantity 用整数千分位（quantity_milli）：premium-request 的
-- 3.6 次存 3600。REAL 会在月度求和上漂移，与金额同理。
-- ============================================================
CREATE TABLE event_charges (
  source_instance_id TEXT NOT NULL,
  event_key   TEXT NOT NULL,
  kind        TEXT NOT NULL,              -- 'web-search' / 'premium-requests' / 'ai-credits' / …
  quantity_milli INTEGER NOT NULL CHECK (quantity_milli >= 0),   -- 数量 × 1000
  unit        TEXT NOT NULL,              -- 'requests' / 'credits' / 'count'
  unit_price_micro_usd INTEGER,           -- 单价口径（可得时）
  detail      TEXT,
  PRIMARY KEY (source_instance_id, event_key, kind),
  FOREIGN KEY (source_instance_id, event_key)
    REFERENCES usage_events (source_instance_id, event_key) ON DELETE CASCADE
) STRICT;

-- ============================================================
-- pending_messages：未定稿消息队列。职责：流式/中断轮次的定稿
-- 状态机（finish 非空 → 定稿；error 非空 → 定稿；15 分钟无更新
-- → 陈旧定稿）。**仅适用于可按 id 随机重读的源**（SQLite 活库
-- 类）；JSONL 类顺序水位线不使用本机制——按偏移推进的流无法
-- 「跳过中间的 pending 继续读后面」，二者的 pending 语义不同。
-- 定稿入库后删除行（本表即未决队列）。
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
) STRICT;

-- ============================================================
-- ingest_errors：摄取错误。职责：坏行/跳过行/未验证格式版本的
-- 可见性——源会清理历史，静默丢弃 = 永久丢弃。
-- 只存错误码/位置/hash/长度/计数，永不存内容。
-- ============================================================
CREATE TABLE ingest_errors (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  stream_key   TEXT NOT NULL,             -- 出错流（同 watermarks.stream_key 口径）
  source_position TEXT NOT NULL DEFAULT '',  -- '' = 位置不适用（NOT NULL 保证 UNIQUE 去重计数生效）
  error_code   TEXT NOT NULL,             -- 'json-parse' / 'schema-mismatch' / 'unresolved-mapping'
                                          -- / 'unverified-schema'（ADR-0014）/ …
  detail_key   TEXT NOT NULL DEFAULT '',  -- 错误码的补充键，如未验证格式的版本标识
  payload_hash TEXT,
  payload_len  INTEGER,
  occurrence_count INTEGER NOT NULL DEFAULT 1,
  first_seen_at INTEGER NOT NULL,
  last_seen_at  INTEGER NOT NULL,
  UNIQUE (source_instance_id, stream_key, source_position, error_code, detail_key)
  -- 重扫遇同位置同错误：ON CONFLICT 计数 +1、刷新 last_seen_at
) STRICT;

-- ============================================================
-- watermarks：增量摄取游标。PK 四元组 + 行内 cursor_version /
-- generation 构成逻辑游标六元组，position 只在同六元组内可比；
-- 身份断裂 generation++ 并归零 position。与事件写入同一事务推进。
-- ============================================================
CREATE TABLE watermarks (
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  channel      TEXT NOT NULL DEFAULT 'default',
  stream_key   TEXT NOT NULL,             -- 文件相对路径 / 库内表 / API 端点+参数
  kind         TEXT NOT NULL CHECK (kind IN ('jsonl-offset','sqlite-position','api-period')),
  cursor_version INTEGER NOT NULL DEFAULT 1,   -- 游标格式版本，演进时 ++
  generation   INTEGER NOT NULL DEFAULT 0,     -- 流身份代际，断裂 ++
  position     TEXT,                      -- 游标本体（kind 解释）；NULL = 尚未读过
  file_id      TEXT,                      -- 文件身份（inode/创建时间等；库文件为 NULL）
  header_hash  TEXT,                      -- 流头指纹（首行/首页采样哈希）
  updated_at   INTEGER NOT NULL,
  PRIMARY KEY (source_instance_id, channel, stream_key, kind)
) STRICT;

-- ============================================================
-- quota_snapshots：配额窗口观测（ADR-0008）。与 UsageEvent 正交。
-- used_pct 允许 NULL——本地重建只承诺 activity + burn rate。
-- 数量列用整数千分位，理由同 event_charges。
-- ============================================================
CREATE TABLE quota_snapshots (
  id           INTEGER PRIMARY KEY AUTOINCREMENT,
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  limit_key    TEXT NOT NULL,             -- 稳定限额键（'codex:primary' / 'claude:5h' / …）
  observed_at  INTEGER NOT NULL,
  used_pct_milli  INTEGER CHECK (used_pct_milli IS NULL OR
                                 (used_pct_milli >= 0 AND used_pct_milli <= 100000)),  -- 百分比 × 1000
  used_amount_milli INTEGER,              -- 观测用量 × 1000（unit 计量）
  capacity_milli    INTEGER,              -- 窗口容量 × 1000；未知/动态为 NULL
  unit         TEXT,                      -- 'tokens' / 'requests' / 'credits' / 'pct'
  window_minutes INTEGER,
  resets_at    INTEGER,
  provenance   TEXT NOT NULL CHECK (provenance IN ('official','local-estimate','user-configured')),
  rule_version TEXT,                      -- 窗口规则版本（本地重建规则硬编码带版本）
  confidence   TEXT CHECK (confidence IS NULL OR confidence IN ('high','medium','low')),
  raw          TEXT,                      -- 原始快照 JSON（溯源与字段演进兜底）
  UNIQUE (source_instance_id, limit_key, observed_at)
) STRICT;

-- ============================================================
-- pricing_history：带生效期的价格规则。reprice 用 event.timestamp
-- 匹配 [effective_from, effective_to) 取价（ADR-0002 / ADR-0015）。
-- 同键区间不得重叠：重叠会让同一事件匹配到两个价格，破坏决定论。
-- UNIQUE 只约束起点，重叠须由写入端在事务内校验并由下方触发器兜底。
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
  UNIQUE (model_key, service_tier, context_tier, price_kind, effective_from),
  CHECK (effective_to IS NULL OR effective_to > effective_from)
) STRICT;

CREATE TRIGGER pricing_history_no_overlap
BEFORE INSERT ON pricing_history
FOR EACH ROW WHEN EXISTS (
  SELECT 1 FROM pricing_history p
   WHERE p.model_key = NEW.model_key
     AND p.service_tier = NEW.service_tier
     AND p.context_tier = NEW.context_tier
     AND p.price_kind = NEW.price_kind
     AND NEW.effective_from < COALESCE(p.effective_to, 9223372036854775807)
     AND COALESCE(NEW.effective_to, 9223372036854775807) > p.effective_from
)
BEGIN
  SELECT RAISE(ABORT, 'pricing_history: overlapping effective range');
END;

-- ============================================================
-- daily_usage_rollups：日聚合，**永久层**（ADR-0011）。
-- day 是本地日界，所用时区随行记录——用户改时区后，投影窗口内
-- 自动重算，窗口外保留原口径并在 UI 标注，不静默改写历史。
-- model 用 '' 表示未知：SQLite 普通表的 PK 列可为 NULL 会使
-- 唯一性失效（STRICT 已隐含 NOT NULL，此处再显式一次）。
-- ============================================================
CREATE TABLE daily_usage_rollups (
  day                    TEXT NOT NULL,      -- YYYY-MM-DD，按 tz 切分的本地日界
  tz                     TEXT NOT NULL,      -- IANA 时区名，生成该行时所用
  utc_offset_minutes     INTEGER NOT NULL,   -- 该日该时区的偏移，便于无 tz 库时校验
  source_instance_id     TEXT NOT NULL REFERENCES source_instances(id),
  model                  TEXT NOT NULL DEFAULT '',   -- '' = 模型未知
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
  rollup_version         INTEGER NOT NULL DEFAULT 1, -- 聚合逻辑版本；变更后窗口内重建、窗口外定格可见
  frozen_at              INTEGER,                    -- 非空 = 支撑事件已清理，本行不再重建
  PRIMARY KEY (day, tz, source_instance_id, model, project_key)
) STRICT;

-- ============================================================
-- daily_tool_rollups：工具调用日聚合，**永久层**（ADR-0011）。
-- 使「会话/工具调用级分析」在事件清理后依然成立——工具名基数
-- 低，体积代价很小。
-- ============================================================
CREATE TABLE daily_tool_rollups (
  day                TEXT NOT NULL,
  tz                 TEXT NOT NULL,
  source_instance_id TEXT NOT NULL REFERENCES source_instances(id),
  tool_name          TEXT NOT NULL,
  call_count         INTEGER NOT NULL DEFAULT 0,
  ok_count           INTEGER NOT NULL DEFAULT 0,
  error_count        INTEGER NOT NULL DEFAULT 0,
  total_duration_ms  INTEGER NOT NULL DEFAULT 0,
  rollup_version     INTEGER NOT NULL DEFAULT 1,
  frozen_at          INTEGER,
  PRIMARY KEY (day, tz, source_instance_id, tool_name)
) STRICT;

-- ============================================================
-- 索引清单
-- ============================================================
CREATE INDEX idx_obs_identity      ON usage_observations (source_instance_id, event_key);
CREATE INDEX idx_obs_prune         ON usage_observations (observed_at);
CREATE INDEX idx_events_status_ts  ON usage_events (status, timestamp);
CREATE INDEX idx_events_ts         ON usage_events (timestamp);   -- 按时间剪枝，不受 status 前导列限制
CREATE INDEX idx_events_source_ts  ON usage_events (source_instance_id, timestamp);
CREATE INDEX idx_events_session    ON usage_events (source_instance_id, session_key, timestamp);
CREATE INDEX idx_events_project_ts ON usage_events (project_key, timestamp);
CREATE INDEX idx_events_model_ts   ON usage_events (model, timestamp);
CREATE INDEX idx_events_repricing  ON usage_events (pricing_eligibility, computed_cost_status);
CREATE INDEX idx_conflict_event    ON conflict_observations (source_instance_id, event_key);
CREATE INDEX idx_conflict_prune    ON conflict_observations (observed_at);
CREATE INDEX idx_sessions_dirty    ON sessions (dirty_since) WHERE dirty_since IS NOT NULL;
CREATE INDEX idx_quota_limit_obs   ON quota_snapshots (limit_key, observed_at);
CREATE INDEX idx_rollup_source_day ON daily_usage_rollups (source_instance_id, day);
```

## 备注

- **集合替换规则**：`tool_calls` / `event_charges` 不做行级 upsert——事件变更时在同事务内「按事件身份整集合删除 → 插入新集合」（ADR-0002 事务边界）。
- **观测层与重建**：fold 的输入是 `usage_observations`，输出是 `usage_events` 加两张子表；`conflict_observations` 同为 fold 的输出（可重建）。「重建」property test = 清空投影/子表/派生表后按 fold 重放，**适用域为观测保留窗口内**（ADR-0011）。
- **派生表重建**：`sessions` / `session_model_stats` 在投影窗口内可由 `usage_events`（`status='counted'`）重建；`dirty_since` 是持久 dirty 队列，崩溃恢复先清队列再服务查询。空会话（重算后 `event_count=0`）直接 DELETE。
- **进程所有权不落表**：daemon 所有权由 `<data_dir>/tokly/daemon.lock` 的 OS 咨询锁承担（[ADR-0012](adr/0012-进程与本地服务.md)），进程崩溃时内核自动释放——不需要租约表、心跳与超时判据。
- **`WITHOUT ROWID` 的评估**：`tool_calls` / `event_charges` / `session_model_stats` / `watermarks` 是窄表且用复合文本 PK，改 `WITHOUT ROWID` 可省一次索引查找与一份索引空间；`usage_events` 行较宽，需按实际行宽与页大小实测后再定。**在 M0.5 spike ① 上一并实测，不先猜。**
- **迁移**：schema 演进只追加新迁移号，不改已应用迁移；`schema_migrations` 最新 version 即握手用的 `schema_version`。
- **一致性门禁**：本文的 SQL 块与迁移代码建出的库必须逐项一致，由 CI 的 `sqlite_schema` diff 测试强制（[文档规范](文档规范.md)）——文档与代码谁先改都行，不一致即红。

## 本次修订的问题清单（2026-08-03）

| 问题 | 后果 | 处置 |
|---|---|---|
| `daily_usage_rollups` 的 `model` 可空且在 PK 内 | SQLite 普通表 PK 列不隐含 NOT NULL，多行 `model IS NULL` 共存 → **永久层静默重复累加**，raw 清理后不可修正 | `NOT NULL DEFAULT ''` + 全表 STRICT（PK 隐含 NOT NULL） |
| 全表非 STRICT | 类型纪律实为口号：往 INTEGER 列写字符串会入库，CHECK 的比较走类型亲和性 | 全表 `STRICT` |
| `conflict_observations` 外键无 `ON DELETE` | `foreign_keys=ON` 下 compact 与「重建」被外键拒绝，**DDL 层跑不通** | 去外键，降为观测层独立表；`has_conflict` 承担永久可见性 |
| 观测层无表 | 「fold 可从 observation 重建」无实际载体，实现已自行加表而文档未同步 | 补 `usage_observations`，并把 2.5 倍行数写进 ADR-0011 存储预算 |
| 日聚合把本地日界写死而不记时区 | 用户换时区后历史**永久错误且不可重算** | 增 `tz` / `utc_offset_minutes` 并入 PK；窗口内重算、窗口外定格可见 |
| 日聚合无版本锚点 | 聚合逻辑出 bug 且 raw 已清理后无法修正 | 增 `rollup_version` / `frozen_at` |
| 工具调用无永久层 | 「工具调用级分析」在事件清理后消失，而文档仍在承诺 | 增 `daily_tool_rollups` |
| `event_charges.quantity` / `quota_snapshots` 用 REAL | 与"禁止 REAL 存可求和量"自相矛盾，月度求和漂移 | 改整数千分位 |
| `tokens_reasoning ≤ tokens_output` 无 CHECK | 口径不变量只在 Rust 侧强制，与"由 DDL 固化"不符 | 加 CHECK |
| `pricing_history` 无区间重叠约束 | 同一事件可匹配两个价格，破坏计价决定论 | 加 `no_overlap` 触发器 + `effective_to > effective_from` |
| `computed_cost_status` 无法表达"价目回溯不可得" | 首装前历史按最早快照计价却标成正常计算值 | 增 `priced-at-snapshot`（ADR-0015） |
| 无按时间剪枝的索引 | 保留期清理的 `WHERE timestamp < X` 受 `(status,timestamp)` 前导列限制 | 加 `idx_events_ts` |
| `sessions` 汇总无法表达部分口径 | native/computed 只有部分事件有值时，汇总被当完整账目读 | 增 `has_partial_cost` |
