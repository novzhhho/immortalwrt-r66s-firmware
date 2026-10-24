# immortalwrt-r66s-firmware

给 **Lunzn FastRhino R66S**（rockchip/armv8）自动构建 ImmortalWrt 固件，并把两个非官方插件
**lucky**、**bandix** 直接烤进镜像 —— 从此升级不再需要「先卸载插件 → 升级 → 再装回去」。

## 为什么不用全量源码编译

lucky 和 bandix 的 OpenWrt 打包 Makefile 里 `Build/Compile` 都是空的：它们不编译任何代码，
只负责下载上游已编好的二进制再打包。所以：

| | 全量 buildroot | 本仓库做法（SDK + ImageBuilder） |
|---|---|---|
| 耗时 | 2–4 小时 | **15–25 分钟** |
| 磁盘 | 20–30 GB（runner 不够，要先清理） | 约 500 MB |
| 上游有坏提交时 | 整轮失败 | 不受影响（只用官方发布产物） |

流程：SDK 只编 lucky 的 3 个 apk → ImageBuilder 拿官方镜像源 + 这 3 个 apk + bandix 上游 3 个
apk 组装成完整镜像。

## 目录结构

```
.github/workflows/build.yml       # 唯一的流水线
files/                            # 会被塞进镜像的文件（ImageBuilder 的 FILES=）
└── etc/uci-defaults/99-fwupdate  # 首次启动把仓库地址写进 uci
```

