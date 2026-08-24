# 手把手自建翻墙链：AnyTLS + Reality 双协议·中转+落地架构（2026-08 版）

> 跟着做就能搭出一条「快 + 身份干净」的私人链路。每一步都有三样东西：**这步在干什么 → 复制什么命令 → 看到什么算成功**。
> 全文所有 IP、域名、密码、密钥都是占位符（如 `1.2.3.4`、`example.com`、`【密码】`），替换成你自己的。
> 适合：会用复制粘贴的新手。不适合：想一键脚本三秒搞定的（这套追求的是知其所以然 + 长期稳定）。

---

## 〇、你将得到什么 & 为什么这样设计

```
你的手机/电脑 → 中转机（优化线路 VPS，只负责快）→ 落地机（住宅 IP VPS，只负责身份干净）→ 目标网站
```

**为什么是两台机器**：优化线路 VPS 速度快但 IP 是机房段（AI 网站风控不待见）；住宅 IP VPS 身份干净但回国线路差。让两台各管一头，快和干净就都有了。

**为什么是双协议**：AnyTLS 和 VLESS+Reality 两条线同时跑，同一个出口 IP。任何一条被针对，一键切另一条，AI 风控无感。

**为什么要自有域名 + 真实网站（回退站）**：防火墙会「敲门试探」你的服务器。没有回退站，敲门就看到代理；有了它，敲门看到的是一个真实的个人网站——证书、域名、内容三合一全是你自己的。

**预算参考**：中转机约 ¥70-90/年起步，落地住宅机约 ¥150-250/月（按 2026-08 行情），域名约 ¥10-70/年。

---

## 一、准备工作

1. 密码管理器（1Password 或同类）——**本教程所有密码/密钥/Token 都要求当场存进去**
2. SSH 工具：Windows 推荐 Termius（汉化版即可）或系统自带 PowerShell
3. 一个账号：Cloudflare（免费计划即可）
4. 一个域名：随便什么后缀（Spaceship / Namecheap / 阿里云万网都行），本教程以 `example.com` 代称

---

## 二、采购两台 VPS

**在干什么**：买两台机器，一台当「快递中转站」，一台当「美国家庭住址」。

| | 中转机 | 落地机 |
|---|---|---|
| 要什么 | 到中国的优化线路（CN2 GIA 等），配置随意最低 | **IP 必须是住宅/ISP 类型**，带宽 50-100M 足够 |
| 不要什么 | 高配置（纯转发用不上） | 机房 IP（DMIT/搬瓦工这类只能当中转） |
| 地区 | 美西（洛杉矶/圣何塞，离中国近） | 和中转机同城最好 |

