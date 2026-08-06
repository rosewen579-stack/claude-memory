# 心潮动态心智 部署笔记

2026-08-06 搭通，记录踩过的所有坑。

---

## 环境

- 电脑：Windows（Lenovo），Node.js v24.17.0
- 心潮版本：xinchao-dynamic-mind@2.6.0
- 安装路径：C:\Users\Lenovo\xinchao-dynamic-mind\
- 目的：通过 Cloudflare Tunnel 把本地心潮暴露到公网，让 ChatGPT/Codex 的 MCP 连接器能连上

---

## 第一步：安装心潮

```
cd C:\Users\Lenovo\xinchao-dynamic-mind
npm install
```

---

## 第二步：启动心潮

**不要用 .env 文件。** Windows 记事本保存的 .env 有编码问题（可能是 BOM），SERVICE_TOKEN 读不到，报 "SERVICE_TOKEN is required"。

用手动 set 环境变量的方式启动，每行输完回车：

```
set SERVICE_TOKEN=你的token（64位十六进制）
set MCP_ENABLED=true
set OAUTH_ENABLED=true
set OAUTH_APPROVAL_TOKEN=至少16位的口令（授权时要输的）
set OAUTH_PUBLIC_BASE_URL=你的tunnel网址（第三步拿到后填）
npm start
```

启动顺序问题：OAUTH_PUBLIC_BASE_URL 需要 tunnel 的网址，但 tunnel 需要心潮先跑着。解决办法：
1. 先不带 OAUTH 启动心潮，确认 port 18110 能跑
2. 开 tunnel 拿到网址
3. Ctrl+C 停心潮，补上 OAUTH 的三个 set，重新 npm start

成功的日志长这样：
```
{"at":"...","event":"service_started","port":18110,"shadow":true,...}
```

---

## 第三步：Cloudflare Tunnel

开一个**新的**命令行窗口（别关心潮那个），下载并启动：

```
curl -L -o cloudflared.exe https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-windows-amd64.exe
cloudflared.exe tunnel --url http://127.0.0.1:18110
```

跑起来后会出一个框，里面有：
```
https://xxxxx-xxxxx-xxxxx.trycloudflare.com
```

这就是公网地址。**每次重启 tunnel 网址都会变**，变了之后：
- 心潮那边要重新 set OAUTH_PUBLIC_BASE_URL 并重启
- ChatGPT 连接器要重新填新地址

验证桥通了：手机浏览器打开 `https://你的地址/health`，看到 `{"ok":true}` 就对了。

---

## 第四步：连 ChatGPT

1. 手机先验证 OAuth 发现端点：打开 `https://你的地址/.well-known/oauth-authorization-server`，出来一坨 JSON 就对
2. ChatGPT → 设置 → 连接器 → 添加自定义连接器
3. 网址填：`https://你的地址/mcp`
4. 弹出授权页面，输入你设的 OAUTH_APPROVAL_TOKEN
5. 连上之后 Codex 就能调心潮的 MCP 工具了

---

## 踩过的坑

### 1. .env 读不到 SERVICE_TOKEN
Windows 记事本保存的 .env 文件编码有问题。解决：不用 .env，直接 set 环境变量。

### 2. bearer_token 报错
Codex CLI 的 config.toml 里写 `bearer_token` 会报 "bearer_token is not supported for streamable_http"。心潮的远程 MCP 走的是 OAuth，不是 Bearer token。**不要往 config.toml 里加心潮的配置。**

### 3. 本地连不上云端
CyberBoss 桥接到 ChatGPT 云端，云端摸不到 localhost。必须走 Cloudflare Tunnel 或公网服务器暴露端口。这个应该一开始就想到的。

### 4. OAUTH_APPROVAL_TOKEN 至少16位
不设或太短会崩，报 "OAUTH_APPROVAL_TOKEN must contain at least 16 characters"。

### 5. Ctrl+C 在命令行是停止不是复制
选中文字后按**回车**才是复制。Ctrl+C 会直接杀进程。

### 6. tunnel 地址会变
免费的 trycloudflare 每次重启分配新域名。想固定要么注册 Cloudflare 账号绑域名，要么租服务器。

---

## 两个窗口不能关

运行时需要保持两个命令行窗口：
1. 心潮服务（npm start，port 18110）
2. cloudflared tunnel

任何一个关了都断。电脑关机也断。

---

## 长期方案

免费 tunnel 的问题：电脑关了就断，网址每次变。

如果要 24 小时在线：租云服务器（~30-40 元/月），把心潮直接跑在服务器上，不需要 tunnel。
