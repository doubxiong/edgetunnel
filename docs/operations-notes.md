# EDgetunnel 运维参数与升级记录

> 2026-11-22：同步上游 + 当日通过 API 完成云端布署。每条参数后面写「为什么」，避免下次只看到数字不知道为什么。

## 一、2026-11-22 升级记录

- 本 fork 的 `CHANGELOG` 原本停在 `2.1.20260617014121`（2026-06-17），fork 之后没有本地提交。
- 已快进合并上游 `cmliu/edgetunnel` main：`0 ahead / 104 behind`，HEAD 为 `5b2aaf0`，`CHANGELOG` 顶部变为 `2.1.20261110232646`（2026-11-10）。
- `_worker.js` 变动 39081 行 —— 说明这不是小补丁，行为面要按下面「默认值变化」重新核对一遍。

### 升级带来的、与「抖动」相关的上游改动

| 日期 | 改动 | 为什么和抖动有关 |
|---|---|---|
| 2026-08-09 | XHTTP 改用 `pipeTo` 双向直通 | Worker 免费版每请求有 CPU 预算，超时即断流；降 CPU 就是减少「用着用着卡住」 |
| 2026-07-24 | GrainTCP 重拨期间连接代际管理 | 重连时上行不再丢包，切节点不再硬断 |
| 2026-10-08 / 09-23 | 修 1101、统一本机标识 | 1101 是 Worker 报错页，表现就是「请求突然失败」 |
| 2026-11-10 | 新增连接复用开关 + 连接参数可调 | 多任务/多人共用时可开多条连接，避免互相挤 |
| 2026-07-29 | `PRELOAD_RACE_DIAL` 默认值 `true → false` | **行为变更**：老 README 的默认认知已失效，要显式设 `1` |

## 二、云端变量（已写入，2026-11-22）

| 变量 | 现值 | 为什么 |
|---|---|---|
| `ADMIN` | 原值保留（6 字符） | 面板登录密码；改它等于换订阅 TOKEN，会打断客户端 |
| `KV` | 绑定 `edgetunnel-kv`（`4d86b02984794161bd3350160be2cad8`） | 面板数据存这里；不绑则每次部署都算全新实例，面板设置全丢 |
| `UUID` | **本次不设** | 设了会把既有订阅里的 UUID 换掉，客户端要重导；等确实要固定时再单独做一次迁移 |
| `PRELOAD_RACE_DIAL` | `1` | 首连先用 DoH 并发查 A/AAAA 再竞速拨号；上游已默认关闭，必须显式打开 |
| `TCP_CONCURRENT_DIAL` | `2` | **反直觉**：并发拨号越高建连越快、但 IP 切换越频繁；抖动场景要往低里调 |
| `PROXY_CONCURRENT_DIAL` | `2` | 同上，作用于反代（proxyip）侧 |
| `PROXYIP` | 未改（默认共享 `proxyip.cmliussss.net`） | 换自建回源能减少晚高峰排队，属下一轮优化，量法见第七节 |
| `DEBUG` | 未设 | 调参期才开；长期开会增加日志量与 CPU |

## 三、客户端侧（mihomo）

- 订阅里保留多个优选 IP，不要把节点地址写死成单个 IP。
- `url-test`：`interval: 240`、`tolerance: 60`、`lazy: false`；探测 URL 用 `http://www.gstatic.com/generate_204`。
- 只换名字、走同一路径的近似节点要删掉，否则选择器会来回跳。

## 四、2026-11-22 实际执行记录（全部经 API 完成，没在面板手点）

- 令牌：`/accounts/{acc}/tokens/verify` 返回 `active`（权限含 Workers Scripts:Edit、Workers KV Storage:Edit）。
- 部署：`PUT /accounts/{acc}/workers/scripts/edgetunnel`，multipart，`metadata` 里同时带 `main_module`、`compatibility_date`、`bindings`。这样一次请求同时完成「换代码」和「写变量/绑 KV」。
- 自定义域：`vless.doub.cc.cd`，service `edgetunnel`，production；证书在重建时自动重签。
- 结果：Worker `modified_on` 更新，绑定读回为 `ADMIN / KV / PRELOAD_RACE_DIAL / PROXY_CONCURRENT_DIAL / TCP_CONCURRENT_DIAL`；账号下只剩这一个 Worker、一个自定义域。

### 踩过的三个坑（写下来免得再踩）

1. `PUT /workers/scripts/{name}/settings` 对 **API Token 一律 405 `10405 Method not allowed for this authentication scheme`**，即使令牌有 Workers 编辑权。该端点只认 Global API Key。→ 改用部署期 `metadata.bindings`。
2. **同一份源码、换新 worker 名 + 新域名 = 200；旧 `edgetunnel` 记录在任何域名下都是 522**（CF 边缘错误页，0.6–1.7s 快速返回，不是真超时）。判定为脚本记录本身状态异常，**删 worker 重建（同名、同绑定、重挂自定义域）后恢复 200**。以后遇到「代码没改、域名没改、突然全站 522」，先按这条查。
3. 国内网络下 `raw.githubusercontent.com` HTTPS 从 WSL/Windows 经代理都 `SSL_ERROR_SYSCALL`；`git@github.com` SSH 却是通的 → 取源码走 SSH clone，别跟 raw 域名较劲。

## 五、部署后验证清单

1. `curl https://vless.doub.cc.cd/` → `200`，内容是 nginx 默认页。**这是设计如此**：源码里 `env.URL` 未设时伪装页默认就是 `'nginx'`，404/未知路径都回落它，不代表坏了。
2. 客户端访问 `http://www.gstatic.com/generate_204` → `204`，才算数据面通（伪装页 200 只证明 Worker 活着）。
3. 大文件下载（>100MB）中途不再归零、不再长时间无响应。
4. 注意：本版本源码里**没有 `/version` 路由**。旧文档写的「访问 `/version` 看版本号」不成立，已删；要确认版本请看 Worker 脚本里的 `const Version = '2026-11-10 23:26:46'` 或部署时间。

## 六、还没做的

- `PROXYIP` 换自建回源（见第七节）。
- `UUID` 固定（会打断既有订阅，需要单独安排迁移窗口）。
- Workers 免费版单请求 CPU 预算仍是硬约束；如果后面上量，先看是不是 CPU 超时（1101/1102）而不是网络。

## 七、自建 PROXYIP 的思路（未实施）

- 在自有 VPS 上起一个只做转发的 Xray/sing-box 入站（VLESS+TLS 或 SOCKS5），把 `PROXYIP` 指向它。
- 目的不是「更快」，是「不跟别人挤同一台回源」；抖动多数来自回源侧的排队，而不是你家里的带宽。
- 实施前先量一次：对同一目标连续 20 次取 RTT 的 p50/p95，对比共享 proxyip 与自建回源的差距，有差距再上。

## 八、临时资源

- 排查期间建过 `dwtest`、`dwtest2` 两个测试 Worker 与 `wtest`、`wtest2`、`vless2` 三个测试域名，**已全部删除**；账号内无残留。
