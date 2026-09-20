# 设备与存储清单

> 唯一事实源。每次接入新卷或换网盘就更新这里，不要散落在各个 incident 里。

## 主机
- dz-mba-local（macOS, Apple Silicon）
- slaanesh（Windows 游戏本双系统里的 Ubuntu；GNOME，仅 Wayland 会话；NVIDIA RTX 4070 Max-Q，nvidia 专有驱动；Ubuntu 用独立 ESP 分区；来源：notes/ubuntu-desktop-handoff.md）

## 卷
| 挂载点 | 设备 | 文件系统 | 用途 | 健康状态 | 备份去向 |
|---|---|---|---|---|---|
| /Volumes/external_ | /dev/disk4s2 | HFS+ | 归档（含 Photos 库） | catalog B-tree 损坏，只读挂载 | pan.sjtu.edu.cn |

## 云端
| 服务 | 用途 | 限制 |
|---|---|---|
| Pan 交大云盘（pan.sjtu.edu.cn） | 大体积归档 | 见 incidents/ 中的兼容性结论 |

## 网络
| 工具 | 用途 | 备注 |
|---|---|---|
| Shadowrocket | 代理 | 流量统计与服务端对账口径见 inbox |

## 包管理与路径（从全局 CLAUDE.md 迁入，2026-09-13）

| 项 | 事实 | 后果 |
|---|---|---|
| Homebrew 前缀 | `/opt/homebrew` | Apple Silicon 默认位置 |
| `/usr/local/Cellar` | **aTrust（VPN 客户端）的数据，不是 Homebrew** | 清理 `/usr/local` 时必须跳过，误删会破坏 VPN 客户端 |
| Node | 纯 Homebrew，刻意不用 nvm/fnm 等版本管理器 | 全局 CLI 用 `brew install`；不用 `npm -g`，不全局升 npm |
| Python | pyenv + uv | 不用系统 python，不用 conda |
| sudo | Bash 工具无 tty，`sudo` 必然失败 | 见 runbooks/sudo-without-tty.md |

## 已知坑

- 外置盘 `external_` 的 HFS+ catalog B-tree 损坏（`fsck_hfs` exit 8，`Keys out of order` / `Invalid sibling link`），只读挂载使用，不要跑写入型修复。
