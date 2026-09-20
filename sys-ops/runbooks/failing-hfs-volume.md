# HFS+ 外接卷只读 / catalog 损坏

## 症状
- Finder 里权限显示 everyone 只读，解锁后仍提示「没有必要的权限」
- 偶发磁盘错误提示
- 卷能挂载但写不进去

## 判定（全是只读操作，一行取证）

```bash
V=/Volumes/external_; ls -d "$V" && diskutil info "$V" | grep -Ei 'File System|Owners|Read-Only|Mount Point|Device Node'; mount | grep -F "$V"; ls -led "$V"; echo "my uid: $(id -u)"; stat -f 'root owner: %u %g %Sp' "$V"; diskutil verifyVolume "$V"
```

关键是这两行的组合：

```
Media Read-Only:   No          ← 盘本身能写
Volume Read-Only:  Yes (read-only mount flag set)   ← 是 macOS 主动降级
```

以及 `fsck_hfs` 的输出：
```
Checking catalog file
Keys out of order
Invalid sibling link
File system check exit code is 8
```

**结论**：这是 catalog B-tree 的结构损坏（节点排序被破坏 + 叶节点双向链表断裂），不是权限问题。everyone 只读、Finder 改不动，全是只读挂载的下游症状。**不要去改 everyone，不要 `diskutil disableOwnership`。**

## 处置顺序（顺序不能反）

| 步骤 | 命令 | 副作用 | 失败时 |
|---|---|---|---|
| 1 先备份 | `sudo rsync -aHAXN --info=progress2 /Volumes/external_/ /Volumes/目标/backup/` | 只读源，安全 | 报 I/O error 的路径记下来，那就是坏区位置，别重试 |
| 2 读 SMART | 见下节 | 只读 | macOS 上大概率读不到 |
| 3 修复 | `sudo diskutil unmountDisk force /dev/disk4 && sudo fsck_hfs -fy /dev/rdisk4s2` | **原地改写元数据** | 见「失败模式」 |
| 4 复验 | 重复 3，直到输出 `appears to be OK` | | HFS+ 的 fsck 常要跑 2–3 轮才收敛 |
| 5 重新挂载 | `sudo diskutil mountDisk /dev/disk4 && mount \| grep external_` | | 确认 `read-only` 消失 |

`rdisk` 是 raw device，比 `disk` 快一个数量级。`-N` 保创建时间、`-H` 保 hard link、`-A -X` 保 ACL 与 xattr。

## 失败模式（决定要不要赌）

`Keys out of order` + `Invalid sibling link` 是 `fsck_hfs` 的经典失败区——它的设计是**修补**已有 B-tree，不是重建。主要风险不是把盘弄坏，而是**把链不回去的节点全扔进 `lost+found`**：文件内容还在，文件名和目录结构没了，几万个 `File_00001`。对代码工程约等于全损。估计概率 30–40%。

硬门槛：**SMART 必须干净才允许跑 fsck**。介质在坏时，fsck 的写入可能让盘彻底掉线，连只读挂载都没了。

修不好时：DiskWarrior（约 $120，做法是从 catalog 数据**重建**一棵新 B-tree 再替换，正对这个错误类型）→ 仍不行则抹盘重建。

## SMART：macOS 上基本读不到

```bash
sudo smartctl -a -d sat /dev/disk4
# → /dev/disk4: Type 'sat+...': Not a device of type 'scsi'
sudo smartctl --scan-open
# → 只列出内置 NVMe
```
**原因是 macOS 自身**：smartctl 的 Darwin 后端只能经 IOKit 访问 ATA 与 NVMe，USB 桥接盘需要 SCSI pass-through（`SG_IO`），macOS 不向用户态开放。`-d usbjmicron` / `-d usbasm1352r` 这些参数在 Linux/Windows 有效，在 macOS 一律撞同一句错——**不是参数选错，是通道不存在**。置信度 85%。

**现实路径：接到 Windows 跑 CrystalDiskInfo**（成功率估 85%）。唯一不可逆风险点：Windows 弹「需要格式化此磁盘」时**每次都点取消**——读 SMART 走 ATA 命令集，与文件系统无关，认不出 HFS+ 不影响读数。

要看的四项：`05 Reallocated_Sector_Ct`、`197 Current_Pending_Sector`、`198 Offline_Uncorrectable`、`199 UDMA_CRC_Error_Count`，外加通电时间/次数。
**199 单独偏高 = 线缆或硬盘盒的问题，不是盘体**，换线可能直接解决——而供电不足导致的传输中掉盘，正是 catalog 写到一半损坏的标准剧本。

## 抹盘：要写零，不要快速抹除

```bash
sudo diskutil unmountDisk /dev/disk4
sudo diskutil secureErase 0 /dev/disk4        # 2TB HDD 约 4–6 小时，全程别拔线
# 桥接不支持时的等价替代：
sudo dd if=/dev/zero of=/dev/rdisk4 bs=4m status=progress   # 末尾报 No space left 是正常结束
```
写零同时是**修复**和**诊断**：快速抹除只改分区表，pending sector 原样潜伏；写零强制固件对每个扇区做一次写入尝试，写得进去的自动重映射，写不进去的暴露出来。4–6 小时无 I/O error 跑完，是介质健康的强证据。

抹完再读一次 SMART 对比：5 涨一点、197 归零且不再增长 → 可继续用；197 仍非零或 5 持续增长 → 盘在死，只能当中转盘。

## 已知坑

- Genius Bar 不做数据恢复、不受理第三方外设，这条路可以划掉（置信度 90%）
- 目标盘若也是机械盘，格式化选 **HFS+（`diskutil eraseDisk JHFS+ rescue /dev/diskN`）而不是 APFS**——APFS 在 HDD 上随机 IO 表现差
- 盘对盘直传不需要 Mac 内置空间，「本机空间不足」是伪困境
- 坏道场景不要用 rsync/Finder（遇坏扇区会疯狂重试，磨损磁头），用 `ddrescue`（先跳过、后回补）
