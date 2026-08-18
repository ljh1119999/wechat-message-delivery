---
name: wechat-message-delivery
description: "AI Agent 微信消息通道完整调整机制——限流熔断、发送节奏、静默隐藏、回合中断应对、自动补发。核心原则：正常唔打扰，异常必通知，通知只一行。适用于所有走 iLink/企业微信通道的 Agent 推送场景。"
version: 1.0.0
author: Ben & Hermes
license: MIT
platforms: [linux, macos]
metadata:
  hermes:
    tags: [wechat, ilink, rate-limit, notification, delivery, message]
    related_skills: [hermes-maintenance, hermes-multi-bot-relay]
---

# 微信消息通道调整机制（Agent 推送不翻车指南）

> 2026-08 实测沉淀。症状：连续发 10 条消息后"冇下文"、图片静默丢图、cron 推送失败、后台任务乱码刷屏。
> 一句话：**正常唔打扰，异常必通知，通知只一行。**

## 触发词
"微信限流" "发唔出" "冇下文" "丢图" "消息断了" "通知太多" "静默" "推送失败"

## 一、发送保障（防限流）

### 1. 限流熔断（iLink 服务端 -2 错误）
本地熔断器 drop-in 配置 `~/.config/systemd/user/hermes-gateway.service.d/weixin-rate-limit.conf`：
```ini
[Service]
Environment="WEIXIN_RATE_LIMIT_CIRCUIT_THRESHOLD=5"     # 5次真限流先开闸（默认1太敏感）
Environment="WEIXIN_RATE_LIMIT_CIRCUIT_WINDOW_SECONDS=30"
Environment="WEIXIN_RATE_LIMIT_CIRCUIT_OPEN_SECONDS=10" # 开闸只锁10秒
```
- 偶发 -2 靠 `send_chunk_retries=4` + 3x 退避消化，唔会锁死后续消息
- 改完 `systemctl --user daemon-reload` + 重启 gateway

### 2. 发送节奏（每条消息间隔）
- 文本 chunk 间隔：`WEIXIN_SEND_CHUNK_DELAY_SECONDS`（默认 1.5s）
- **图片/文件/视频间隔：weixin.py `send()` 媒体循环加 `await asyncio.sleep(self._send_chunk_delay_seconds)`**（2026-08-18 补丁：媒体之前背靠背连发触发限流+丢图）
- ⚠️ weixin.py 系 hermes-agent 源码，`hermes update` 会覆盖补丁，更新后要重打

### 3. 批量图片交付（防静默丢图）
iLink 图片通道存在**静默丢图**：日志显示已发送、无报错，但用户实际收唔到（实测 6 张收 4 张）。
```bash
# 流程：zip 全量兜底 → 按方向分批（≤6张/批）→ 用户报收取数 → 缺边张补边张
zip -r output.zip output_dir/          # 先发 zip（完整交付保证）
# 再逐批发 MEDIA:/path 图片，等用户确认数量
```
丢图排查：gateway.log 有 `Sending image` 但用户冇收到 = 静默丢，补发即好，唔使深挖。

## 二、静默/隐藏（唔烦用户）

| 场景 | 机制 | 实现 |
|------|------|------|
| 定时同步无变化 | 零消息 | 脚本先比对内容 hash，一致即 `sys.exit(0)` 静默 |
| 监控类正常状态 | 全静默 | no_agent cron：stdout 空=静默，非空=告警（watchdog 模式） |
| 后台任务原始输出 | 唔推 | 输出重定向日志文件，微信只收一行"人话"状态 |
| 记忆上下文块 | 唔显示 | `hermes config set display.memory_notifications off`（AI 内部照用） |
| 代理/备份正常 | 静默 | 正常 exit 0 无输出，异常才 echo 告警 |

## 三、唔中断（回合超时应对）

**症状：连续几条消息后冇下文。根因系单回合工具调用过多（8-10 个）或长阻塞命令（如 sleep 25）触发平台截断回合，唔系限流。**
```bash
# 对策（行为层，写进 agent 规则）
1. 每 3-4 个工具调用发一条进度消息
2. 长等待拆短：sleep 25 → 5 秒轮询（for i in {1..5}; do sleep 5; check; done）
3. 回合一断，恢复后第一条消息先讲"做到边度"
4. 必要时拆多回合：做完一段就收工，用户回一句再继续
```

## 四、推送保障（cron 侧）

1. **补发脚本** `~/.hermes/scripts/delivery_retry.py`（cron 每5分钟）：扫 WATCH_JOBS 的 `last_delivery_error`，用活跃 bot token 补发最新 md。⚠️ 硬编码旧 token 系常见坑——比对 `gateway.log` 的 `Connected account=` + 文件 mtime
2. **代理预检** `~/.hermes/scripts/mihomo_precheck.sh`（cron 交易日 10:20/15:20，Gemini cron 前10分钟）：代理掉线自动拉起，拉起失败自动把 Gemini cron 降级 DeepSeek（直接改 jobs.json 原子写），恢复自动切返。唔好做每5分钟空转看门狗——用户明确反对浪费
3. **日报带执行情况**：每日 22:00 日报列出今日全部 cron 状态（✅/❌/⏳），❌ 当晚修复，修复唔到写明原因入待办

## Pitfalls（全部实测）

1. **mihomo 系 gateway 连坐仔**：nohup 启动嘅 mihomo 喺 gateway cgroup，gateway 重启就杀。治本=独立 systemd 化（`~/.config/systemd/user/mihomo.service`，Restart=always），同 gateway 零依赖
2. **googleapis 必须走美国节点**：mihomo url-test 自动选择会命中 AWS 东京段，Gemini API 报 `HTTP 400 User location is not supported`。规则固定 `DOMAIN-SUFFIX,googleapis.com,🇺🇸 美国`
3. **.env 嘅 export PATH 变量未展开**会污染 cron 子进程 PATH（`$HOME/...` 字面量）→ 外部命令全挂。cron 脚本开头必须 `export PATH="/usr/local/sbin:...:$PATH"` 防御
4. **cron 脚本依赖外部命令**（curl/pgrep/rsync）：全部要 PATH 防御，Python 脚本同样 `os.environ["PATH"] = ...` 前置
5. **限流排查顺序**：先 `cronjob list` 睇 `last_delivery_error`（唔好信 last_status: ok）→ 睇补发脚本输出 → `grep "rate limited" gateway.log` → 对比活跃 bot token。**"限流"唔系借口**，要查真限流 vs 本地熔断放大 vs 静默丢失

## 验证

1. 限流：`grep -c "cooldown active" gateway.log`（熔断拒绝数）vs `grep -c "backing off"`（真 -2 数）
2. 发送间隔：`grep "Sending image" gateway.log | tail` 睇时间戳间隔
3. 静默：正常 cron 输出 md 系 `Status: silent (empty output)`
4. 回合中断：用户唔再见到"连续多条后消失"
