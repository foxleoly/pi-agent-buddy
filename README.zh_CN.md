<p align="right">
  <strong>简体中文</strong> · <a href="README.md">English</a>
</p>

# Pi Agent Buddy

Pi Agent Buddy 的**电脑侧**：给 [pi](https://pi.dev) 编码 Agent 用的随身状态牌与遥控器。
AI Passport 设备负责显示你的 Agent 正在做什么、并让你用按键把它停下；本仓库是喂数据给它、
接住它按键的那一半。

| 部分 | 作用 |
| --- | --- |
| [`extensions/pibuddy.ts`](extensions/pibuddy.ts) | pi 扩展：在 pi 底部状态栏显示链路状态，`/pibuddy` 汇报状态并提供复制配对码。 |
| [`sidecar/`](sidecar/) | Go 程序：读取你的 pi 会话与 `herdr` 的 agent 快照，把心跳推给设备，并把设备按键转成 `herdr` 输入。 |

## 前置条件

- 一个正在运行的 **pi** 会话。
- **herdr**：sidecar 读取并驱动它，从 <https://herdr.dev> 安装（例如 `brew install herdr`）。
- 只有你自己编译 sidecar 时才需要 **Go 1.27+**（也可直接下载预编译二进制）。

## 安装 pi 扩展

```bash
pi install git:github.com/foxleoly/pi-agent-buddy@v0.1.0
```

之后 `/pibuddy` 会汇报链路状态，并提供把配对码复制到剪贴板。

## 运行 sidecar

从[最新 Release](https://github.com/foxleoly/pi-agent-buddy/releases/latest) 下载对应平台的
二进制（`pi-buddy-sidecar-<os>-<arch>`，附 `checksums.txt`）：

```bash
chmod +x pi-buddy-sidecar-darwin-arm64
./pi-buddy-sidecar-darwin-arm64 --ws --ws-addr :51820 --target <pane>
```

也可以自行编译：

```bash
cd sidecar
go build -o pi-buddy-sidecar .
./pi-buddy-sidecar --ws --ws-addr :51820 --target <pane>
```

`--target` 是必需的。不带它时，设备按键会打到当前**聚焦**的 herdr pane，而那可能正是跑
sidecar 的这个会话。

常用参数：

```bash
./pi-buddy-sidecar --show-token                   # 打印设备配网页要的链接码
./pi-buddy-sidecar --ws --ws-addr :51820 --target w4:p1
./pi-buddy-sidecar --usb --usb-path /dev/ttyACM0 --target w4:p1   # 走有线的 USB 通道
```

## 作为后台服务运行

两个服务定义都在 [`service/`](service/)。把二进制放到 `~/.local/bin/pi-buddy-sidecar`，并把
`@PANE@` 换成设备按键要控制的 herdr pane（这里用 `w4:p1`）。

### macOS（launchd）

launchd 既不展开 `~` 也不展开环境变量，所以先把 plist 渲染一遍：

```bash
mkdir -p ~/.local/bin ~/Library/LaunchAgents
mv pi-buddy-sidecar-darwin-arm64 ~/.local/bin/pi-buddy-sidecar
sed "s|@HOME@|$HOME|g; s|@PANE@|w4:p1|g" \
  service/dev.foxleoly.pi-buddy-sidecar.plist \
  > ~/Library/LaunchAgents/dev.foxleoly.pi-buddy-sidecar.plist
launchctl bootstrap "gui/$(id -u)" ~/Library/LaunchAgents/dev.foxleoly.pi-buddy-sidecar.plist
launchctl print "gui/$(id -u)/dev.foxleoly.pi-buddy-sidecar" | head -20
```

卸载：

```bash
launchctl bootout "gui/$(id -u)/dev.foxleoly.pi-buddy-sidecar"
rm ~/Library/LaunchAgents/dev.foxleoly.pi-buddy-sidecar.plist
```

### Linux（systemd 用户单元）

```bash
mkdir -p ~/.local/bin ~/.config/systemd/user
mv pi-buddy-sidecar-linux-amd64 ~/.local/bin/pi-buddy-sidecar
sed "s|@PANE@|w4:p1|g" service/pi-buddy-sidecar.service \
  > ~/.config/systemd/user/pi-buddy-sidecar.service
systemctl --user daemon-reload
systemctl --user enable --now pi-buddy-sidecar
systemctl --user status pi-buddy-sidecar
loginctl enable-linger "$USER"    # 没有登录会话时也保持运行
```

卸载：

```bash
systemctl --user disable --now pi-buddy-sidecar
rm ~/.config/systemd/user/pi-buddy-sidecar.service
```

两个单元都会在 sidecar 退出后重启它，并且都显式设了 `PATH` —— launchd 任务和 systemd 用户
单元默认的 `PATH` 很精简，会漏掉 Homebrew，而 sidecar 要调 `herdr`。日志方面 sidecar 自己会写
`~/.pi-buddy/ble.log`；launchd 那份另外把 stdout/stderr 收集到 `~/.pi-buddy/sidecar.out.log`
与 `~/.pi-buddy/sidecar.err.log`。

## 设备怎么找到 sidecar

sidecar 通过 mDNS 广播 `_pibuddy._tcp`，并监听 `--ws-addr` 指定的端口。设备在同一网络里
发现它，然后拨 `/<链接码>`——一个只生成一次、存放在 `~/.pi-buddy/token` 的 12 位十六进制码
（用 `--show-token` 可再次打印）。其他路径会在 WebSocket 升级前被拒绝，所以网内未认证的
对端碰不到控制逻辑。

sidecar 还会把链路状态发布到 `~/.pi-buddy/status.json`（`connected`、`device`、`pid`、
`updated`），让本机其他工具（比如上面的扩展）无需 IPC 就能显示链路状态。`pid` 用来区分
「已死的 sidecar 留下的陈旧文件」和「真实在线的链路」。

## 说明

- 纯 Go、无 cgo：macOS、Linux、Windows 均可构建运行。
- sidecar 只是电脑侧的一半，不会直接接触设备硬件；设备固件另行分发。
- 采用 MIT 许可证 —— 见 [LICENSE](LICENSE)。