**✅ 验收**：拿到两台机器的 IP、root 密码/密钥，并且两台能互相 ping 通。落地机 IP 用 [ipdata.co](https://ipdata.co) 查，type 显示 **isp**（不是 hosting/business）才算买对。

> ⚠️ 记下两个 IP，全文用 `1.2.3.4` 代指中转机、`5.6.7.8` 代指落地机。

---

## 三、两台机器基础加固（两台都执行）

**在干什么**：改 SSH 端口、上密钥登录、装防火墙——裸奔的 VPS 上线几分钟就会被扫。

```bash
# 两台机器都执行（先连上，再逐段粘贴）
apt update && apt upgrade -y
apt install -y ufw fail2ban unattended-upgrades curl socat cron
systemctl enable --now unattended-upgrades   # 自动安全更新，零打扰
```

**改 SSH 端口**（以 23456 为例，自己换一个）：

```bash
sed -i 's/#\?Port .*/Port 23456/' /etc/ssh/sshd_config
systemctl restart ssh
```

> ⚠️ 改完**先别关当前窗口**，新开一个连接确认能进，再关旧的。同时检查 `/etc/ssh/sshd_config.d/` 下有没有覆盖文件强行开着密码登录（有的商家镜像会埋），用 `sshd -T | grep -i passwordauthentication` 看实际生效值。

**防火墙**：

```bash
# 落地机（5.6.7.8）
ufw allow 23456/tcp                                   # SSH
ufw allow from 1.2.3.4 to any port 8444               # 只放中转机进 AnyTLS 分流口
ufw allow from 1.2.3.4 to any port 8443               # 只放中转机进 Reality
ufw enable

# 中转机（1.2.3.4）
ufw allow 23456/tcp
ufw allow 443/tcp && ufw allow 8443/tcp               # 这两个对全网开放（客户端要连）
ufw enable
```

**✅ 验收**：`ufw status` 规则齐全；新端口能 SSH 上；两台机互访正常。

---

## 四、域名接入 Cloudflare

**在干什么**：让域名由 CF 解析，后面签证书要用它的 API。

1. CF 首页 → **Onboard a domain** → 输入你的域名 → 选 Free
2. 拿到两个 NS 地址（形如 `xxx.ns.cloudflare.com`）→ 去你的域名商后台把 DNS 服务器改成这两个
3. CF 的 DNS 页加 3 条 A 记录，全部**灰云（仅 DNS，千万别点成橙云）**：
   - `@` → `1.2.3.4`（中转机 IP）
   - `www` → `1.2.3.4`
   - `node` → `1.2.3.4`
4. CF → My Profile → API Tokens → 用「Edit zone DNS」模板建 Token，权限范围限定本域名。**Token 和 Account ID 立刻存密码管理器**

**✅ 验收**：CF 显示域名 Active；`ping node.example.com` 能解析到中转机 IP。

---

## 五、落地机：签真证书

**在干什么**：给域名签一张 Let's Encrypt 真证书。因为家宽商一般禁止开 80 端口，所以用 DNS-01 方式（通过 CF API 证明域名是你的）。

```bash
# 📍 落地机上执行
curl https://get.acme.sh | sh
export CF_Token="【你的Token】"
export CF_Account_ID="【你的Account ID】"
~/.acme.sh/acme.sh --issue --dns dns_cf -d node.example.com -d example.com -d www.example.com
~/.acme.sh/acme.sh --install-cert -d node.example.com \
  --key-file /etc/sing-box/private.key \
  --fullchain-file /etc/sing-box/cert.pem \
  --reloadcmd "systemctl restart sing-box && systemctl reload nginx"
```

**✅ 验收**：看到绿色的 `Cert success`；`/etc/sing-box/cert.pem` 存在。证书会自动续期，不用管了。

---

## 六、落地机：回退站（真实网站）

**在干什么**：放一个真实的小型个人网站。探测者敲门时看到的就是它；Reality 的伪装目标也是它。

```bash
# 📍 落地机上执行
apt install -y nginx libnginx-mod-stream
rm -f /etc/nginx/sites-enabled/default    # 删掉默认站点（灭 80 监听，守商家规矩）
mkdir -p /var/www/fallback
# 把你的静态网站文件放进 /var/www/fallback/（一个多页纯文字个人博客即可：
# 首页 + 几篇能点进去的完整文章 + 关于页，内容和落地 IP 所在地相符更自然）
```

`/etc/nginx/conf.d/fallback.conf`：

```nginx
server {
  listen 127.0.0.1:9443 ssl;
  http2 on;
  server_name example.com www.example.com;
  ssl_certificate     /etc/sing-box/cert.pem;
  ssl_certificate_key /etc/sing-box/private.key;
  ssl_stapling on;
  ssl_stapling_verify on;
  resolver 8.8.8.8 1.1.1.1 valid=300s;
  resolver_timeout 5s;
  root /var/www/fallback;
  index index.html;
}
```

**✅ 验收**：`nginx -t` 通过，`systemctl reload nginx` 无报错。（此时网站只在本机 9443，外网还看不到，正常。）

---

## 七、落地机：sing-box 跑 AnyTLS

**在干什么**：装 sing-box，只开 AnyTLS 一个入口，只听本机 8445（外面由 nginx 分流罩着）。

```bash
# 📍 落地机上执行
curl -fsSL https://sing-box.app/install.sh | sh
```

`/etc/sing-box/config.json` 整段写入（【密码】换成 32 位随机密码，存密码管理器）：

```json
{
  "log": { "level": "warn" },
  "inbounds": [ {
    "type": "anytls",
    "listen": "127.0.0.1",
    "listen_port": 8445,
    "users": [ { "name": "u", "password": "【密码】" } ],
    "padding_scheme": [
      "stop=9", "0=45-45", "1=120-380",
      "2=380-560,c,480-950,c,480-950,c,480-950",
      "3=11-11,480-950", "4=480-950", "5=480-950",
      "6=480-950", "7=480-950", "8=480-950"
    ],
    "tls": { "enabled": true,
      "certificate_path": "/etc/sing-box/cert.pem",
      "key_path": "/etc/sing-box/private.key" }
  } ],
  "outbounds": [ { "type": "direct" } ]
}
```

```bash
sing-box check -c /etc/sing-box/config.json && systemctl enable --now sing-box
ss -tlnp | grep 8445   # 应看到 127.0.0.1:8445 由 sing-box 监听
```

> `padding_scheme` 是自定义流量填充方案——官方默认填充是公开指纹，换成私有一套等于换了件衣服。服务端会自动把方案下发给客户端，客户端不用配置。

---

## 八、落地机：Xray 跑 VLESS+Reality

**在干什么**：装 Xray（Reality 的亲生父母家），开 8443 入口，伪装目标指向自己的回退站（「偷自己」）。

```bash
# 📍 落地机上执行
bash -c "$(curl -L https://github.com/XTLS/Xray-install/raw/main/install-release.sh)" @ install
xray x25519     # 生成密钥对：PrivateKey / Password / Hash32 三栏
openssl rand -hex 8   # 生成 short_id
```

> 🔑 **铁律：密钥只生成这一次，立刻把 PrivateKey 和 Password 两栏存进密码管理器。** UUID 也用密码管理器生成一个（或 `cat /proc/sys/kernel/random/uuid`）。

`/usr/local/etc/xray/config.json`：

```json
{
  "log": { "loglevel": "warning" },
  "inbounds": [ {
    "listen": "0.0.0.0", "port": 8443, "protocol": "vless",
    "settings": {
      "clients": [ { "id": "【UUID】", "flow": "xtls-rprx-vision" } ],
      "decryption": "none"
    },
    "streamSettings": {
      "network": "tcp", "security": "reality",
      "realitySettings": {
        "dest": "127.0.0.1:9443",
        "serverNames": [ "example.com" ],
        "minClientVer": "1.8.2",
        "privateKey": "【PrivateKey】",
        "shortIds": [ "【short_id】" ]
      }
    }
  } ],
  "outbounds": [ { "protocol": "freedom" } ]
}
```

```bash
/usr/local/bin/xray run -test -config /usr/local/etc/xray/config.json && systemctl enable --now xray
ss -tlnp | grep 8443
```

> 字段说明：`dest` 指自己的回退站 = 伪装零借用痕迹；`minClientVer: "1.8.2"` 是给 mihomo 客户端的兼容门票（它自报版本硬编码 1.8.2，新版 Xray 默认门禁会拦）；`xray x25519`（25.8+）生成的是抗量子密钥，Password 栏就是客户端要填的「公钥」。

---

## 九、落地机：nginx SNI 分流

**在干什么**：8444 端口站岗分流——报出 `node.example.com` 的连接送去 AnyTLS，其他（探测者）送去回退站。

`/etc/nginx/nginx.conf` **顶层**（和 `http {}` 平级）追加：

```nginx
stream {
  map $ssl_preread_server_name $upstream {
    node.example.com 127.0.0.1:8445;
    default          127.0.0.1:9443;
  }
  server {
    listen 8444;
    proxy_pass $upstream;
    ssl_preread on;
  }
}
```

```bash
nginx -t && systemctl restart nginx
ss -tlnp | grep -E '8444|8445|9443'
# 验收：8444=nginx（对网），8445=sing-box 与 9443=nginx 都只在本机，没有 80 端口
```

---

## 十、中转机：双路转发

**在干什么**：中转机只当搬运工，两个端口分别转两条线。

```bash
# 📍 中转机上执行，整段复制
cat > /etc/systemd/system/relay.service << 'EOF'
[Unit]
Description=relay to landing (AnyTLS)
After=network.target
[Service]
ExecStart=/usr/bin/socat TCP4-LISTEN:443,fork,reuseaddr TCP4:5.6.7.8:8444
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF

cat > /etc/systemd/system/relay-reality.service << 'EOF'
[Unit]
Description=relay to landing (Reality)
After=network.target
[Service]
ExecStart=/usr/bin/socat TCP4-LISTEN:8443,fork,reuseaddr TCP4:5.6.7.8:8443
Restart=always
RestartSec=3
[Install]
WantedBy=multi-user.target
EOF

systemctl enable --now relay relay-reality
systemctl status relay relay-reality --no-pager | grep Active   # 两个都 active
```

---

## 十一、客户端配置

### mihomo 系（Clash Verge Rev 等，YAML 片段）

```yaml
proxies:
  - name: "家宽-AnyTLS"
    type: anytls
    server: 1.2.3.4          # 中转机 IP
    port: 443
    password: "【密码】"
    sni: node.example.com    # 真证书，不写 skip-cert-verify
    udp: true
  - name: "家宽-Reality"
    type: vless
    server: 1.2.3.4
    port: 8443
    uuid: "【UUID】"
    flow: xtls-rprx-vision
    tls: true
    servername: example.com
    client-fingerprint: chrome
    reality-opts:
      public-key: "【Password栏】"
      short-id: "【short_id】"
      support-x25519mlkem768: true   # 抗量子密钥时加；经典密钥则删
    udp: true
```

### iOS Loon（`[Proxy]` 段）

```ini
家宽-AnyTLS = AnyTLS, 1.2.3.4, 443, "【密码】", sni=node.example.com, skip-cert-verify=false, udp=true, block-quic=false
家宽-Reality = VLESS, 1.2.3.4, 8443, "【UUID】", transport=tcp, flow=xtls-rprx-vision, public-key="【Password栏】", short-id=【short_id】, over-tls=true, sni=example.com, udp=true
```

**✅ 总验收**：
1. 两节点延迟测试均绿
2. 浏览器开 `https://example.com` → 你的回退站正常、无证书警告
3. [ipdata.co](https://ipdata.co) 显示落地机住宅 IP（两节点同一个）
4. [browserleaks.com/dns](https://browserleaks.com/dns) 无真实 IP 泄露
5. 目标服务（如 AI 站点）实测无验证码、无降智

---

## 十二、踩坑实录（本教程最值钱的部分）

1. **家宽商上游可能封入站 443**：`ss` 显示监听但外网不通、服务端日志零连接。判据：`tcpdump` 抓包为 0。这就是本教程 AnyTLS 落地用 8444（nginx 分流口）的原因——中转机 443 照开，客户端无感
2. **微软证书事件（2026-07 起）**：`www.microsoft.com` 更换超大证书（TLS Certificate 记录 8273 字节），Reality 握手重放路径处理不了，握手永不完成（日志特征：`Certificate: 8273` + `isHandshakeComplete: false`）。**微软系 dest（含 bing）不可再用**；排障神器：Xray 的 `realitySettings` 加 `"show": true` + 日志调 debug，握手解剖一目了然（查完调回 warning）
3. **Xray ≥26.7.11 的 minClientVer 门禁**：默认最低 26.3.27，mihomo 硬编码自报 1.8.2 会被拦在认证阶段。解法：服务端显式 `minClientVer: "1.8.2"`（本教程配置已含）
4. **密钥纪律**：多生成一次密钥就多一副，两端贴串后症状是「配置看起来全对但认证失败」。校验：`xray x25519 -i <私钥>` 推导配对公钥比对
5. **排障方法论**：换密钥/换版本/换指纹之前先看握手日志；同客户端有可用同类节点时逐项 diff 字段；「真不通」要用真实流量判（全局模式 + curl），延迟测试会误报
6. **dest 套 CDN = 被偷流量**：dest 域名挂 Cloudflare 这类开放 CDN 时，攻击者用自己的同 CDN 域名做 SNI 即可无鉴权借你的 VPS 回源。判一个域名是否套 CF：访问 `https://域名/cdn-cgi/trace` 有返回即中招。优先级：自有站 > 未套 CDN 的直连站 > CDN 域名。没有自有域名时选 dest 的工具：本机跑 [RealiTLScanner](https://github.com/XTLS/RealiTLScanner) 扫同段邻居（**别在 VPS 上跑**，多线程扫描可能被商家判定攻击而停机）+ [RealityChecker](https://github.com/V2RaySSR/RealityChecker) 筛掉套 CDN/不可达的

## 十三、优化与「明确不做」

**建议做**：自定义 padding_scheme（已在配置里）/ 真证书 + 回退站 / OCSP Stapling / BBR（`net.core.default_qdisc=fq` + `net.ipv4.tcp_congestion_control=bbr` 写入 `/etc/sysctl.d/99-bbr.conf` 后 `sysctl --system`）/ unattended-upgrades

**明确不做**（都查过证据）：TCP Fast Open（两侧内核均有 bug issue）/ `limitFallback*`（官方警告是一种特征）/ sockopt 调参（默认已合理）/ 装 Web 面板（攻击面 + 面板默认装最新版内核踩门禁）

---

*基于 2026-08-24 一次真实搭建整理。协议生态变化快，动工前先验证时效性。*

## 致谢与参考

本教程站在这些文章、讨论和文档的肩膀上，特此致谢 🙏

**架构参考**
- [LINUX DO：小丸子《自建家宽节点+中转站教程》](https://linux.do/t/topic/2086346)——中转+落地架构的原型参考
- [LINUX DO：DMIT + 3x-ui + Reality + 自有域名实战记录](https://linux.do/t/topic/2625350)——域名+证书+回退站路线的互证

**官方文档**
- [XTLS/REALITY 官方 README](https://github.com/XTLS/REALITY)（dest 最低标准与加分项的权威出处）
- [Xray 配置文档](https://xtls.github.io/config/) / [sing-box AnyTLS 入站文档](https://sing-box.sagernet.org/configuration/inbound/anytls/) / [mihomo wiki](https://wiki.metacubex.one/)
- [anytls-go 协议原文](https://github.com/anytls/anytls-go/blob/main/docs/protocol.md) 与 [AnyTLS Wiki](https://anytls.wiki/docs/server/)（padding 下发机制的设计说明）
- [Xray/mihomo Reality 配置教程（argsment）](https://core-tutorial.argsment.com/zh/xray/reality)

**关键社区病例**
- [LINUX DO：3x-ui Reality 节点 timeout——Microsoft 伪装站证书过大事件](https://linux.do/t/topic/2571392)（「坑 2」的同案首发记录，感谢楼主抓出 8273 字节这个关键证据）
- [LINUX DO：搬瓦工 IP 一月被墙两次求助帖](https://linux.do/t/topic/2757894)（dest 选型集体智慧：勿用谷歌系/CF 套壳域名，「偷自己」多人多年实证）
- [LINUX DO：x-ui Reality 在 Shadowrocket 正常、Clash 系全灭](https://linux.do/t/topic/2692871)（minClientVer 门禁实战记录）
- [LINUX DO：小心自己搭建的 VPS 被别人偷流量 + Reality 伪装域名选择教程](https://linux.do/t/topic/2736045)（「坑 6」的机制与工具链）
- [idcflare：高位端口批判](https://idcflare.com/t/topic/76439)（端口与伪装的讨论）
- [V2EX：近日 AnyTLS 流量已被识别或通报](https://www.v2ex.com/t/1214944)（2026-05 指纹识别事件）

**上游 issue（兼容性问题的一手证据）**
- [mihomo#2967](https://github.com/MetaCubeX/mihomo/issues/2967)（ClientVer 硬编码与 wontfix）/ [mihomo#3042](https://github.com/MetaCubeX/mihomo/issues/3042) / [Xray-core#6477](https://github.com/XTLS/Xray-core/issues/6477)（版本兼容事件三方视角）
- [Xray-core Discussions #2256](https://github.com/XTLS/Xray-core/discussions/2256)（RPRX 本人的 dest 选型指导）/ [#2308](https://github.com/XTLS/Xray-core/discussions/2308)（dest 密钥曲线导致握手失败）
- [sing-box#4023](https://github.com/SagerNet/sing-box/issues/4023)（sing-box Reality 服务端兼容性病例）
