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
├── usr/bin/fwupdate              # 路由器上的一键升级脚本
└── etc/uci-defaults/99-fwupdate  # 首次启动初始化 fwupdate 的 uci 配置
```

## 工作流

- **每天 02:00（北京时间）** 轮询三处上游版本，拼成指纹
  `r25.12.2-lucky2.27.2-bandix0.12.11-luci0.12.12`；
  若该 tag 的 Release 已存在则**直接跳过**，不浪费 Actions 时间。
- 手动触发：Actions → Firmware Build → Run workflow
  - `force`：忽略已有 Release 强制重建
  - `extra_packages`：临时覆盖额外包清单
- GitHub 无法订阅第三方仓库的更新事件，因此只能轮询；见下方「改版本来源」。

产物 Release 内含：

| 文件 | 用途 |
|---|---|
| `r66s-<tag>.img.gz` | sysupgrade 镜像（保留配置） |
| `r66s-<tag>.img.gz.sha256` | 校验文件，`fwupdate` 自动核对 |
| `luci-app-bandix-*.apk` 等 6 个 | 单独重装/救急用 |
| `*.manifest` | 镜像包清单 |

## 路由器上升级

镜像内自带 `/usr/bin/fwupdate`：

```sh
fwupdate        # 检查 + 下载 + 校验 + 确认后刷写
fwupdate -c     # 只检查下载，不刷
fwupdate -y     # 不确认
fwupdate -f     # 版本相同也重刷
fwupdate -l     # 列出最近版本
```

可选：配置镜像加速 / 改仓库

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

## 注意

- 刷上自编固件后不要再装 `luci-app-attendedsysupgrade`：ASU 会把 lucky/bandix 这些
  本地来源的包一起上报，服务器报 `Impossible package selection: missing (...)`。
  本仓库的构建路径完全绕开了 ASU。
- 首次从官方 25.12.x 刷到本镜像时，`fwupdate` 还不存在，需要手动下载 Release 里的
  `.img.gz` 后 `sysupgrade -v`；之后就可以用 `fwupdate` 一键了。
