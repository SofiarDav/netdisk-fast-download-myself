# netdisk-fast-download 部署与维护指南

> **项目来源**：Fork 自 [qaiu/netdisk-fast-download](https://github.com/qaiu/netdisk-fast-download)  
> **维护仓库**：[SofiarDav/netdisk-fast-download-myself](https://github.com/SofiarDav/netdisk-fast-download-myself)  
> **文档版本**：2026-10-07  
> **适用环境**：Linux (WSL/原生) + Docker + Cloudflare Tunnel

---

## 1. 项目目的

本项目旨在**自建一个多网盘分享链接解析服务**，将各大网盘（蓝奏云、移动云、小飞机盘、123云盘等）的分享链接转为直链，支持：

- Web 界面可视化解析
- API 接口集成（302 跳转 / JSON 直链）
- 文件夹解析、缓存机制、访问统计
- 认证参数加密传递（解决夸克/UC/123 等需登录网盘）

**核心诉求**：个人使用、零成本、免维护、隐藏源站 IP、自动 HTTPS。

---

## 2. 架构概览

```
┌─────────────────────────────────────────────────────────────┐
│ 客户端 (浏览器 / curl / aria2 / 手机)                          │
└──────────────────────────┬──────────────────────────────────┘
                           │ HTTPS
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ Cloudflare Edge (CDN / WAF / DDoS 防护 / 自动证书)             │
│   - 隐藏源站真实 IP                                          │
│   - 免费 HTTPS (Let's Encrypt 自动续期)                       │
└──────────────────────────┬──────────────────────────────────┘
                           │ Cloudflare Tunnel (cloudflared)
                           ▼
┌─────────────────────────────────────────────────────────────┐
│ 家庭/本地服务器 (WSL Ubuntu / 原生 Linux)                      │
│   ├─ Docker: netdisk-fast-download (端口 6401)               │
│   │    ├─ Java 17 + Vert.x 4.5 后端                          │
│   │    ├─ Vue 3 前端 (内置于 Jar)                            │
│   │    ├─ H2 数据库 (文件模式，持久化到 ./db)                 │
│   │    └─ 配置文件挂载 (./resources)                         │
│   │                                                         │
│   └─ cloudflared (SysV/systemd 守护进程)                     │
│        - 建立出站隧道到 Cloudflare                           │
│        - 无需公网端口映射、无需固定公网 IP                   │
└─────────────────────────────────────────────────────────────┘
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
    直连网盘 API      走代理网盘 API      认证服务
    (蓝奏/移动云/等)    (小飞机 fj)       (123盘/夸克/UC)
```

---

## 3. 先决条件

| 组件 | 版本要求 | 说明 |
|------|----------|------|
| Docker | ≥ 20.10 | 容器运行时 |
| cloudflared | 最新 | Cloudflare Tunnel 客户端 |
| Git | ≥ 2.30 | 同步源码/配置 |
| 域名 | 1 个 | 已托管在 Cloudflare (NS 指向 CF) |
| 公网 IP / NAT | 任意 | Tunnel 仅需出站连接，不需入站端口 |

> **本地开发环境**：WSL2 Ubuntu (root) 已验证可用。原生 Linux 同理。

---

## 4. 部署步骤

### 4.1 克隆源码与准备目录

```bash
# 1. 克隆自己 Fork 的仓库 (含 GitHub Actions 构建配置)
git clone https://github.com/<你的用户名>/netdisk-fast-download-myself.git
cd netdisk-fast-download-myself

# 2. 创建运行时数据目录 (仓库外，不纳入版本控制)
mkdir -p /opt/nfd-prod/{resources,db,logs}
```

### 4.2 获取官方配置模板

```bash
# 从官方镜像提取完整配置结构
docker create --name nfd-temp ghcr.io/qaiu/netdisk-fast-download:latest
docker cp nfd-temp:/app/resources /opt/nfd-prod/
docker rm nfd-temp
```

### 4.3 修改核心配置 (`/opt/nfd-prod/resources/app-dev.yml`)

**必须修改项**：

```yaml
server:
  domainName: "https://your-subdomain.your-domain.com"  # 你的固定域名
  authEncryptKey: 'your_16_char_random_key'             # openssl rand -hex 8

# 小飞机网盘出站代理 (强烈建议配置，防 IP 封禁)
proxy:
  - panTypes: fj
    type: http        # 或 socks5
    host: YOUR_PROXY_IP
    port: YOUR_PROXY_PORT
    username: ""
    password: ""
```

**敏感文件** `/opt/nfd-prod/resources/secret.yml`：

```yaml
encrypt:
  key: "your_32_bytes_random_string"  # openssl rand -hex 16
```

> 生成密钥命令：
> ```bash
> openssl rand -hex 8   # 16 字符，用于 authEncryptKey
> openssl rand -hex 16  # 32 字符，用于 secret.yml
> ```

### 4.4 启动 Docker 服务

```bash
# 创建部署脚本 /opt/nfd-prod/deploy.sh (参考仓库 deploy.sh)
chmod +x /opt/nfd-prod/deploy.sh
/opt/nfd-prod/deploy.sh

# 验证
curl http://localhost:6401/v2/build-version
# 应返回 {"code":200,"data":"0.4.7build..."}
```

### 4.5 配置 Cloudflare Tunnel (固定域名 + 自动 HTTPS)

```bash
# 1. 安装 cloudflared
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# 2. 登录授权 (浏览器选择域名所在账号)
cloudflared tunnel login

# 3. 创建命名隧道
cloudflared tunnel create nfd-home
# 记录输出的 Tunnel ID (UUID)

# 4. 生成配置 ~/.cloudflared/config.yml
TUNNEL_ID="<你的 Tunnel ID>"
DOMAIN="nfd.your-domain.com"
cat > ~/.cloudflared/config.yml <<EOF
tunnel: $TUNNEL_ID
credentials-file: /root/.cloudflared/$TUNNEL_ID.json
ingress:
  - hostname: $DOMAIN
    service: http://localhost:6401
    originRequest:
      noTLSVerify: true
  - service: http_status:404
EOF

# 5. 路由 DNS (自动创建 CNAME)
cloudflared tunnel route dns nfd-home $DOMAIN

# 6. 安装为系统服务 (开机自启)
sudo cloudflared service install
sudo systemctl enable --now cloudflared
# WSL 无 systemd 时用: service cloudflared start

# 7. 验证
curl -I https://nfd.your-domain.com  # 应返回 HTTP/2 200
```

---

## 5. 关键配置说明

### 5.1 `app-dev.yml` 核心参数

| 参数 | 说明 | 推荐值 |
|------|------|--------|
| `server.domainName` | 生成分享链接/二维码用的公网地址 | `https://your.domain.com` |
| `server.authEncryptKey` | `auth` 参数 AES-128 加密密钥 (16 字符) | **随机生成，勿泄露** |
| `cache.duration.*` | 各网盘直链缓存时长(分钟)，`-1` 不缓存 | 蓝奏 30、小飞机 20、123盘 -1 |
| `proxy` | 出站代理，**仅小飞机 fj 强烈建议配置** | 你的代理 IP:端口 |
| `auths.ye` | 123云盘静态认证 (用户名/密码/token) | 如需大文件下载请填写 |

### 5.2 `secret.yml`

| 参数 | 说明 |
|------|------|
| `encrypt.key` | 数据库敏感字段加密密钥 (32 字节)，**必须修改默认值** |

### 5.3 网盘认证体系

| 网盘 | 认证方式 | 配置位置 |
|------|----------|----------|
| 123云盘 (ye) | 静态配置 | `app-dev.yml → auths.ye` |
| 夸克/UC/小飞机大文件 | 动态 `auth` 参数 | Web 界面「认证配置」生成 |
| 其他免登网盘 | 无需认证 | — |

> **小飞机 (fj) >100MB 必须登录**：在 Web 界面填入账号密码/Cookie，生成 `auth` 参数随请求传递。

---

## 6. 日常维护

### 6.1 更新镜像 (官方发布新版)

```bash
cd /opt/nfd-prod
./deploy.sh  # 内含 docker pull + 重启
```

### 6.2 同步上游源码 (获取新功能/修复)

```bash
cd /root/workspace/netdisk-fast-download-myself
git remote add upstream https://github.com/qaiu/netdisk-fast-download.git
git fetch upstream
git merge upstream/main   # 或 rebase
git push origin main

# 触发专属镜像构建 (可选)
git tag v0.4.x
git push origin v0.4.x
# → ghcr.io/<你的用户名>/netdisk-fast-download-myself:v0.4.x
```

### 6.3 修改配置热重载

```bash
# 编辑配置
vim /opt/nfd-prod/resources/app-dev.yml

# 通知容器重新加载 (无需重启容器)
docker exec netdisk-fast-download kill -HUP 1
# 或重启容器
docker restart netdisk-fast-download
```

### 6.4 查看日志

```bash
# 应用日志
docker logs netdisk-fast-download -f --tail 100

# Tunnel 日志
journalctl -u cloudflared -f
# 或 WSL: tail -f /var/log/cloudflared.log
```

---

## 7. 加密备份与恢复

### 7.1 备份策略

- **内容**：运行时配置 (`resources/`)、部署脚本、Tunnel 配置、源码 Commit 记录
- **加密**：`age` (X25519)，公钥加密，私钥解密
- **存储**：推送到 GitHub 专用分支 `encrypted-backup`，打标签 `backup-YYYYMMDD-HHMMSS`
- **频率**：配置变更后手动执行，或定时任务

### 7.2 备份脚本 (已内置于仓库)

```bash
# 位置: scripts/encrypt-backup.sh
# 用法: ./scripts/encrypt-backup.sh
# 需预先安装 age 并设置公钥环境变量
```

### 7.3 恢复流程

```bash
# 1. 克隆备份分支
git clone -b encrypted-backup https://github.com/<用户名>/netdisk-fast-download-myself.git restore
cd restore

# 2. 解密 (需私钥)
age -d -i $AGE_PRIVATE_KEY backup-YYYYMMDD-HHMMSS.tar.gz.age | tar -xz

# 3. 恢复文件
cp -r resources/* /opt/nfd-prod/resources/
cp cloudflared-config.yml ~/.cloudflared/config.yml
# 重启服务生效
```

> **私钥管理**：仅离线保存 (密码管理器/加密U盘/纸质)，**绝不上传任何平台**。

---

## 8. 常见问题排查

| 现象 | 可能原因 | 排查步骤 |
|------|----------|----------|
| Web 界面打不开 | 容器未启动 / 端口未监听 | `docker ps`、`ss -tlnp \| grep 6401` |
| 解析返回 500 | 代理配置格式错误 / 网盘接口变更 | 查看 `docker logs`，检查 `proxy` 字段 |
| 小飞机解析失败 | 出站 IP 被风控 / 未配置代理 / 需登录 | 配置 `proxy.fj`、Web 界面填 `auth` |
| Tunnel 502 | 本地 6401 服务未就绪 | 确认 Docker 容器健康，`curl localhost:6401` |
| 域名证书错误 | TLS 验证失败 | `originRequest.noTLSVerify: true` |
| WSL 重启后 Tunnel 不跑 | 无 systemd | 在 `.bashrc` 加入 `service cloudflared start` |

---

## 9. 安全注意事项

1. **密钥轮换**：`authEncryptKey`、`secret.yml` 定期更换 (建议半年)
2. **最小权限**：Docker 容器以非 root 用户运行 (镜像已内置 `appuser`)
3. **代理隔离**：仅高风险网盘 (小飞机) 走代理，其他直连
4. **访问控制**：如需限制访问，可在 `app-dev.yml` 开启 `rateLimit` 或接入 OAuth2 Proxy
5. **备份加密**：私钥离线保管，备份分支仅存密文
6. **依赖更新**：关注 GitHub Dependabot 告警，及时合并安全补丁

---

## 10. 关键文件清单 (运行时)

```
/opt/nfd-prod/
├── deploy.sh                 # 部署/更新脚本
├── resources/
│   ├── app-dev.yml           # 主配置 (含密钥占位)
│   ├── secret.yml            # 加密密钥 (含密钥占位)
│   ├── app.yml               # 激活配置指向
│   ├── logback.xml           # 日志配置
│   ├── server-proxy.yml      # 反代规则
│   ├── server-proxy-local.yml
│   ├── http-proxy.yml
│   ├── http-tools/           # HTTP 测试工具
│   └── ssl/                  # 自签证书 (可选)
├── db/                       # H2 数据库文件 (运行时生成)
└── logs/                     # 应用日志 (运行时生成)

~/.cloudflared/
├── cert.pem                  # Tunnel 证书 (cloudflared login 生成)
├── config.yml                # Tunnel 配置
└── <TUNNEL_ID>.json          # Tunnel 凭据
```

---

## 11. 交接清单 (给新维护者)

- [ ] 理解架构：Docker + Tunnel + Cloudflare
- [ ] 拿到域名 Cloudflare 管理权限
- [ ] 拿到 `age` 私钥 (解密备份用)
- [ ] 会执行 `deploy.sh` 更新镜像
- [ ] 会修改 `app-dev.yml` 调整缓存/代理/认证
- [ ] 会查看日志定位问题
- [ ] 知道如何同步上游源码并触发构建
- [ ] 知道如何恢复加密备份

---

## 12. 参考链接

- 官方文档：https://nfd-parser.github.io/
- API 文档：https://nfdparser.apifox.cn/
- 公益实例：https://lz.qaiu.top (仅供测试，生产请自建)
- Cloudflare Tunnel 文档：https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/
- age 加密工具：https://github.com/FiloSottile/age

---

> **维护者备注**：本服务运行在个人家庭网络环境，依赖 Cloudflare Tunnel 穿透，**无公网端口映射、无固定 IP 依赖**。核心风险点在于「出站 IP 信誉」影响小飞机等网盘解析，**务必维护好出站代理池**。