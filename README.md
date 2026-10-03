# Ubuntu 26 这台虚拟机要做的事

机器是 VMware 里的 Ubuntu 26.04.1 桌面，GNOME，Wayland。网卡是 ens33，系统盘是 /dev/sda2。VMware 那边只需要把网络设成桥接，其余全是虚拟机里面要做的。

不写登录密码、出口 IP。

## VMware 设成桥接网络

桥接后虚拟机和主机在同一个局域网，拿到同网段的 IP，别的机器能直接 SSH 进来。以 Windows 上的 VMware Workstation 为例：

1. 在虚拟机上右键，选「设置」，再选「网络适配器」。
2. 选「桥接模式」。主机用 Wi-Fi 或者是笔记本的话，把「复制物理网络连接状态」也勾上。点确定。
3. VMware 菜单里选「编辑」，打开「虚拟网络编辑器」，点右下角「更改设置」（要管理员权限）。
4. 选中 `VMnet0`，把「桥接到」从「自动」改成主机实际上网用的那张网卡（Wi-Fi 或有线）。不要选虚拟网卡或 VPN 网卡。点确定。
5. 回到 Ubuntu，重连网络并查看 IP：

```bash
sudo nmcli networking off && sudo nmcli networking on
ip -4 addr show ens33
```

看到和主机同网段的地址（比如 `192.168.x.x`）就对了，后面 SSH 就连这个地址。拿不到 IP，多半是第 4 步选错了网卡，换一张再试。

## 开 SSH

桌面版默认没有 SSH，外面连不进去。先在虚拟机自己的终端里装 OpenSSH，并让 `ssh.socket` 开机就听 22 端口。Ubuntu 26 用的是 socket 激活。

```bash
sudo apt update
sudo apt install -y openssh-server
sudo systemctl enable --now ssh.socket
```

防火墙开着就放行 OpenSSH：

```bash
sudo ufw allow OpenSSH
```

通了以后，其余操作从外面 SSH 进去做。只有 `gsettings` 那几条要在桌面会话里的终端跑（见最后一节）。

## 装 Clash Verge 和 Claude 桌面

