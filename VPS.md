# VPS 这一块：日本 NAT 加台湾住宅节点，以及订阅怎么维护

2026-09 下旬实际做过的事。主线是在日本 NAT 机上加一个“分流节点”：GPT / Claude 走台湾 HiNet 住宅代理，别的流量走 NAT 机自己的出口。

不写密码、UUID、Reality 公私钥、short_id、SS 密码、服务器 IP、住宅代理的域名、订阅地址。下面全用占位符，比如 `<UUID>`、`<PASSWORD>`、`<NAT_IP>`。

## 整体布局

| 机器 | 系统 / 程序 | 用途 |
|---|---|---|
| DMIT 洛杉矶（主 VPS） | Debian 13，sing-box 1.11，systemd | 自己的节点；分流节点 ATT / 夏威夷 / Cox / Wave（100G）；出订阅 `/sub-self`（Clash yaml 和 Shadowrocket vless 列表） |
| nat2 日本 NAT | Alpine，sing-box 1.13，openrc | 只开了一段转发端口。分流节点 `nat2-日本Home（100G）`、`nat2-台湾Hinet（100G）`、`nat2-US机房固定IP` |
| 中转机 `<RELAY_IP>` | Alpine，busybox | 另一份订阅 `/sub`，用 `nc -lk -e` 跑一个 shell 脚本出文件 |

“（100G）”的意思是 AI 出口用的是按流量计费的住宅代理，一个月 100G。

## 设计原则

- 客户端只在 `Proxy` 一个组里选节点。别的组只是 `Proxy` / `DIRECT` 二选一。
- 分流放在服务端做。节点本身决定 AI 走住宅、其余走机房，客户端不用写 AI 规则也不用切换。
- 国内直连靠规则末尾的 `GEOSITE,CN` 和 `GEOIP,CN`，再 `MATCH` 到代理。
- 不用 `PROCESS-NAME`（手机上无效，桌面上也不稳）。不做组套组。
- `GLOBAL` 组把所有节点都列出来。客户端切到全局模式时只显示 `GLOBAL`，不列的话没法手动挑节点。

## 核心：在 nat2 上加台湾 HiNet 分流节点

### 为什么这么接

- 住宅流量只有 100G，只让 GPT / Claude 走住宅，其它（视频、下载、更新）走 NAT 机自己的出口。
- 台湾那台住宅节点拒绝大陆直连，必须挂在 VPS 后面当上游。
- 同一个住宅节点给了 SS 和 VLESS Reality 两种接入。从 nat2 和 DMIT 各测过一次：带宽差不多（60–75 Mbps，瓶颈在住宅那头），但 SS 每个新连接的首字节快大约 0.25 秒，VLESS Reality 握手要多几个来回（nat2 到台湾单程 RTT 约 155 ms）。所以上游用 SS。
- 实测 nat2 到台湾的延迟和洛杉矶到台湾差不多（都是 150 多 ms），日本机并没有地理优势。选哪台当中转看自己到哪台近。

### 1. 找一个空闲端口

NAT 机只有商家分配的那一段端口能从外面进来。先看哪个没被占：

```sh
netstat -tuln | grep -E ':2104[0-9] '
```

没有输出的就是空的。下面记作 `<NEW_PORT>`。

### 2. 复制一个现成的 Reality 入站

直接复制 `nat2-jp-in`，只改 `tag` 和端口。UUID、Reality 私钥、short_id、SNI 都沿用，这样订阅里那条也只需要改端口和名字。

```json
{
  "type": "vless",
  "tag": "nat2-tw-in",
  "listen": "::",
  "listen_port": <NEW_PORT>,
  "users": [{ "uuid": "<UUID>", "flow": "xtls-rprx-vision" }],
  "tls": {
    "enabled": true,
    "server_name": "addons.mozilla.org",
    "reality": {
      "enabled": true,
      "handshake": { "server": "addons.mozilla.org", "server_port": 443 },
      "private_key": "<PRIVATE_KEY>",
      "short_id": ["<SHORT_ID>"]
    }
  }
}
```

### 3. 加住宅出站（Shadowsocks）

SS 链接 `ss://<base64>@<RESIDENTIAL_HOST>:<PORT>` 里的 base64 解出来是 `方法:密码`。

```json
{
  "type": "shadowsocks",
  "tag": "res-tw",
  "server": "<RESIDENTIAL_HOST>",
  "server_port": <RESIDENTIAL_PORT>,
  "method": "aes-128-gcm",
  "password": "<PASSWORD>"
}
```

### 4. 路由规则

顺序很重要，从上往下第一条命中就走。新入站要加进所有“公共”规则的 `inbound` 列表，再单独加一条 AI 规则指向 `res-tw`。

