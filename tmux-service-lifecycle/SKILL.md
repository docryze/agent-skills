---
name: tmux-service-lifecycle
description: 本地服务生命周期管理 — tmux session/window 操作、启动模板、验证流程
---

# tmux 本地服务生命周期管理

标准化管理本地服务的启动、重启、状态检查与停止。

---

## 1. 识别与准备

- 识别待启动的 `App Name` 和对应端口
- 工具链：`uv` (Python)、`pnpm` (Node.js)、`podman` (容器)、`mvn` (Java)

---

## 2. tmux Session/Window 管理

### 检查 session
```bash
tmux has-session -t local 2>/dev/null && echo "exists" || echo "not exists"
```

### 创建/复用 session + window
```bash
# session 不存在则创建，存在则复用
tmux new-session -d -s local -n <app-name> 2>/dev/null || tmux new-window -t local -n <app-name>
```

### 发送命令到指定 window
```bash
tmux send-keys -t local:<app-name> '<command>' Enter
```

### 查看运行状态（最后 20 行）
```bash
tmux capture-pane -t local:<app-name> -p -S -20
```

### 停止服务
```bash
# 1. 发送 Ctrl+C
tmux send-keys -t local:<app-name> C-c
sleep 3

# 2. 验证端口是否释放
lsof -ti:<port> 2>/dev/null && echo "进程仍在运行" || echo "进程已退出"

# 3. 进程未退出则强制 kill
PID=$(lsof -ti:<port> 2>/dev/null)
if [ -n "$PID" ]; then
  kill -9 $PID
  sleep 1
fi
```

### 删除 window
```bash
tmux kill-window -t local:<app-name>
```

---

## 3. 服务启动模板

### Spring Boot (Maven)
```bash
cd <project-dir>
tmux send-keys -t local:<app-name> 'mvn clean spring-boot:run -pl <module-name> -am -DskipTests' Enter
sleep 5
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/actuator/health 2>/dev/null || \
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/api/hello 2>/dev/null
```

### Python FastAPI / Flask (uv)
```bash
cd <project-dir>
tmux send-keys -t local:<app-name> 'uv run uvicorn main:app --port <port>' Enter
sleep 3
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>/health
```

### Node.js (pnpm)
```bash
cd <project-dir>
tmux send-keys -t local:<app-name> 'pnpm dev' Enter
sleep 3
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>
```

### Container (podman)
```bash
cd <project-dir>
tmux send-keys -t local:<app-name> 'podman-compose up' Enter
sleep 5
curl -s -o /dev/null -w "%{http_code}" http://localhost:<port>
```

---

## 4. 生命周期流程

### 启动
1. 检查 session → 不存在则 `new-session -d -s local`
2. `new-window -t local:<app-name>`
3. `send-keys` 启动命令
4. sleep + curl 验证

### 重启
1. `send-keys C-c` 停止旧进程
2. sleep 3，验证端口已释放（未释放则 kill -9）
3. `send-keys` 启动命令
4. sleep + curl 验证

### 状态检查
1. `tmux has-session -t local`
2. `tmux capture-pane -t local:<app-name> -p -S -20`
3. `curl` 端点

### 停止
1. `send-keys C-c` → sleep 3
2. 端口未释放 → `kill -9`
3. `tmux kill-window -t local:<app-name>`

---

## 5. 异常处理

| 场景 | 处理方式 |
|---|---|
| 无法获取 App Name | 用中文向用户确认 |
| 启动验证失败 | 捕获最后 20 行日志反馈给用户 |
| session `local` 已存在 | 不复建，直接复用 |
| window 名冲突 | `kill-window` 旧窗口后重建 |
| Java 进程不响应 C-c | 经常忽略 SIGINT，必须用 `lsof -ti:<port>` 验证并 `kill -9` |
| 启动时间不确定 | 循环 curl 重试（最多 10 次）代替固定 sleep |

---

## 6. 注意事项

### 端口验证
不要只看 `has-session`（窗口存活不代表进程存活），必须 `lsof -ti:<port>` 确认。