升级引擎 `/usr/bin/fwupdate` 和网页界面由独立仓库
[novzhhho/luci-app-fwupdate](https://github.com/novzhhho/luci-app-fwupdate) 提供，
构建时 clone 进 `package/` 一起编译（不再放在 `files/` 里，避免两份拷贝）。

## 工作流

- **每天 02:00（北京时间）** 轮询三处上游版本 + 升级插件的提交 sha，拼成指纹
  `r25.12.2-lucky2.27.2-bandix0.12.11-bandixweb0.12.12-fw924dec8`；
  若该 tag 的 Release 已存在则**直接跳过**，不浪费 Actions 时间。
  - 段名含义：`bandix` 是流量统计的守护进程，`bandixweb` 是它的 LuCI 面板
    （`luci-app-bandix`，独立仓库、独立版本号），`fw` 是升级插件自身的提交。
    早先这两段写成 `bandix` / `luci`，`luci0.12.12` 会被误读成 LuCI 本体的版本。
- 手动触发：Actions → Firmware Build → Run workflow
  - `force`：忽略已有 Release 强制重建
  - `extra_packages`：临时覆盖额外包清单
- GitHub 无法订阅第三方仓库的更新事件，因此只能轮询；见下方「改版本来源」。

产物 Release 内含：

| 文件 | 用途 |
|---|---|
| `r66s-<tag>.img.gz` | sysupgrade 镜像（保留配置） |
| `r66s-<tag>.img.gz.sha256` | 校验文件，`fwupdate` 自动核对 |
| `luci-app-fwupdate-*.apk` | 升级插件（可单独装到别的机器上） |
| `luci-app-bandix-*.apk` 等 | 单独重装/救急用 |
| `*.manifest` | 镜像包清单 |

## 路由器上升级

### 网页（推荐，不用 SSH）

**LuCI → 系统 → 固件升级** → 「检查更新」→「立即升级」。
页面会显示新版本号、大小、发布日期、更新说明，升级时实时显示下载进度和日志。

### 命令行

镜像内自带 `/usr/bin/fwupdate`：

```sh
fwupdate        # 检查 + 下载 + 校验 + 确认后刷写
fwupdate -c     # 只检查下载，不刷
fwupdate -y     # 不确认
fwupdate -f     # 版本相同也重刷
fwupdate -l     # 列出最近版本
```

可选：配置镜像加速 / 改仓库（网页里也能改）

```sh
uci set fwupdate.main.mirror='https://ghfast.top/'
uci set fwupdate.main.repo='<owner>/<repo>'
uci commit fwupdate
```

镜像里同时打了 `luci-app-commands`，可以在 LuCI 里配一条命令，点按钮就升级。

## ⚠️ 两个不能动的值

1. `ROOTFS_PARTSIZE: "4096"` —— 必须和现网 p2 分区大小一致。不一致时
   `platform_check_image` 会判定分区布局变化并整盘 `dd`，p1 上的配置备份会一起没。
2. `PROFILE: lunzn_fastrhino-r66s`、`ARCH_PKGS: aarch64_generic` —— 与设备实际架构对应，
   写错会在 `apk` 安装阶段报架构不匹配。

## 改版本来源

`env` 段可改：

```yaml
LUCKY_FEED: https://github.com/gdy666/luci-app-lucky.git;main   # 换 fork 改这里
BANDIX_BACKEND: timsaya/openwrt-bandix      # 后端（Rust/eBPF 主体）
BANDIX_FRONTEND: timsaya/luci-app-bandix    # 前端 + i18n
```

lucky 的版本号取自 feed 里 `lucky/Makefile` 的 `PKG_VERSION`，**不是** core 仓库的 Release tag ——
core 发版后 feed 的 Makefile 常常滞后几天，盯 Release 会导致「以为该编了，其实包还是旧的」。

## 验证记录（2026-10-23，tag `r25.12.2-lucky2.27.2-bandix0.12.11-luci0.12.12`）

- Actions 9/9 步骤全绿，用时 33 分钟；Release 共 15 个资产。
- `r66s-<tag>.img.gz` 39,948,918 字节，sha256 `528d3f0d…607080`，与 `.sha256` 资产一致。
- 镜像解包（squashfs @ `0x4000000`，xz，bytes_used 27.22 MiB）核对：
  - `/etc/fw-version` = tag，`/etc/fwupdate-repo` = `novzhhho/immortalwrt-r66s-firmware`
  - `/usr/bin/fwupdate` 5492 字节、`-rwxr-xr-x`、纯 LF，与仓库内文件 sha256 **完全一致**
  - `/etc/uci-defaults/99-fwupdate` 649 字节
- manifest 237 个包：lucky 2.27.2-r3 / luci-app-lucky 3.0.0-r28 / bandix 0.12.11-r1 /
  luci-app-bandix 0.12.12-r1 / luci-app-openclash / argon / kmod-r8125 全在；
  `owut`、`luci-app-attendedsysupgrade` 按设计排除。
- 真机（R66S，`10.10.10.1`）跑 `fwupdate -c`：下载 → sha256 校验通过 → gzip 魔数校验通过
  → 退出码 0；全程只写 `/tmp`，`/etc` 未改动，测试残留已清理。

复核脚本：

```sh
python .workbuddy/tools/verify_release.py --dl     # 资产清单 + sha256
python .workbuddy/tools/inspect_image.py           # 解包镜像，逐项核对注入文件
```

## 注意

- 刷上自编固件后不要再装 `luci-app-attendedsysupgrade`：ASU 会把 lucky/bandix 这些
  本地来源的包一起上报，服务器报 `Impossible package selection: missing (...)`。
  本仓库的构建路径完全绕开了 ASU。
- 首次从官方 25.12.x 刷到本镜像时，`fwupdate` 还不存在，需要手动下载 Release 里的
  `.img.gz` 后 `sysupgrade -v`；之后就可以用 `fwupdate` 一键了。
- **上游没更新时手动强制重建，tag 不会变**（tag 只由三方版本指纹组成），Release 会被
  原地覆盖（`gh release upload --clobber`）。此时路由器上 `/etc/fw-version` 与新版 tag
  相同，`fwupdate` 会判定「已是最新」，需要用 `fwupdate -f` 强制重刷。
  正常的上游更新（immortalwrt / lucky / bandix 任一变化）会产生新 tag，没这个问题。
- `fwupdate` 依赖 `curl`、`jsonfilter`、`sha256sum`、`hexdump`。注意**不要**改成依赖 `od`：
  ImmortalWrt(apk 版) 默认不带 `od`（`od: not found`），只有 `hexdump`。