```json
"route": {
  "rules": [
    { "inbound": ["nat2-jp-in", "nat2-usdc-in", "nat2-tw-in"], "action": "sniff" },

    { "inbound": ["nat2-jp-in", "nat2-usdc-in", "nat2-tw-in"],
      "network": "udp", "port": 443,
      "domain_suffix": ["<AI 域名集>"], "action": "reject" },

    { "inbound": ["nat2-jp-in", "nat2-usdc-in", "nat2-tw-in"],
      "domain": ["persistent.oaistatic.com", "downloads.claude.ai", "videos.openai.com", "..."],
      "outbound": "direct-out" },

    { "inbound": ["nat2-jp-in"],  "domain_suffix": ["<AI 域名集>"], "ip_cidr": ["<AI 网段>"], "outbound": "res-jp" },
    { "inbound": ["nat2-tw-in"],  "domain_suffix": ["<AI 域名集>"], "ip_cidr": ["<AI 网段>"], "outbound": "res-tw" },
    { "inbound": ["nat2-usdc-in"], "domain_suffix": ["<AI 域名集>"], "ip_cidr": ["<AI 网段>"], "outbound": "res-usdc" },

    { "inbound": ["nat2-jp-in", "nat2-usdc-in", "nat2-tw-in"], "outbound": "direct-out" }
  ]
}
```

说明：

- `sniff`：入站是 VLESS，sing-box 要先嗅探出 TLS 的 SNI，后面的域名规则才有东西可比。
- 拒绝 AI 域名的 UDP 443：浏览器会先试 QUIC。QUIC 走 UDP，SS 上游对它不友好，容易卡住或者绕去别的出口。拒掉以后浏览器会退回 TCP，走得就是住宅。
- 例外直连：大文件和静态资源走本机出口，省住宅流量。当前 10 个：`persistent.oaistatic.com`、`downloads.claude.ai`、`videos.openai.com`，以及几个 OpenAI / Anthropic 官网用的 CDN 主机（Azure blob / azureedge、imgix、ghost.io、b-cdn）。这一条必须放在 AI 规则前面，因为 `oaistatic.com`、`claude.ai` 本身在 AI 集合里。
- AI 集合：`chatgpt.com`、`chat.com`、`openai.com`、`oaistatic.com`、`oaiusercontent.com`、`sora.com`、`anthropic.com`、`claude.ai`、`claude.com`、`clau.de`、`claudeusercontent.com` 等后缀，加几个精确域名、一个关键词 `chatgpt-async-webps-prod`，以及 OpenAI / Anthropic 公布的 4 个网段（v4 + v6）。一共 39 条左右。三个分流入站用同一份。
- 最后一条兜底直连。

改的时候用 jq 在副本上改，别手写整份 json。密码放在临时文件里用 `--rawfile` 读，不要出现在命令行上：

```sh
mkdir -p /tmp/twsplit && cd /tmp/twsplit
jq --rawfile pw pw.txt -f add-tw.jq /etc/sing-box/config.json > config.new
rm -f pw.txt
```

### 5. 检查、备份、重启

```sh
sing-box check -c /tmp/twsplit/config.new
cd /etc/sing-box
cp -p config.json config.json.bak.twsplit-$(date -u +%Y%m%dT%H%M%SZ)
cat /tmp/twsplit/config.new > config.json     # 保留原来的 600 权限
rc-service sing-box restart                    # Alpine / openrc；systemd 上是 systemctl restart sing-box
netstat -tln                                    # 所有入站端口都还在
tail -n 50 /var/log/sing-box.log               # 没有新的 error / fatal
rm -rf /tmp/twsplit
```

改之前可以先确认“新配置删掉新增部分之后和旧配置完全一样”，保证没有顺手改到别的。

### 6. 验证

在本地用 mihomo 挂上这个节点开一个临时端口（或者直接用客户端），测三样：

```sh
X="-x http://127.0.0.1:<LOCAL_PORT>"
curl -s $X https://ipinfo.io/json                 # 非 AI：出口应是 <NAT_IP>，日本机房
curl -s $X https://chatgpt.com/cdn-cgi/trace      # AI：ip= 住宅 IP，loc=TW
curl -s $X -o /dev/null -w '%{http_code}\n' https://api.anthropic.com/v1/models   # 401 = 通了（只是没带 key）
```

`chatgpt.com`、`claude.ai` 首页用 curl 打开是 403，响应头里有 `cf-mitigated: challenge`。那是 Cloudflare 对 curl 的人机验证，不是 IP 被封，浏览器里正常。

顺手把老节点也测一遍，确认重启没有弄坏别的入站。

### 7. 发布到订阅

Clash yaml：复制 `nat2-日本Home（100G）` 那一段，只改名字和端口：

```yaml
- name: nat2-台湾Hinet（100G）
  type: vless
  server: <NAT_IP>
  port: <NEW_PORT>
  uuid: <UUID>
  udp: true
  tls: true
  network: tcp
  flow: xtls-rprx-vision
  servername: addons.mozilla.org
  client-fingerprint: chrome
  reality-opts:
    public-key: <PUBLIC_KEY>
    short-id: <SHORT_ID>
```

