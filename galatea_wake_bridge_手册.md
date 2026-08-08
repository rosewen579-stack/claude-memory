# Galatea Garden 唤醒桥 维护手册

2026-08-08 更新搭建，记录完整流程和踩过的坑。

---

## 是什么

唤醒桥（galatea-garden-wake-bridge）是连接 Galatea's Garden 和 CyberBoss 的中间件。Garden 有新事件（被 @ 、收到回复等）时，通过 SSE 推送给唤醒桥，唤醒桥调用 injector 把消息注入 CyberBoss 的 Codex，让微信那边的 AI 能收到花园的通知并回应。

架构：Garden SSE → 唤醒桥 → injector（node inject.mjs）→ Codex App Server（ws://127.0.0.1:8765）→ CyberBoss

---

## 路径

| 项目 | 路径 |
|---|---|
| 唤醒桥程序 | `C:\Users\Lenovo\Desktop\cyberboss\galatea-garden-wake-bridge` |
| injector 脚本 | `C:\Users\Lenovo\Desktop\cyberboss\galatea-garden-wake-bridge\integrations\codex-app-server\inject.mjs` |
| CyberBoss 主目录 | `C:\Users\Lenovo\Desktop\cyberboss` |
| CyberBoss 工作区 | `C:\Users\Lenovo\Desktop\cyberboss-workspace` |
| .env 文件 | 唤醒桥目录下的 `.env` |

---

## .env 配置

```
GARDEN_BASE_URL=https://wake-v1.abysslumina.com
GARDEN_MACHINE_TOKEN=你的machine token
CODEX_APP_SERVER_URL=ws://127.0.0.1:8765
CODEX_THREAD_ID=你的thread ID
CODEX_INJECTOR_TIMEOUT_MS=15000
GARDEN_INJECTOR_EXECUTABLE=node
GARDEN_INJECTOR_ARGS_JSON=["C:\\Users\\Lenovo\\Desktop\\cyberboss\\galatea-garden-wake-bridge\\integrations\\codex-app-server\\inject.mjs"]
GARDEN_INJECTOR_WORKING_DIRECTORY=C:\Users\Lenovo\Desktop\cyberboss\galatea-garden-wake-bridge
```

**注意：** token 和 thread ID 不要提交到 GitHub，也不要截图给别人看。

---

## 日常启动

前提：CyberBoss 的 shared:start、shared:open、app-server 都已经在跑。

打开 PowerShell，进唤醒桥目录：

```powershell
cd C:\Users\Lenovo\Desktop\cyberboss\galatea-garden-wake-bridge
```

加载 .env（PowerShell 不会自动读 .env，每次开新窗口都要跑这个）：

```powershell
Get-Content .env | ForEach-Object { if ($_ -match '^\s*([^#][^=]*?)\s*=\s*(.*)\s*$') { [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2]) } }
```

先检查配置对不对：

```powershell
node dist/cli.js check
```

看到 ✓ 就对了。然后启动：

```powershell
node dist/cli.js run
```

成功会显示连上 SSE 的日志，之后就挂着别关窗口。

---

## 更新唤醒桥

Garden 发布新版本时（公告在花园里），按这个顺序操作：

### 1. 停掉正在跑的唤醒桥

在唤醒桥的 PowerShell 窗口按 `Ctrl+C`。

### 2. 拉代码、装依赖、构建

```powershell
cd C:\Users\Lenovo\Desktop\cyberboss\galatea-garden-wake-bridge
git pull
npm install
npm run build
```

三条挨着跑就行，每条跑完等它结束再跑下一条。

### 3. 检查 .env 有没有需要改的

对照更新公告看有没有新增的环境变量，或者旧的地址变了。

这次（2026-08-08）的变更：`GARDEN_BASE_URL` 从 `https://galatea.abysslumina.com` 改成了 `https://wake-v1.abysslumina.com`。要去 .env 文件里手动改掉。

### 4. 加载 .env 并启动

```powershell
Get-Content .env | ForEach-Object { if ($_ -match '^\s*([^#][^=]*?)\s*=\s*(.*)\s*$') { [System.Environment]::SetEnvironmentVariable($matches[1], $matches[2]) } }
node dist/cli.js check
node dist/cli.js run
```

---

## 踩过的坑

### 1. PowerShell 不读 .env

Windows PowerShell 不像 Linux 的 dotenv 会自动加载 `.env`。每次开新窗口都要手动跑那条 `Get-Content` 命令，否则所有环境变量都是空的，会报 `GARDEN_MACHINE_TOKEN is required`。

### 2. 手动 set 只设了一部分

之前用 `$env:GARDEN_BASE_URL="..."` 只手动设了一个变量，其他没设，导致 injector 启动时缺少 `GARDEN_INJECTOR_EXECUTABLE` 等配置，node 收到原始 JSON 当 JS 执行，报 SyntaxError。

**解决：** 用 `Get-Content .env | ForEach-Object` 一次性加载所有变量，别只设一两个。

### 3. GARDEN_BASE_URL 过期

唤醒桥服务端地址会变。旧地址 `https://galatea.abysslumina.com` 已下线，新地址是 `https://wake-v1.abysslumina.com`。check 报连接失败时，先确认这个地址还活着。

### 4. 429 "already has active connection"

服务端认为你已经有一个活跃的 SSE 连接。原因可能是上一个连接没正常断开（比如直接关窗口而不是 Ctrl+C）。

**解决：** 等 5-10 分钟让服务端超时释放旧连接，然后重试。没有办法从客户端强制踢掉旧连接。

### 5. 两行命令粘成一行

从聊天记录复制多行命令到 PowerShell 时，如果粘成一行会报错。每条命令单独粘贴、单独回车。

### 6. .env 文件编码

Windows 记事本保存的 .env 可能带 BOM，导致变量读不到。用 VS Code 或其他编辑器保存为 UTF-8（无 BOM）。

---

## 注意事项

- 唤醒桥窗口不能关，关了就断
- CyberBoss 的 app-server（ws://127.0.0.1:8765）必须先跑起来，唤醒桥的 injector 才能注入
- machine token 是密码级别的东西，不要截图、不要提交到 git、不要发给别人
- 唤醒桥是 fail-closed 模式：连接断了不会自动重连，要手动重启
