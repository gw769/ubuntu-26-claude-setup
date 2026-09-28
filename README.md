# Ubuntu 26 这台虚拟机要做的事

机器是 VMware 里的 Ubuntu 26.04.1 桌面，GNOME，Wayland。网卡是 ens33，系统盘是 /dev/sda2。VMware 软件本身不用改，下面全是虚拟机里面要做的。

## 开 SSH

桌面版默认没有 SSH，外面连不进去。先在虚拟机自己的终端里装 OpenSSH，并让 `ssh.socket` 开机就听 22 端口。Ubuntu 26 用的是 socket 激活。防火墙开着就放行 OpenSSH。通了以后，其余操作从外面 SSH 进去做。

## 装 Clash Verge 和 Claude 桌面

Clash Verge 用上游 2.5.6 的 amd64 deb。这台系统已经有 webkit 4.1，不用再补老的 4.0 依赖。装完到桌面里打开，订阅自己填。

Claude 要装桌面版 `claude-desktop`，不要只装命令行。官方软件源在 `downloads.claude.ai`，签名密钥指纹对上再安装。当天桌面版是 2.7032.0，应用菜单里叫 Claude，在图形界面里登录。命令行包 `claude-code` 不是桌面。要用桌面里的 Cowork，把登录用户加进 `kvm` 组，然后重新登录。

## 时区和语言

时区用新加坡 `Asia/Singapore`。钟还是东八区。不要用台北：IPPure 的 Claude 检测页把台北和香港、澳门算在一起。

语言用繁体 `zh_TW`。浏览器语言写成 `zh-TW, en-US, en`。不要单独的 `zh`，也不要 `zh-CN`。单独的 `zh` 会被当成简体。

系统语言切到繁体后，家目录会被改成「桌面」「文件」「下載」。这些名字改回英文：Desktop、Documents、Downloads、Music、Pictures、Videos、Templates、Public。关掉登录时按语言重命名目录。改完注销一次，文件管理器才会显示英文。

## 卸字体

浏览器会按字体名字认宋体、微软雅黑、楷体、`Noto Sans CJK SC`。只改 fontconfig 不够，snap 版 Firefox 直接读字体文件。

卸掉这两个来源：

- `fonts-noto-cjk`。简体、繁体、日文、韩文写在同一个文件里，留着就会报出简体名字。
- `fonts-arphic-ukai` 和 `fonts-arphic-uming`。Noto 卸掉之后，宋体和楷体还会配到文鼎这些字，浏览器照样认。

卸完之后中文走 `Droid Sans Fallback`，字形粗一点。不要把整包 Noto CJK 装回去。

Firefox 如果已经开着，要完全退出再开，语言和字体才会变。改的是当时真正在用的那个配置目录，不是旁边另一个没用的 profile。

## 不休眠、不锁屏

空闲不关屏幕，不锁屏。睡眠和休眠关掉，机器一直醒着。

代理、日本 NAT 和台湾住宅节点写在 [VPS.md](VPS.md)。
