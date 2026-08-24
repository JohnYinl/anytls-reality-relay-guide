# 自建双协议翻墙链实战教程：AnyTLS + Reality 中转落地架构（2026-08 版）

> 一天从零搭完一条「中转机负责快、落地机负责身份干净」的双协议链路，并把当天踩的所有坑如实记录。
> 面向有一定动手能力的新手。所有 IP、域名、密钥均为占位符，替换成你自己的即可。

## 这套架构是什么

```
你的设备 → 中转机（优化线路 VPS，负责快）→ 落地机（家宽 VPS，负责 IP 干净）→ 目标网站
           ├ AnyTLS 线：中转:443 → 落地 nginx:8444 按 SNI 分流 → sing-box（本机 8445）
           │             探测者 → 落地 nginx 回退站（本机 9443，真实网站）
           └ Reality 线：中转:8443 → Xray:8443 → dest 指向自己的回退站（「偷自己」）
```

- **双协议异构**：AnyTLS 和 VLESS+Reality 走同一个落地出口 IP，任何一条被针对，另一条顶上，切换时出口 IP 不变（对 AI 风控无感）
- **中转只做哑转发**（socat），不跑代理协议，换机/换 IP 成本为零
- **自有域名 + 真证书 + 真实回退站**：探测者看到的证书、SNI、网站内容三位一体全是你的站，零借用痕迹

## 为什么这样设计

1. **快和干净是两件事**：优化线路 VPS（如 DMIT 这类 CN2 GIA 商家）速度快但 IP 是机房段；住宅 IP VPS 身份干净但回国线路差。拆开各管一头
2. **2026 年的协议环境**：AnyTLS 满血（真证书+自有域名+回退站）与 VLESS+Reality+Vision 是 T0 组合；单一协议被针对性识别时双协议可互为备线
3. **回退站不是摆设**：AnyTLS 服务端没有原生 fallback，必须靠 nginx stream 按 SNI 分流——客户端进 sing-box，探测者进真实网站

## 施工清单（按顺序）

### 1. 采购
- 中转机：优化线路 VPS（CN2 GIA 等），最低配即可，只跑转发
- 落地机：**真家宽/住宅 IP** VPS（商家提供的 IP 是 ISP 类型，不是机房段）
- 域名：任意便宜域名，托管到 Cloudflare（**灰云，仅 DNS**）

### 2. 两台机器加固（同一套）
```bash
apt update && apt upgrade -y
apt install -y ufw fail2ban unattended-upgrades curl socat cron
# 时区 + NTP（Reality 对时间敏感）
timedatectl set-timezone America/Los_Angeles   # 按落地机实际时区
# SSH：改非标端口、密钥登录、关密码登录
# ufw：SSH 端口 + 必要业务端口；代理端口只放中转机的 IP
```

### 3. 证书（落地机）：acme.sh + DNS-01
> 家宽商通常禁 80 端口（ToS），HTTP-01 走不通，**必须 DNS-01**。CF 上创建 API Token（Edit zone DNS 权限，限定本域名），Token 和 Account ID 存密码管理器。
```bash
curl https://get.acme.sh | sh   # 需要 cron：先 apt install -y cron
export CF_Token="***"; export CF_Account_ID="***"
~/.acme.sh/acme.sh --issue --dns dns_cf -d node.example.com -d example.com -d www.example.com
~/.acme.sh/acme.sh --install-cert -d node.example.com \
  --key-file /etc/sing-box/private.key --fullchain-file /etc/sing-box/cert.pem \
  --reloadcmd "systemctl restart sing-box && systemctl reload nginx"
```

### 4. 落地机三件套（sing-box + Xray + nginx）

**sing-box（只管 AnyTLS，只听本机）** `/etc/sing-box/config.json`：
```json
{
  "log": { "level": "warn" },
  "inbounds": [ {
    "type": "anytls", "listen": "127.0.0.1", "listen_port": 8445,
    "users": [ { "name": "u", "password": "<32位随机密码>" } ],
    "padding_scheme": [ "stop=9", "0=45-45", "1=120-380",
      "2=380-560,c,480-950,c,480-950,c,480-950", "3=11-11,480-950",
      "4=480-950", "5=480-950", "6=480-950", "7=480-950", "8=480-950" ],
    "tls": { "enabled": true,
      "certificate_path": "/etc/sing-box/cert.pem",
      "key_path": "/etc/sing-box/private.key" }
  } ],
  "outbounds": [ { "type": "direct" } ]
}
```
> `padding_scheme` 自定义填充是协议作者设计的「默认指纹逃生舱」：默认方案被识别时可换私有方案，服务端自动下发给客户端。

**Xray（只管 Reality）**：`bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install`，然后 `/usr/local/etc/xray/config.json`：
```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [ {
    "listen": "0.0.0.0", "port": 8443, "protocol": "vless",
    "settings": { "clients": [ { "id": "<UUID>", "flow": "xtls-rprx-vision" } ], "decryption": "none" },
    "streamSettings": { "network": "tcp", "security": "reality",
      "realitySettings": {
        "dest": "127.0.0.1:9443",
        "serverNames": [ "example.com" ],
        "minClientVer": "1.8.2",
        "privateKey": "<xray x25519 生成的 PrivateKey>",
        "shortIds": [ "<openssl rand -hex 8>" ]
      } }
  } ],
  "outbounds": [ { "protocol": "freedom" } ]
}
```
> 三个要点：①`dest` 指自己的回退站（「偷自己」，证书/内容全自控）②`minClientVer: "1.8.2"` 是 mihomo 客户端的兼容门票（它硬编码自报 1.8.2，Xray ≥26.7.11 默认门禁会拒）③密钥用 `xray x25519` 生成（25.8+ 输出 PQ 抗量子格式，客户端 Password 栏当公钥用；mihomo 需加 `support-x25519mlkem768: true`）