然后在 `Proxy` 组里放到 `nat2-日本Home（100G）` 后面，`GLOBAL` 组里也加上。别的组不动。

Shadowrocket 的 vless 列表加一行，放在日本 Home 那行下面，名字要 URL 编码：

```
vless://<UUID>@<NAT_IP>:<NEW_PORT>?encryption=none&flow=xtls-rprx-vision&security=reality&sni=addons.mozilla.org&fp=chrome&pbk=<PUBLIC_KEY>&sid=<SHORT_ID>&type=tcp#nat2-%E5%8F%B0%E6%B9%BEHinet%EF%BC%88100G%EF%BC%89
```

上线按下面的“部署步骤”走。

## 订阅服务和流量显示

- `/sub-self` 由一个 python 小服务出文件。按 User-Agent 分：Clash / mihomo 给 yaml，Shadowrocket 给 vless 列表。
- 响应头带 `subscription-userinfo: upload=…; download=…; total=…; expire=…`，客户端就能显示“已用 / 总量 / 到期”。
- 数据来自一个采集脚本，cron 每 5 分钟跑一次，读网卡 `/sys/class/net/eth0/statistics` 的 rx/tx，按账期累加，处理重启（boot_id 变了就从 0 接着加），写 `usage.json`。
- 配置 `/etc/subscription-usage.conf`：

| 项 | 值 | 意思 |
|---|---|---|
| `MODE` | `max` | DMIT 按单向最大计费，取 max(rx, tx)；双向计费用 `sum` |
| `TOTAL_BYTES` | 1 TiB | 月流量 |
| `RESET_DAY` | 24 | 每月 24 号服务器本地时间 0 点重置 |
| `OFFSET_BYTES` / `OFFSET_CYCLE` | 100 GiB / 2026-09-24 | 校准：脚本是账期中途装的，把面板上已用的量补进去，只在那一个账期生效 |
| `DMIT_RENAME` | 1 | 把节点名 `dmit` 显示成 `dmit（已用NNNG/1TB）` |

有的客户端不显示响应头，所以才有改名这一招。另有一个 `INFO_NODE` 可以塞一个假节点显示用量，现在关着，只留响应头和改名。

## 部署步骤（每次改订阅都这样）

1. 在副本上改，不直接改线上文件。
2. `mihomo -t -d <临时目录> -f 新文件.yaml`，看到 `test is successful`。
3. `sha256sum -c` 对线上文件，确认它还是你以为的那一版（防止别人刚改过被你覆盖）。
4. 带时间戳备份：`cp -p sub.yaml sub.yaml.<时间>-before-<改了什么>`。
5. 用 `cat 新文件 > 线上文件` 覆盖，保持原来的权限和属主。
6. 分别用 `curl -A clash.meta` 和 `curl -A Shadowrocket/2000` 拉一次，和文件逐字节比对（服务端改了名字的话先把名字换回来再比）。确认 userinfo 头还在。
7. 本地留一份和线上一致的副本。

## 踩过的坑

- **国内站走直连却打不开**：比如 `sheincorp.cn`，它 CNAME 到 Akamai。一开始怀疑是 fallback-filter 把境外 IP 换成了 8.8.8.8 的结果。实际查下来，`.cn` 本来就命中 `nameserver-policy` 里的 `geosite:cn`，只用国内 DNS，不会走 fallback；国内 DNS 带大陆 ECS 时返回的是阿里云深圳的地址，规则也本来就是直连。服务端都正常，最后按用户意思改成 `DOMAIN-SUFFIX,sheincorp.cn,Proxy`。教训：先用 mihomo 开 debug 日志看它实际问了哪个 DNS、命中了哪条规则，再动配置。另外 mihomo 文档里 `fallback-filter.domain` 的意思是“这些域名只用 fallback”，别往里加国内域名。
- **QUIC**：不拒 AI 域名的 UDP 443，浏览器走 QUIC 时分流会不稳定。
- **大文件例外**：不加直连例外的话，下载 Claude 安装包、看 OpenAI 的视频都会吃住宅流量。
- **上游死了节点也“能连”**：`nat2-US机房固定IP` 的 AI 出口是另一台 VPS 上的 Reality，那台挂了以后，这个节点普通网站照常，只有 GPT / Claude 打不开。排查时先分别测非 AI 和 AI 两个出口。
- **NAT 机的端口**：TCP 能连上不代表后面有服务，商家网关可能先接下连接再断。要做 TLS 握手或者真走一遍代理才算数。
- **Alpine 上别用 `pkill -f`**：模式会匹配到执行它的那个 shell，把自己也杀了。用 `ps | grep "[x]"` 取 pid 再 kill。
