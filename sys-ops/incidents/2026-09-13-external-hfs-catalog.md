# 2026-09 外接盘 catalog 损坏与照片库抢救

## 时序与事实（全部来自实际命令输出）

- 卷：`/dev/disk4s2` → `/Volumes/external_`，HFS+，`Owners: Enabled`，`Media Read-Only: No` / `Volume Read-Only: Yes`
- `diskutil verifyVolume` → `Keys out of order` / `Invalid sibling link` / **exit code 8**；随后 `Problem -69842 ... Error: -69845`
- 卷根 `drwxrwxr-x 18 root wheel`，本人 uid 501 → Finder 改不动权限，是只读挂载的下游症状
- SMART：`smartctl --scan-open` 只列出内置 NVMe，外置盘未被枚举；`diskutil info` 报 `SMART Status: Not Supported`（macOS 对 USB 桥接的一律回答，不含健康信息）

## 照片库取证

`LIB=/Volumes/external_/Images/photoslib.photoslibrary`

| 子目录 | 大小 |
|---|---|
| originals | **274 G**（不可再生） |
| resources | 50 G（可重建） |
| database | 691 M |
| private | 312 M |
| internal | 60 K / scopes 4 K / external 0 B |

- 文件总数 **46,948**
- 零字节文件 **76**，全部在 `database/search/Spotlight/` 与 `database/protection` 下 → Spotlight 索引的 journal/toc/锁文件，Photos 每次打开都会重建，**零风险**
- symlink **0**（担心的一类问题不存在）
- hard link **28**（网盘丢失后最多多占几百 MB，数据不丢）
- 隐藏文件 **1**（「网盘跳过隐藏文件」这个高频失败模式影响面极小）

结论：验证范围从 325 G 缩小到 274 G + 691 M。

## 云端

SJTU **Pan 交大云盘**（pan.sjtu.edu.cn），1 TB，单文件上限 50 GB → 「超大文件被跳过」可排除。
**无官方 WebDAV**；第三方 TboxWebdav 走 cookie/token 桥接，对 4 万多文件的批量校验不建议（出错时无法区分是中间层还是数据的问题），rclone 路径作废。
注：jBox 已于 2025-04-30 停服并清空数据。

## 当前状态与未决项

- [ ] Pan 网页端已用空间是否 ≈ 325 G（差 40–50 G 说明缺 resources，无所谓；明显少于 274 G 是严重问题）
- [ ] Pan 客户端是否有「挂载/按需下载」模式——有的话可以直接对云端跑同样的 find 计数，零流量
- [ ] 下载 `database/` 做 `PRAGMA integrity_check` + `ZASSET` 计数交叉验证
- [ ] 接 Windows 跑 CrystalDiskInfo 拿 SMART
- **硬规则：以上未完成前，源盘一个字节都不动。**
