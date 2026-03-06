# 日志、会话记录与报错查看指南

本文档针对 `openclaw-gateway-huiling` 部署，说明如何查看会话记录、消耗统计和报错信息。

---

## 会话记录（Session Logs）

### 文件位置

| 文件                                                             | 说明                                               |
| ---------------------------------------------------------------- | -------------------------------------------------- |
| `/home/wellingwong/.openclaw/agents/main/sessions/sessions.json` | 会话索引，记录所有 session key → session ID 的映射 |
| `/home/wellingwong/.openclaw/agents/main/sessions/<id>.jsonl`    | 每个会话的完整对话记录，每行一个 JSON 对象         |

### JSONL 字段结构

每条 assistant 消息包含：

```json
{
  "type": "message",
  "timestamp": "2026-02-26T05:34:05Z",
  "message": {
    "role": "assistant",
    "model": "claude-sonnet-4-6",
    "usage": {
      "input": 1234,
      "output": 567,
      "totalTokens": 1801,
      "cost": {
        "input": 0.0037,
        "output": 0.0085,
        "total": 0.0122
      }
    }
  }
}
```

### 方式 1：直接问 agent（最方便）

在飞书或 Discord 里发消息：

> "查一下今天的会话消耗了多少 token 和费用"
> "搜索我上次和你说过的关于 XX 的对话"

agent 内置 `session-logs` skill，会自动解析 JSONL 文件。

### 方式 2：本地命令查询

以下命令在本地终端运行（无需 SSH 进服务器）。

**查看会话列表：**

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="docker exec openclaw-openclaw-gateway-1 node dist/index.js sessions"
```

**今日总消耗：**

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="sudo jq -s '[.[] | .message.usage.cost.total // 0] | {total_usd: add, messages: length}' \
  /home/wellingwong/.openclaw/agents/main/sessions/*.jsonl"
```

**按会话列出消耗：**

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap --command="
for f in /home/wellingwong/.openclaw/agents/main/sessions/*.jsonl; do
  cost=\$(sudo jq -s '[.[] | .message.usage.cost.total // 0] | add' \"\$f\")
  date=\$(sudo head -1 \"\$f\" | jq -r '.timestamp' | cut -dT -f1)
  echo \"\$date \\\$\$cost \$(basename \$f)\"
done | sort -r
"
```

**提取某个会话的全部对话文本：**

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="sudo jq -r 'select(.message.role==\"user\" or .message.role==\"assistant\") |
  \"\(.message.role): \(.message.content[]? | select(.type==\"text\") | .text)\"' \
  /home/wellingwong/.openclaw/agents/main/sessions/<id>.jsonl"
```

---

## 报错查看

OpenClaw 有三层报错位置，从最快到最详细：

### 1. Docker 实时日志（最快，最常用）

适合：刚发了消息没有响应时，实时看发生了什么。

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="docker logs openclaw-openclaw-gateway-1 --follow 2>&1"
```

只看报错行：

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="docker logs openclaw-openclaw-gateway-1 --tail=100 2>&1 | grep -i 'error\|failed\|drop\|blocked'"
```

常见报错关键词：

| 关键词                 | 含义                                      |
| ---------------------- | ----------------------------------------- |
| `handler failed`       | 消息处理失败（权限、网络、模型错误）      |
| `gateway error`        | Discord WebSocket 连接问题                |
| `drop message`         | 消息被过滤掉（mention 未满足、DM 策略等） |
| `Blocked discord`      | Discord 访问控制拦截                      |
| `authentication_error` | API key 错误或打到了错误的 endpoint       |
| `EACCES`               | 文件权限问题                              |

### 2. Gateway 日志文件（更完整的历史）

适合：排查几小时前发生的问题。

日志文件格式为 `openclaw-YYYY-MM-DD.log`（JSON 格式），已通过 bind mount 持久化到宿主机：

| 位置             | 路径                                               |
| ---------------- | -------------------------------------------------- |
| 宿主机（持久化） | `~/.openclaw/runtime-logs/openclaw-YYYY-MM-DD.log` |
| 容器内           | `/tmp/openclaw/openclaw-YYYY-MM-DD.log`            |

```bash
# 查今天的报错（直接读宿主机文件，容器重启也不丢）
gcloud compute ssh <gateway-host> --zone=us-central1-a --tunnel-through-iap \
  --command="sudo grep -i 'error\|failed' ~/.openclaw/runtime-logs/openclaw-\$(date +%Y-%m-%d).log | tail -20"
```

```bash
# 或通过容器 exec 读取
gcloud compute ssh <gateway-host> --zone=us-central1-a --tunnel-through-iap \
  --command="docker exec openclaw-openclaw-gateway-1 \
  grep -i 'error\|failed' /tmp/openclaw/openclaw-\$(date +%Y-%m-%d).log | tail -20"
```

### 3. Session JSONL 里的报错（最精准）

适合：确认某次对话失败的具体原因（包含完整 API 错误信息）。

失败的 assistant 消息会有 `errorMessage` 字段：

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="sudo jq -r 'select(.message.errorMessage != null) |
  \"\(.timestamp) \(.message.errorMessage)\"' \
  /home/wellingwong/.openclaw/agents/main/sessions/*.jsonl | tail -20"
```

### 4. 综合健康检查

```bash
gcloud compute ssh openclaw-gateway-huiling --zone=us-central1-a --tunnel-through-iap \
  --command="docker exec openclaw-openclaw-gateway-1 node dist/index.js status"
```

---

## 快速排障流程

```
消息没有响应？
    ↓
docker logs --tail=50 | grep error     ← 先看有没有明显报错
    ↓
openclaw sessions                      ← 确认会话是否建立
    ↓
session JSONL errorMessage 字段        ← 看 API 调用失败的具体原因
    ↓
openclaw channels status --probe       ← 检查 Discord/飞书连接状态
```
