# Ubuntu 26.04 上装远程桌面和 Claude 的记录

2026-09-28，一台新装的 Ubuntu 26.04.1 桌面（GNOME，x86_64）。下面是当天实际做过的事，以及两个公开检测页规则并不一样这件事。

不写登录密码、RustDesk 公钥、出口 IP。

## 系统

- 发行版：Ubuntu 26.04.1 LTS
- 桌面：GNOME，Wayland
- 浏览器：Firefox（snap）
- 时区最后停在 `Asia/Singapore`。钟还是东八区。IPPure 那一页把 `Asia/Taipei` 和香港、澳门放在一起，记 0.6 分；`Asia/Singapore` 不在那张名单里。
- 语言最后停在繁体 `zh_TW`，浏览器语言是 `zh-TW, en-US, en`。没有单独的 `zh`，也没有 `zh-CN`。

## SSH

桌面版默认没有 SSH 服务。在机器本机的终端里：

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh.socket
```

Ubuntu 26.04 用 `ssh.socket` 在 22 端口监听。防火墙开着的话再放行：

```bash
sudo ufw allow OpenSSH
```

## Clash Verge

装的是上游 [clash-verge-rev](https://github.com/clash-verge-rev/clash-verge-rev) 2.5.6 的 `Clash.Verge_2.5.6_amd64.deb`。

Ubuntu 26 已经带 `libwebkit2gtk-4.1`。旧文里为 24.04 去补 `libwebkit2gtk-4.0` 的做法这里用不上。直接：

```bash
sudo apt install -y ./Clash.Verge_2.5.6_amd64.deb
```

这是图形程序，要在桌面里打开。订阅没有代填。

## Claude：要桌面版，不要拿命令行当桌面

两套东西不是同一个包。

| 包 | 来源 | 当天版本 | 是什么 |
|---|---|---|---|
| `claude-code` | `https://downloads.claude.ai/claude-code/apt/stable` | 2.1.274 | 终端里的 `claude` |
| `claude-desktop` | `https://downloads.claude.ai/claude-desktop/apt/stable` | 2.7032.0 | 应用菜单里的 Claude |

桌面版按官方文档装。签名密钥指纹是 `31DDDE24DDFAB679F42D7BD2BAA929FF1A7ECACE`，对上了再加入软件源：

```bash
sudo curl -fsSLo /usr/share/keyrings/claude-desktop-archive-keyring.asc \
  https://downloads.claude.ai/claude-desktop/key.asc
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/claude-desktop-archive-keyring.asc] https://downloads.claude.ai/claude-desktop/apt/stable stable main" \
  | sudo tee /etc/apt/sources.list.d/claude-desktop.list
sudo apt update
sudo apt install -y claude-desktop
```

桌面文件是 `com.anthropic.Claude.desktop`，命令是 `claude-desktop`。登录在图形界面里完成，这个包装不了 API key。

Cowork 需要 KVM。用户已加入 `kvm` 组，要重新登录后 `/dev/kvm` 才稳定。

## RustDesk

装的是官方 1.4.9 的 `rustdesk-1.4.9-x86_64.deb`。服务用户是 root，单元是 `rustdesk.service`，`WantedBy=multi-user.target`。

只写 `After=network.target` 时，开机时网卡起来了但路由还没好，服务会先空转。补了一份 drop-in：

`/etc/systemd/system/rustdesk.service.d/boot.conf`

```ini
[Unit]
After=network-online.target
Wants=network-online.target

[Service]
Restart=on-failure
RestartSec=3
```

另外在 `~/.config/autostart/rustdesk.desktop` 里让登录后再拉起窗口。

自定义 ID 服务器要写两份配置，否则桌面一打开会把服务配置盖回公共服务器：

- 服务：`/root/.config/rustdesk/RustDesk2.toml`
- 登录用户：`~/.config/rustdesk/RustDesk2.toml`

官方命令行是：

```bash
sudo systemctl stop rustdesk
sudo rustdesk --config 'host=服务器:端口,key=公钥,'
sudo systemctl start rustdesk
```

普通用户直接跑 `rustdesk --config` 会报需要管理权限。用户那份是在服务停掉之后写入 toml 的。`custom-rendezvous-server` 和 `rendezvous_server` 都要带端口。同学连接用的是服务那一侧的 ID。

## 两个检测页不是同一套规则

当天浏览器里开过两个页面。先按第一个改，再打开第二个，分数对不上。原因是名单不一样。

### fuck-claude.app

