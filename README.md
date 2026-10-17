# 💬 APIChat

![License](https://img.shields.io/badge/license-MIT-green)
![Python](https://img.shields.io/badge/python-3.8%2B-blue)
![No Dependencies](https://img.shields.io/badge/dependencies-zero-brightgreen)

通用 OpenAI 兼容 API 网页聊天室。纯 Python 标准库后端 + 单文件前端，零第三方依赖。

在线演示：https://api1.lblb.eu/chat/

## ✨ 功能

- 🔌 **多渠道管理**：添加多个 API 渠道（OpenAI 兼容接口），一键切换
- 💬 **多对话管理**：每个渠道独立对话历史，互不串味
- 📡 **群发模式**：同时向多个渠道/模型发送，回复并行返回
- 🌊 **流式输出**：SSE 流式打字机效果
- 🔒 **CORS 代理**：后端按白名单转发，解决浏览器跨域问题
- 💾 **本地存储**：渠道配置和对话历史存浏览器 localStorage，服务端不存密钥

## 🚀 快速开始

### 1. 配置

```bash
cp config.example.json config.json
# 编辑 config.json，把你的上游 API 域名加进 proxy_whitelist
```

### 2. 运行

```bash
python3 server.py
```

访问 http://localhost:18790。

### 3. 添加渠道

在页面右上角点"渠道管理"，填写：
- 名称：如 `deepseek`
- Base URL：如 `https://api.deepseek.com/v1`
- API Key：你的 key（只存浏览器本地，不上传）

## 🔧 配置说明

`config.json`：

| 字段 | 说明 |
|------|------|
| `port` | 监听端口，默认 18790（可用 `PORT` 环境变量覆盖） |
| `proxy_whitelist` | 允许转发的上游域名白名单 |
| `channels` | 预置渠道模板（不含 key） |

## 🚀 生产部署（详细步骤）

以下以 Ubuntu/Debian 服务器为例，从零开始部署。

### 步骤 1：准备服务器

需要一台有公网 IP 的 VPS，Python 3.8+（无需安装任何第三方包）：

```bash
python3 --version  # 确认 3.8+
```

### 步骤 2：上传文件

```bash
# 在服务器上
sudo mkdir -p /opt/apichat/www
sudo chown $USER:$USER /opt/apichat /opt/apichat/www

# 在本地，把文件传上去（或 git clone）
scp server.py config.json user@your-server:/opt/apichat/
scp www/index.html user@your-server:/opt/apichat/www/
```

### 步骤 3：配置

```bash
cd /opt/apichat
cp config.example.json config.json
vim config.json
```

把你要用的上游 API 域名加进 `proxy_whitelist`，例如：

```json
{
  "port": 18790,
  "proxy_whitelist": [
    "api.openai.com",
    "api.deepseek.com",
    "openrouter.ai"
  ],
  "channels": []
}
```

> ⚠️ 白名单是安全关键：只有这里的域名能被代理转发，防止你的服务器被当成开放代理滥用。

### 步骤 4：systemd 服务（开机自启）

```bash
sudo cp apichat.service.example /etc/systemd/system/apichat.service
sudo systemctl daemon-reload
sudo systemctl enable apichat
sudo systemctl start apichat
```

验证：

```bash
sudo systemctl status apichat
curl -s http://127.0.0.1:18790/health
```

### 步骤 5：Nginx 反代

后端只监听 `127.0.0.1`，通过 Nginx 对外提供 HTTPS 访问：

```nginx
# 在你的 server { } 块里加
location /chat/ {
    proxy_pass http://127.0.0.1:18790/;
    proxy_http_version 1.1;

    # SSE 流式必需：关缓冲
    proxy_buffering off;
    proxy_cache off;

    # 长超时（流式回复可能持续几分钟）
    proxy_read_timeout 600s;
    proxy_send_timeout 600s;

    # 透传头
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
}

# /chat 无斜杠时 301 跳到 /chat/
location = /chat {
    return 301 /chat/;
}
```

重载 Nginx：

```bash
sudo nginx -t && sudo systemctl reload nginx
```

> ⚠️ 三个关键点缺一不可：
> 1. `proxy_buffering off` —— 否则流式输出会被缓冲，打字机效果变"攒一波再吐"
> 2. `proxy_read_timeout 600s` —— 默认 60s 会把长回复掐断
> 3. `location = /chat` 的 301 —— 前端用相对路径，两种地址都要能进

### 步骤 6：防火墙

后端端口 **不要** 对公网开放，只留 Nginx 的 80/443：

```bash
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
# 不要放行 18790
```

### 步骤 7：验证

1. 浏览器打开 `https://你的域名/chat/`，应看到聊天界面
2. 点右上角"渠道管理"，添加一个渠道（Base URL + Key）
3. 发一句话，确认流式回复正常

### 步骤 8：更新

```bash
cd /opt/apichat
# 替换 server.py / www/index.html
sudo systemctl restart apichat
```

前端是纯静态文件，替换即生效，无需构建。

### 故障排查

| 现象 | 可能原因 |
|------|----------|
| 页面打不开 | Nginx 配置错误，`nginx -t` 检查语法 |
| 能打开但发消息没反应 | 白名单没加该 API 域名，检查 `config.json` |
| 回复一次性吐出、无打字机效果 | Nginx 没关 `proxy_buffering` |
| 长回复到一半断掉 | `proxy_read_timeout` 太小，改成 600s |
| 服务起不来 | `journalctl -u apichat -n 50` 看日志，多半是 `config.json` 格式错误 |

## 📁 项目结构

```
server.py              后端（Python 标准库，无第三方依赖）
www/index.html         前端（单文件，872 行原生 JS）
config.example.json    配置模板
apichat.service.example  systemd 服务模板
```

## 🔌 API

| 方法 | 路径 | 说明 |
|------|------|------|
| GET | `/` | 聊天页面 |
| GET | `/health` | 健康检查 |
| GET | `/channels` | 预置渠道模板 |
| POST | `/proxy` | 转发到白名单上游（解决 CORS） |

## 📄 License

MIT
