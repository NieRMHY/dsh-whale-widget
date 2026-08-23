# DSH 小鲸鱼挂件（MHY fork）

![DSH 小鲸鱼挂件](assets/DSH2.png)

> Fork 自 **[MeteorNOX/DeepSeek-Balance-Whale-Widget](https://github.com/MeteorNOX/DeepSeek-Balance-Whale-Widget)**（MIT License）。
>
> 原项目显示 DeepSeek 官方账户余额；**本 fork 把数据源换成了自建 / 中转 New API 实例的账号级 token 用量**。
> 因此原项目中关于余额、`DEEPSEEK_API_KEY`、峰谷金额换算的说明对本 fork **均不适用**。

DSH（DeepSeek Harness）Web 界面右下角的常驻挂件：小鲸鱼 + 气泡，显示**今日 token**、累计 token 与订阅剩余额度。标准 DSH bundle 插件，随 Web 界面自动启用。

当前版本 `0.2.8-mhy.2`（基于上游 `0.2.8`）。

## 本 fork 改了什么

| 项 | 原项目 | 本 fork |
|----|--------|---------|
| 主数字 | DeepSeek 账户余额（`¥`） | **今日 token** |
| 提示行 | 今日已用金额 | **累计总 token · 订阅剩余额度** |
| 数据源 | `api.deepseek.com/user/balance` | New API `/api/log/self`（type=2 消费日志）聚合 |
| 凭据 | `DEEPSEEK_API_KEY` + 可选 `DEEPSEEK_PLATFORM_TOKEN` | New API 访问令牌（凭据名可配置），前两者都不再需要 |
| 缓存 | 25s 内存 TTL | `~/.dsh/.dshw-newapi.json`，按天分桶，5 分钟增量刷新 |
| 限速 | 无 | 起跑间隔节流 6 次/s + 并发 4 |
| 首次启动 | 立即显示 | 全量统计约 1 分钟，期间显示 WARMUP 提示 |
| 每轮对话气泡 | 本轮消耗金额 | 本轮 **token 数** |
| 订阅额度 | 无 | `/api/subscription/self`，多条 active 订阅合并求和 |

原项目的交互全部保留：拖拽 + 四边吸附、左吸附镜像翻转、按压 Q 弹、随窗口缩放、音效、随机台词、汉堡菜单（大小 / 音量 / 音效 / 气泡开关 / 每轮消耗开关）。

### 为什么每轮气泡不显示金额

经 New API 中转时，实际扣费由中转侧的计费表达式决定，未必与官方价一致（实测差异：中转侧只按小时判高峰、无周末谷价；部分模型超长上下文按 0 计费；扣减单位是订阅 quota 而非人民币）。显示一个对不上账的金额没有意义，所以直接显示本轮 token 数，与主数字口径统一。

### 统计实现要点（踩过的坑）

对接 New API 的 `/api/log/self` 时有几个反直觉的地方，值得记下来：

- **`id` 字段不是稳定主键，而是结果集内的排名**（第一页 `1..100`、第二页 `101..200`）。带 `start_timestamp` 的增量查询会**重新从 1 编号**，若拿它去重，每条增量都会撞上缓存里的旧排名而被误判为「已存在」，统计从此永久停滞。本 fork 用 `request_id` 去重，配合 `created_at` 水位与同秒边界集。
- **分页参数 `p` 是 1-based**，`p=0` 与 `p=1` 返回同一页；按 `0..pages-1` 遍历会漏掉最后一页。
- **`page_size` 被服务端硬顶在 100**（传 500 / 1000 / 2000 都只返回 100 条），加大页码无法提速，只能提高请求速率。
- **聚合按本地日期分桶且只追加不重算**，所以服务端清理旧日志后累计值不会缩水；今日用量直接取当天桶，不需要跨天重置逻辑。
- 分页期间新到的日志会把旧行推向更大的排名，**升序遍历会重复读到、反向遍历会漏行**，故只能升序 + 去重。
- 漏页不做逐页补抓（排名会漂移，补页可能重复计入），而是标记 `incomplete` 并隔 1 小时整体重建。

改动集中在 `lib/index.js`，均带 `Add by MHY` / `Modify by MHY` 标注。

## 配置

部署私有信息不写进源码，放在 `~/.dsh/.dshw-config.json`（**不要提交**）：

```json
{
  "base": "https://<your-newapi-host>",
  "tokenKey": "NEWAPI_ACCESS_TOKEN",
  "quotaCnyRate": 0
}
```

| 键 | 说明 |
|----|------|
| `base` | New API 实例地址。为空时挂件降级显示「未配置」提示，不报错 |
| `tokenKey` | DSH 凭据名，对应 New API 网页「个人设置 → 生成访问令牌」拿到的 token |
| `quotaCnyRate` | 每 1 quota 折算的本币金额，用于把订阅剩余额度显示成钱；填 `0` 则按原始 quota 显示 |

`quotaCnyRate` 怎么定：用一个已知面值的订阅反推即可 —— 拿该订阅的 `amount_total`（quota）去除面值，例如面值 100 元、`amount_total` 为 `5000000`，则 `100 / 5000000 = 2e-5`。填 `0` 就不折算，直接显示 quota。

令牌本身通过 DSH 凭据服务配置（`~/.dsh/.credentials.yaml`），键名与 `tokenKey` 一致。

## 安装

在仓库根目录（`package.json` 所在目录，仓库根本身就是插件包）执行：

```bash
dsh plugin --profile web add link:.
```

然后重启 `dsh web`，浏览器 F5。

- 复制到别处时用绝对路径：`dsh plugin --profile web add link:/path/to/dsh-whale-widget`
- 不要写成 `link:./dsh-whale-widget`——仓库里没有这个子目录，那样会装成普通依赖，挂件不出现
- 之后移动了源码目录，需要重新 add；提示冲突时先 `remove` 再 `add`
- 用 `link:` 安装时改了源码要重启 `dsh web` 才生效（ESM 模块缓存）

卸载：

```bash
dsh plugin --profile web remove dsh-whale-widget
```

## 验证

```bash
dsh --profile web --dump-config | grep whale

curl -s localhost:3080/dsh-whale/balance.json
curl -s localhost:3080/dsh-whale/last-turn.json
curl -sI localhost:3080/dsh-whale/image.png
```

- `balance.json` → `{"ok":true,"totalBalance":<累计token>,"todayUsage":<今日token>,"currency":"tok","subRemain":...,"requestDrift":...}`
  - 首轮全量期间返回 `{"ok":false,"code":"WARMUP"}`，属正常
  - `requestDrift` = 本地天桶汇总请求数 − 服务端 `request_count`，偏差过大时会 `console.warn`
  - `incomplete: true` 表示上次全量有漏页，会在 1 小时后自动重建
- `last-turn.json` → `{seq, turn, tokens, ts}`
- `image.png` → 200 `image/png`

## 目录结构

```text
.
├── package.json           # DSH bundle 插件元数据
├── cordis.patch.yml       # 插件挂载声明
├── lib/index.js           # 宿主侧插件本体（唯一源码文件）
├── assets/                # 鲸鱼 PNG、展示图、gif、音效 mp3
└── whale-widget-prompt.md # 上游的完整规格 / 视觉参数说明
```

调整文字位置、颜色、动画、吸附逻辑或台词组时参考 `whale-widget-prompt.md`（该文件为上游内容，其中的定价表部分对本 fork 已不适用）。

## 常见问题

- **挂件不出现**：确认 `dsh --profile web --dump-config` 里有 `dsh-whale-widget`；重启 `dsh web` 后 F5。
- **显示「未配置 New API 地址」**：`~/.dsh/.dshw-config.json` 的 `base` 为空。
- **显示「未配置凭据 xxx」**：DSH 凭据里缺少 `tokenKey` 指定的那个键。
- **数字一直是「统计中」**：首轮全量约 1 分钟，期间不要关掉 `dsh web` 进程。
- **累计值不涨 / 今日恒为 0**：升级到本 fork 前的旧缓存（`v1`，用排名去重）会被自动作废重建；若仍不动，删掉 `~/.dsh/.dshw-newapi.json` 重启。
- **每轮气泡不显示**：确认菜单里「每轮对话后自动显示」已勾选；一轮对话必须完整结束（`turn/end`）才结算。
- **没有声音**：确认 `assets/*.mp3` 在包内，缺失时静默降级。

## 许可证

MIT License，详见 [LICENSE](LICENSE)。原项目版权归 [MeteorNOX](https://github.com/MeteorNOX) 及其贡献者所有。