和 [LinXiaoTao/FuckClaude](https://github.com/LinXiaoTao/FuckClaude) 的公开规则接近：

- Claude Code 真正读的时区只有 `Asia/Shanghai` 和 `Asia/Urumqi`，而且要在自定义了 API 地址时才会把结果写进提示词。
- `Asia/Taipei` 明确不计分。
- 浏览器语言首选 `zh-TW` 不计分。`zh-TW` 后面跟着的那个单独 `zh` 被当成地区标签的尾巴，也不计分。
- 简体字体（宋体、微软雅黑、`Noto Sans CJK SC`）算高分。只有繁体字体名时分数压在命中线下面。

### ippure.com/claude.html

这是后来浏览器里实际开着的那一页。权重和名单都更严：

| 信号 | 权重 | 这一页怎么算 |
|---|---|---|
| 时区 | 30 | 上海、乌鲁木齐等是 1 分。香港、澳门、**台北**是 0.6 分 |
| 语言 | 24 | 首选 `zh`、`zh-CN`、带 `hans` 是 1 分。首选 `zh-TW` / `zh-HK` / 带 `hant` 是 0.5 分 |
| 字体 | 20 | 任一简体字体名至少 0.75。没有简体、但有繁体字体名是 0.5 |
| Intl 区域 | 10 | 只要 locale 以 `zh` 开头就是 0.5，包括 `zh-Hant-TW` |
| UTC+8 | 8 | 偏移是 -480 分钟记 0.7。新加坡、台北都是东八区，这一项消不掉 |
| 表情风格 | 8 | 按 UA 猜系统。Linux 记 0.5 |

所以：

- 只改成台北，这一页时区仍有 `0.6 × 30 = 18` 分。
- 语言写成 `zh-TW, zh, en` 时，单独的 `zh` 还在列表里。这一页把单独的 `zh` 当成简体。首选如果是 `zh-TW`，语言项是 0.5，不是 0。
- 把语言改成只有 `zh-TW, en-US, en` 之后，Firefox 报出来的列表是 `zh-tw, en-us, en`，`hasBareZh` 为假。

页面自己的说明写的是：不方便改别的时区时，用 `Asia/Singapore`。钟还是东八区，时区名字不在它的台北/香港/澳门名单里。

用这套公式、英文语言、无中文字体名、时区新加坡，在这台机器的 Firefox 里测过一次，总分 9.6（UTC+8 的 5.6，加上 Linux 表情的 4）。改回 `zh-TW` 且不带单独 `zh` 之后，语言大约再加 12，Intl 的 `zh-Hant-TW` 大约再加 5。字体和时区名字仍是 0。

## 字体：fontconfig 拒绝名单挡不住 Firefox

snap 版 Firefox 能看见 `/usr/share/fonts` 里的字体文件。只在 `/etc/fonts/conf.d` 里 `rejectfont` 掉 `Noto Sans CJK SC`，`fc-match` 已经落到别的字体，Firefox 的 canvas 宽度测试仍然报宋体、微软雅黑、楷体、`Noto Sans CJK SC`。

原因是 `fonts-noto-cjk` 的那个 TTC 里同时登记了 SC / TC / JP / KR 这些家族名。别名在 `30-cjk-aliases.conf`：

- `SimSun`、`NSimSun` 接受 `Noto Serif CJK SC`，然后是 `AR PL UMing CN`
- `KaiTi` 接受 `Noto Serif CJK SC`，然后是 `AR PL UKai CN`
- `Microsoft YaHei` 接受 `Noto Sans CJK SC`

卸掉 `fonts-noto-cjk` 之后，这三个名字改去配文鼎字体，Firefox 仍判为命中。再卸掉 `fonts-arphic-ukai` 和 `fonts-arphic-uming`，canvas 对简体名单和繁体名单都是空的。假字体名 `DefinitelyNotAFontXYZ` 也是未命中，说明不是“随便一个名字都会变宽”。

中文还在，走剩下的 `Droid Sans Fallback`。它不在这两页的点名名单里。代价是没有单独的繁体字形文件，字形比较粗。

不要把 `Noto Sans CJK TC` 单独装回去却留着整包 CJK。整包会把 `Noto Sans CJK SC` 一起带出来。IPPure 上只要还探测到繁体字体名，字体项就是 0.5。

## 文件夹名

locale 切到 `zh_TW` 之后，`xdg-user-dirs` 把家目录改成了「桌面、文件、下載、圖片、音樂、影片、公共、模本」。旁边还留着更早生成的英文 `Videos`。

改回英文目录，并在 `~/.config/user-dirs.conf` 里写 `enabled=False`，避免下次登录按繁体 locale 再改一次名字。侧边栏书签 `~/.config/gtk-3.0/bookmarks` 也改成英文路径。

正在用的桌面会话是用旧目录启动的。注销一次，文件管理器才会稳定显示 `Desktop`、`Documents`、`Downloads`。

## Firefox 改的是当时真正在用的配置

snap 里先有一个 `*.default`，后来实际打开页面的是另一个 `*.default-*`。只改前一个的 `user.js`，正在用的窗口不会变。

最后写进实际配置的是：

```
user_pref("intl.accept_languages", "zh-TW, en-US, en");
user_pref("intl.locale.requested", "zh-TW");
user_pref("media.peerconnection.enabled", false);
```

`media.peerconnection.enabled` 关掉是为了 IPPure 那一页的 WebRTC 检查。Firefox 已经开着时，`user.js` 要完全退出再开才生效。

## 当天版本

- Clash Verge Rev 2.5.6
- Claude Code 2.1.274
- Claude Desktop 2.7032.0
- RustDesk 1.4.9
- Firefox 154（snap）

## VPS

代理这边的 VPS、日本 NAT 加台湾住宅节点的分流、订阅维护，单独写在 [VPS.md](VPS.md)。