**nginx（分流 + 回退站）**：装 `nginx libnginx-mod-stream`，删掉默认站点（灭 80 监听）。
`/etc/nginx/nginx.conf` 顶层加：
```nginx
stream {
  map $ssl_preread_server_name $upstream {
    node.example.com 127.0.0.1:8445;   # 你的 AnyTLS
    default         127.0.0.1:9443;    # 其他人 → 回退站
  }
  server { listen 8444; proxy_pass $upstream; ssl_preread on; }
}
```
`/etc/nginx/conf.d/fallback.conf`（回退站，也是 Reality 的 dest）：
```nginx
server {
  listen 127.0.0.1:9443 ssl;
  http2 on;
  server_name example.com www.example.com;
  ssl_certificate     /etc/sing-box/cert.pem;
  ssl_certificate_key /etc/sing-box/private.key;
  ssl_stapling on; ssl_stapling_verify on;
  resolver 8.8.8.8 1.1.1.1 valid=300s;
  root /var/www/fallback;
  index index.html;
}
```
回退站放一个**真实内容的小型静态站**（多页个人博客最佳：文章能点进去、有微记录、有「关于」页；内容与落地 IP 所在城市/文化一致更自然）。

### 5. 中转机：双路转发
```bash
# /etc/systemd/system/relay.service → AnyTLS 线
ExecStart=/usr/bin/socat TCP4-LISTEN:443,fork,reuseaddr TCP4:<落地机IP>:8444
# /etc/systemd/system/relay-reality.service → Reality 线
ExecStart=/usr/bin/socat TCP4-LISTEN:8443,fork,reuseaddr TCP4:<落地机IP>:8443
```

### 6. 客户端
- **mihomo 系（Clash Verge Rev 等）**：AnyTLS 节点 `sni: node.example.com`（真证书，不开 skip-cert-verify）；VLESS 节点 `servername: example.com`、`flow: xtls-rprx-vision`、`client-fingerprint: chrome`、PQ 密钥时加 `support-x25519mlkem768: true`
- **iOS Loon**：AnyTLS / VLESS+Reality 均原生支持，字段一一对应

## 当天踩的坑（最有价值的部分）

1. **家宽商上游封入站 443**：`ss` 显示监听但外网不通、服务端日志零连接。判据：tcpdump 抓包为 0。解法：落地 AnyTLS 挪 8444（nginx 分流口），中转机 443→8444 转发，客户端零改动
2. **微软证书事件（2026-07 起）**：`www.microsoft.com` 更换超大证书（TLS Certificate 记录 8273 字节），Xray/sing-box 同宗的 Reality 握手重放路径处理不了，握手永不完成。日志特征：`Certificate: 8273` + `isHandshakeComplete: false` + `processed invalid connection`。**微软系 dest（含 bing）不可再用**；这就是「偷自己」成为终态的直接原因。排障工具：Xray `realitySettings` 加 `"show": true` + loglevel debug，握手解剖一目了然（查完记得调回）
3. **Xray ≥26.7.11 的 minClientVer 门禁**：默认最低 26.3.27，mihomo 硬编码 ClientVer=1.8.2 被拒。解法：服务端显式 `minClientVer: "1.8.2"`
4. **密钥纪律**：Reality 密钥对**只生成一次，立即进密码管理器**。多生成一次就多一副，两端贴串后症状是「配置看起来全对但认证失败」。校验工具：`xray x25519 -i <私钥>` 推导配对公钥
5. **排障方法论**：换密钥/换版本/换指纹之前，先看握手日志；同客户端里有可用同类节点时逐项 diff 字段；「真不通」的判决要用真实流量（全局模式 + curl），延迟测试会误报

## 优化与「明确不做」

**做了**：自定义 padding_scheme / 真证书 + 回退站 / nginx OCSP Stapling / BBR（`net.core.default_qdisc=fq` + `net.ipv4.tcp_congestion_control=bbr`）/ unattended-upgrades 自动安全更新

**不做**（查过证据）：TCP Fast Open（两侧内核均有 bug issue）/ limitFallback*（官方警告是一种特征）/ sockopt 调参（默认已合理）/ 装面板（攻击面 + 默认带最新版坑）

## 验收清单

- [ ] 客户端两节点延迟均绿，出口 IP 显示为落地机住宅 IP（两节点同 IP）
- [ ] 浏览器开 `https://你的域名` → 回退站正常，无证书警告
- [ ] DNS / WebRTC 泄露测试无真实 IP
- [ ] 目标服务（AI 站点等）实测无验证码、无降智
- [ ] 服务端日志级别调回 warning、show 关闭、临时测试进程与端口清理完毕

---

*基于 2026-08-24 一次真实搭建整理。协议生态变化快，动工前先验证时效性。*
