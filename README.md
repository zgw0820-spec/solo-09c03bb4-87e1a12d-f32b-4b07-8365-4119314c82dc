# 制品标签摘要查询（跨实例单飞缓存 + 共享回源名额）

FastAPI + Redis 的制品标签查询服务：两个查询实例共享同一份 Redis 缓存与同一个回源名额信号量。对同一个标签的一轮并发缓存未命中只会向源站回源一次；两个实例合计最多同时执行 `ORIGIN_FETCH_CONCURRENCY`（默认 2）个源站 GET，排不上名额的请求最多等待 `CACHE_WAIT_TIMEOUT_MS`（默认 1000ms），随后返回明确的可重试错误而不会回源。发布确认之后开始的查询一定能看到新摘要，晚到的旧回源结果不会覆盖它。

## 组成

| 服务 | 说明 | 端口 |
| --- | --- | --- |
| `redis` | 缓存与协调状态 | 6379 |
| `origin` | 可设置响应延迟/失败的本地制品源站，自行统计请求数 | 9000 |
| `query-1` / `query-2` | 两个 FastAPI 查询实例（同一镜像） | 8001 / 8002 |

网页由任一查询实例直接提供：<http://localhost:8001/>（或 <http://localhost:8002/>）。

## 启动

```bash
docker compose up --build
```

首次启动后源站预置了 `v1.0.0=sha256:old1111`。

## 网页功能

* **发布标签摘要**：选择经由哪个实例发布，填写标签、新摘要和 `request_id`。发布 = 写源站 + 原子写缓存并递增发布代号；同一 `request_id` 重试是幂等的。
* **查询与并发测试**：标签输入框支持逗号/空格分隔的多个标签；「两实例并发查询」会同时向两个实例各发 N 个请求（多标签交叉排队），汇总请求数、实际回源数、等待者数、缓存命中数、排队超时数、排队被发布短路数、最长等待与摘要一致性，并在请求进行中以 100ms 间隔采样「源站 GET 在途峰值」。「仅使缓存失效」只把缓存标记为过期（保留发布代号），用来在不动源站的前提下制造一轮并发未命中——随后并发查询时同一标签的源站计数应只增加 1。
* **源站模拟**：设置响应延迟、接下来 N 次查询失败。卡片展示共享名额的「当前在途 / 上限」、Redis 记录的在途峰值与网页本轮观测峰值；「清零计数」同时重置峰值。
* **发布状态**：查看已确认摘要、发布代号、源站版本、在途发布。
* **统计**：源站的 `request_count` 由源站自己统计，是“回源次数”的权威依据；网关侧另有各实例的缓存命中/未命中/等待/回源调用计数。

## 一致性设计

所有协调状态在 Redis 中，键前缀 `artifact:{tag}:`：

| 键 | 用途 |
| --- | --- |
| `state` | 已确认摘要 + `gen`（发布代号）、`origin_revision`、`fetch_seq`、`result_seq`、`fresh_until` |
| `lock` | 跨实例单飞回源锁（UUID 值 + TTL） |
| `fetch_seq` | 回源写缓存的单调序号 |
| `result_seq` | 状态写入与错误记录共享的结果序号，供等待者判断“本轮是否有结果” |
| `fetch_error` | 最近一次失败回源，含结果序号，等待者据此得到同样的明确错误 |
| `pending` | 在途发布（短 TTL） |
| `pub_unconfirmed` | 结果未知的发布标记（源站可能已写入，网关未确认），读者会看到“旧摘要 + 明确标记” |
| `pub_seq` / `pub_lock` | 发布会话代号 / 发布串行锁 |
| `last_publish` | 最近一次已确认发布 |
| `fetch_slots` | 集群级源站 GET 名额 zset：每个在途回源一个成员，score 为该名额的过期时刻；成员数即当前在途数 |
| `fetch_slots:peak` | 集群级在途名额高水位（峰值），随「清零计数」重置 |

关键保证：

1. **单飞**：同轮未命中的请求竞争 Redis 锁，只有 1 个成为 leader 回源，其余等待 leader 写入的 `state` 或 `fetch_error`，不会各自回源。leader 拿到锁后会再检查一次缓存，避免“leader 刚释放锁、等待者补位”导致重复回源。
2. **发布栅栏**：leader 在调用源站**之前**记录当前发布代号；写缓存用 Lua 原子校验代号未变，且 `fetch_seq` 单调。因此旧回源晚到只会被拒绝，不会覆盖新发布的摘要。源站版本号只用于展示，不参与栅栏——演示源站重启后版本号会归零，若拿它当栅栏会让缓存永久写不进去。
3. **确认后可见**：发布提交用 Lua 原子完成“校验代号单调 → 写 `state` → 写 `last_publish` → 清在途标记 → 递增 `result_seq`”，发布确认后开始的查询直接命中新摘要。
4. **未确认不冒充新结果**：源站发布超时/网关提交失败时保留 `pub_unconfirmed`；读者仍能拿到最后已确认摘要，但响应带 `stale_due_to_pending: true`、`result_kind: "last_confirmed_while_publishing"`，发布状态显示 `publishing`。
5. **重试不回退**：发布代号单调，且 `request_id` 有幂等记录；对旧发布的重试返回 `superseded` 而不是把较新摘要回滚。
6. **明确错误**：源站超时 → `504 origin_timeout`；源站 5xx → `502 origin_error`；Redis 不可用/协调失败 → `503 cache_unavailable`；共享名额（或同标签回源）等待超过 1 秒 → `503 fetch_queue_timeout`（带 `Retry-After: 1`，不调用源站）；被并发写入拒绝 → `503 cache_write_rejected`。均可重试（非 404 标签不存在），等待者与 leader 得到同一错误，之后可以再查。
7. **共享回源名额**：两个实例的回源 leader 在调用源站前必须先从 Redis zset 信号量 `fetch_slots` 获取名额，全集群上限 `ORIGIN_FETCH_CONCURRENCY=2`。同标签请求仍由 `lock` 单飞合并，只可能有一个 leader，因此同标签并发只占一个名额；不同标签同时失效时，超出上限的 leader 排队等待，总等待（锁等待 + 名额排队）不超过 `CACHE_WAIT_TIMEOUT_MS=1000ms`，超时返回可重试的 `fetch_queue_timeout`。发布走 `POST /publish`，完全不经过该信号量，名额占满也能发布。
8. **排队短路**：leader 排队期间持续检查该标签缓存；若发布被确认（或另一轮写入了新鲜摘要），立即返回新值并标记 `queued_short_circuit: true`、`origin_called: false`，不再回源。
9. **名额安全释放**：名额只能由持有自己 token 的请求释放（Lua `ZREM` 带 token 比较），晚到的旧请求释放不了后来者的名额；持有进程被杀死等异常退出时，名额在 `FETCH_SLOT_TTL_MS`（默认 10 秒）后由下一次申请按 score 自动回收。请求取消（`finally` 同时释放名额与标签锁）、源站失败都会立刻归还名额。既有的发布代号 / `fetch_seq` 栅栏保证晚到的旧回源响应无法把旧摘要写回缓存。