Clash Verge 用上游 [clash-verge-rev](https://github.com/clash-verge-rev/clash-verge-rev) 2.5.6 的 amd64 deb。这台系统已经有 webkit 4.1，不用再补老的 4.0 依赖。

```bash
wget https://github.com/clash-verge-rev/clash-verge-rev/releases/download/v2.5.6/Clash.Verge_2.5.6_amd64.deb
sudo apt install -y ./Clash.Verge_2.5.6_amd64.deb
```

装完到桌面里打开，订阅自己填。

Claude 要装桌面版 `claude-desktop`，不要只装命令行。官方软件源在 `downloads.claude.ai`。先下密钥，指纹对上再加源：

```bash
curl -fsSLo /tmp/claude-desktop.asc https://downloads.claude.ai/claude-desktop/key.asc
gpg --show-keys /tmp/claude-desktop.asc
# 指纹应为 31DDDE24DDFAB679F42D7BD2BAA929FF1A7ECACE，对不上就停
sudo install -m 644 /tmp/claude-desktop.asc /usr/share/keyrings/claude-desktop-archive-keyring.asc
echo "deb [arch=amd64,arm64 signed-by=/usr/share/keyrings/claude-desktop-archive-keyring.asc] https://downloads.claude.ai/claude-desktop/apt/stable stable main" \
  | sudo tee /etc/apt/sources.list.d/claude-desktop.list
sudo apt update
sudo apt install -y claude-desktop
```

应用菜单里叫 Claude，命令是 `claude-desktop`，在图形界面里登录。命令行包 `claude-code`（源是 `https://downloads.claude.ai/claude-code/apt/stable`）是终端里的 `claude`，不是桌面。

要用桌面里的 Cowork，把登录用户加进 `kvm` 组，然后注销重新登录：

```bash
sudo usermod -aG kvm $USER
```

## 时区和语言

时区用台北 `Asia/Taipei`。钟是东八区。
如果这台要看起来像日本，时区应对 `Asia/Tokyo`（东九区），记在 [VPS.md](VPS.md) 的「焚决检查」。这次先不改。

```bash
sudo timedatectl set-timezone Asia/Taipei
timedatectl
```

语言用繁体 `zh_TW`：

```bash
sudo apt install -y language-pack-zh-hant language-pack-gnome-zh-hant
sudo locale-gen zh_TW.UTF-8
sudo localectl set-locale LANG=zh_TW.UTF-8
```

GNOME 登录用户还有自己的一份语言设置，在「设置 → 系统 → 区域与语言」里也选成繁体中文（台湾），然后注销。

浏览器语言写成 `zh-TW, en-US, en`。不要单独的 `zh`，也不要 `zh-CN`。单独的 `zh` 会被当成简体。具体写在 Firefox 那一段。

### 家目录改回英文

系统语言切到繁体后，家目录会被改成「桌面」「文件」「下載」「圖片」「音樂」「影片」「公共」「模本」。改回英文。已经有同名英文目录（比如早先留下的 `Videos`）就把内容并进去：

```bash
cd ~
for p in 桌面:Desktop 文件:Documents 下載:Downloads 圖片:Pictures 音樂:Music 影片:Videos 公共:Public 模本:Templates; do
  zh=${p%%:*}; en=${p##*:}
  [ -d "$zh" ] || continue
  if [ -d "$en" ]; then mv -n "$zh"/* "$en"/ 2>/dev/null; rmdir "$zh"; else mv "$zh" "$en"; fi
done
LC_ALL=C xdg-user-dirs-update --force
```

关掉登录时按语言重命名目录：

```bash
mkdir -p ~/.config
echo 'enabled=False' > ~/.config/user-dirs.conf
```

文件管理器侧边栏的书签也改成英文路径：

```bash
f=~/.config/gtk-3.0/bookmarks
[ -f "$f" ] && sed -i -e 's#/桌面#/Desktop#' -e 's#/文件#/Documents#' -e 's#/下載#/Downloads#' \
  -e 's#/圖片#/Pictures#' -e 's#/音樂#/Music#' -e 's#/影片#/Videos#' -e 's#/公共#/Public#' -e 's#/模本#/Templates#' "$f"
```

书签里的中文如果是 URL 编码的（`%E4%B8%8B…`），用文本编辑器直接改。改完注销一次，文件管理器才会稳定显示英文。

## 卸字体

浏览器会按字体名字认宋体、微软雅黑、楷体、`Noto Sans CJK SC`。只改 fontconfig 不够，snap 版 Firefox 直接读字体文件。

卸掉这两个来源：

- `fonts-noto-cjk`。简体、繁体、日文、韩文写在同一个文件里，留着就会报出简体名字。
- `fonts-arphic-ukai` 和 `fonts-arphic-uming`。Noto 卸掉之后，宋体和楷体还会配到文鼎这些字，浏览器照样认。

```bash
sudo apt purge -y fonts-noto-cjk fonts-arphic-ukai fonts-arphic-uming
sudo fc-cache -f
fc-list :lang=zh family      # 应该只剩 Droid Sans Fallback 这类
```

卸完之后中文走 `Droid Sans Fallback`，字形粗一点。不要把整包 Noto CJK 装回去。

### Firefox 配置

Firefox 是 snap 版，配置目录在：

```
~/snap/firefox/common/.mozilla/firefox/
```

里面可能有好几个 profile（比如 `xxxx.default` 和 `yyyy.default-zzzz`）。要改的是当时真正在用的那个，不是旁边另一个没用的。两种找法：

- Firefox 地址栏打开 `about:profiles`，标着“正在使用”的那个，看它的“根目录”。
- 看 `profiles.ini` 里 `[Install…]` 段的 `Default=`。

```bash
cd ~/snap/firefox/common/.mozilla/firefox/
grep -A3 '^\[Install' profiles.ini
```

在那个目录里写 `user.js`：

```js
user_pref("intl.accept_languages", "zh-TW, en-US, en");
user_pref("intl.locale.requested", "zh-TW");
user_pref("media.peerconnection.enabled", false);
```

`media.peerconnection.enabled` 关掉是为了不让 WebRTC 报出本机地址。Firefox 如果已经开着，要完全退出再开，语言和字体才会变。

## 繁体输入法

原来只有键盘布局 `xkb cn`，打不出繁体。装注音和新酷音用的拼音，并把拼音默认改成出繁体：

```bash
sudo apt install -y ibus-chewing ibus-libpinyin
gsettings set com.github.libpinyin.ibus-libpinyin.libpinyin init-simplified-chinese false
gsettings set org.gnome.desktop.input-sources show-all-sources true
gsettings set org.gnome.desktop.input-sources sources "[('xkb', 'us'), ('ibus', 'libpinyin'), ('ibus', 'chewing')]"
```

右上角用 Super+空格 轮换：英语、智能拼音（出繁体）、新酷音（注音）。右上角没出现就注销一次。

## 远程桌面用 RDP

Ubuntu 26 的 GNOME 50 自带远程桌面只有 RDP，没有 VNC。开的是当前已登录桌面，端口 3389。人要留在桌面里，注销后这条连接会断。

在桌面会话里的终端跑。用户名用登录用户，密码用登录密码：

```bash
sudo apt install -y freerdp3-x11
install -d ~/.local/share/gnome-remote-desktop
openssl req -new -newkey rsa:2048 -days 3650 -nodes -x509 \
  -subj "/CN=$USER" \
  -keyout ~/.local/share/gnome-remote-desktop/rdp-tls.key \
  -out ~/.local/share/gnome-remote-desktop/rdp-tls.crt
grdctl rdp set-tls-cert ~/.local/share/gnome-remote-desktop/rdp-tls.crt
grdctl rdp set-tls-key ~/.local/share/gnome-remote-desktop/rdp-tls.key
grdctl rdp set-credentials "$USER" '登录密码'
grdctl rdp disable-view-only
grdctl rdp enable
systemctl --user enable --now gnome-remote-desktop.service
```

Windows 上打开「远程桌面连接」，地址填这台机器的局域网 IP。第一次会提示证书不受信任，选连接。

## 不休眠、不锁屏

空闲不关屏幕，不锁屏。这几条是当前登录用户的设置，要在桌面会话里的终端跑：

```bash
gsettings set org.gnome.desktop.session idle-delay 0
gsettings set org.gnome.desktop.screensaver lock-enabled false
gsettings set org.gnome.desktop.lockdown disable-lock-screen true
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-ac-type 'nothing'
gsettings set org.gnome.settings-daemon.plugins.power sleep-inactive-battery-type 'nothing'
```

睡眠和休眠从系统层面关掉，机器一直醒着：

```bash
sudo systemctl mask sleep.target suspend.target hibernate.target hybrid-sleep.target
```

## 当天版本

- Clash Verge Rev 2.5.6
- Claude Desktop 2.7032.0
- Claude Code 2.1.274
- Firefox（snap）

代理、日本 NAT 和台湾住宅节点写在 [VPS.md](VPS.md)。