## API

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/api/tag/{tag}?refresh=1` | 查询摘要（`refresh` 强制绕过新鲜度检查，仍走单飞） |
| POST | `/api/publish` | `{tag, digest, request_id?}`，发布并确认 |
| GET | `/api/status/{tag}` | 发布状态、已确认摘要、代号、在途发布 |
| POST | `/api/cache/invalidate` | `{tag}`，只让已有缓存过期，改源站与发布代号 |
| GET | `/api/stats` | 网关计数 + 源站计数 + `fetch_slots`（名额 `capacity` / 当前 `inflight` / 历史 `peak`） |
| POST | `/api/origin/control` | 设置源站 `delay_ms` / `fail_next` / `reset_counters` 等；`reset_counters` 同时清零名额峰值 |
| GET | `/api/health`, `/api/config` | 健康检查与实例配置 |

## 配置（环境变量）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `INSTANCE_NAME` | `query` | 实例名 |
| `REDIS_URL` | `redis://localhost:6379/0` | Redis 地址 |
| `ORIGIN_URL` | `http://localhost:9000` | 源站地址 |
| `PUBLIC_INSTANCES` | `http://localhost:8001,http://localhost:8002` | 网页可直连的实例列表（也用于 CORS） |
| `ORIGIN_TIMEOUT_MS` | `2000` | 源站查询超时 |
| `CACHE_WAIT_TIMEOUT_MS` | `1000` | 锁等待 + 回源名额排队的合计上限（毫秒） |
| `ORIGIN_FETCH_CONCURRENCY` | `2` | 两个实例共享的源站 GET 在途上限 |
| `FETCH_SLOT_TTL_MS` | `10000` | 名额持有者异常退出后的回收时间（毫秒） |
| `LOCK_TTL_MS` | `12000` | 单飞锁 TTL |
| `PUBLISH_TIMEOUT_MS` | `5000` | 源站发布确认超时 |
| `PENDING_TTL_MS` | `15000` | 在途发布标记 TTL |
| `UNCONFIRMED_TTL_S` | `600` | 结果未知的发布标记 TTL |
| `REVALIDATE_TTL_S` | `300` | 缓存新鲜期 |
| `REDIS_SOCKET_TIMEOUT_MS` | `1500` | Redis 操作超时（超时即报协调失败） |

## 测试

```bash
# 1) 单元/交错测试（fakeredis，无需外部服务）
pip install -r requirements.txt fakeredis pytest pytest-asyncio
python3 -m pytest tests/test_cache.py

# 2a) Docker 栈
docker compose up --build -d
python3 tests/e2e_check.py --q1 http://localhost:8001 --q2 http://localhost:8002 --origin http://localhost:9000 --redis-port 6379
```

没有 Docker 时可用本地栈（需要本机 `redis-server`、`uvicorn`）：

```bash
scripts/dev_stack.sh     # 起 origin + query-1 + query-2（8101/8102/9100，Redis 6390）
python3 tests/e2e_check.py
```

`tests/e2e_check.py` 覆盖：16 并发请求单飞仅回源 1 次、缓存命中不回源、旧回源被发布栅栏拦截、确认后两实例都看到新摘要、重试旧发布不回退、发布失败/超时明确报错且未确认标记生效、源站超时 504 后可恢复、含 `/` 的标签、非法输入 422，以及共享名额专项：4 个不同标签跨实例并发时在途峰值为 2、排队约 1 秒返回可重试的 `fetch_queue_timeout` 且不调用源站、名额占满时发布不受限、排队中的请求在同标签发布确认后直接返回新摘要且不回源。

`tests/test_cache.py` 另外用 fakeredis 覆盖单元级交错：等待者优先返回“发布后”的新状态而不是上一轮错误、发布提交失败后保留未确认标记并可重试恢复、源站重启导致版本号倒退后仍可写入、request_id 复用校验，以及名额专项：全局在途不超过 2、排队超时可重试且释放名额、发布不占名额、排队短路、请求取消/源站失败归还名额、旧 token 释放不了后来者的名额、同标签并发仍只占一个名额。
