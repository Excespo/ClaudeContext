# .photoslibrary 备份与「云端那份到底能不能用」的验证

## 为什么这件事特别

`.photoslibrary` 是 **bundle**（Finder 显示成单文件，实际是目录），里面有 SQLite 库 + hard link + xattr。它的 bundle 身份来自**扩展名 + LaunchServices 声明**，不是文件系统标志位——所以下载回 HFS+/APFS 卷上，只要顶层目录名还叫 `xxx.photoslibrary`，Finder 和 Photos.app 就认。这一条可以从担心清单里划掉。真正会丢的是 hard link、symlink、xattr。

## 风险排序（网盘同步整个库）

1. **非原子同步导致 SQLite 损坏**（最主要）：客户端逐个文件上传，拿不到 `Photos.sqlite` + `-wal` + `-shm` 在同一时刻的一致快照。Photos 或 `photoanalysisd` 在后台跑时尤其危险
2. xattr 丢失（iCloud Drive 保留，OneDrive/Google Drive 基本不保留，国内网盘视为全丢）
3. 文件名：NFD/NFC 不一致、大小写冲突、Windows 非法字符、路径长度；部分客户端**静默跳过隐藏文件与空文件**
4. 按需下载占位符：本地只剩 placeholder，Photos 打开缺文件的库行为不可预料

**正确做法**：退出 Photos（让 WAL 写回）→ 打包成单文件再传
```bash
ditto -c -k --sequesterRsrc --keepParent Photos.photoslibrary lib.zip
# 或（能完整保留元数据）
hdiutil create -srcfolder Photos.photoslibrary -format UDZO lib.dmg
```
上传前后各算一次 `shasum -a 256`。Apple 官方也建议不要把图库放在任何云同步文件夹里（含 iCloud Drive）。

## 已经直传成了目录怎么办：验证方法

**关键判据不是「云端 vs 源盘文件数一致」。** 源盘的 catalog B-tree 已损坏，`find` 遍历到断链处后面的整个子树**直接不可见、也不报错**，网盘客户端当初读到的就是同一棵被截断的树。两边一致只能证明「上传没有额外丢东西」，证明不了「没丢东西」。

唯一独立于损坏目录树的清单是 `Photos.sqlite`。验证链：

**第 0 步（零成本，可能直接终结流程）**：看网盘客户端的传输列表「失败」条目数。三类静默跳过在此场景概率最高：`.` 开头的隐藏文件、超过单文件上限的长视频、**从坏扇区读取时报 I/O 错误的文件**。

**第 1 步：确认云端有 `originals/`**。子目录里只有它不可再生：

| 子目录 | 内容 | 丢了会怎样 |
|---|---|---|
| `originals/` | 照片视频原始文件 | **永久丢失** |
| `database/` | SQLite，相册/People/编辑记录 | 不可再生，但体积小 |
| `resources/` | 缩略图与衍生图 | 可完全重建，**不值得为它花验证成本** |
| `scopes/` `private/` `internal/` `external/` | 内部状态 | 基本可再生 |

**第 2 步：源盘取证**（只读挂载不影响）
```bash
LIB=/Volumes/external_/Images/photoslib.photoslibrary
du -sh "$LIB"/*; find "$LIB" -type f | wc -l; find "$LIB" -type l
find "$LIB" -type f -links +1 | wc -l; find "$LIB" -name '.*' -type f | wc -l
find "$LIB" -type f -size 0 | tee ~/zero_src.txt | wc -l
grep -c '/originals/' ~/zero_src.txt      # 必须是 0
```

**第 3 步：只下载 `database/`（几百 MB），做交叉验证**。三个文件必须齐全（缺 `-wal` 会得到「能打开但是旧快照」的库）：
```bash
sqlite3 Photos.sqlite "PRAGMA integrity_check;"    # 期望恰好一行 ok
sqlite3 Photos.sqlite "PRAGMA foreign_key_check;"  # 期望空
sqlite3 Photos.sqlite "SELECT COUNT(*) FROM ZASSET;"  # 旧版是 ZGENERICASSET
```
拿 `ZASSET` 计数与 `originals/` 的实际文件数对比——**这才是 ground truth**。

一个意外的优势：卷是只读的，Photos 从故障起就无法写入，库处于静止状态，不存在「边写边传」导致的数据库自身不一致。

## 硬规则

**在云端那份被证明可用之前，不要格式化源盘。** 只读挂载是在保护数据（内核不再写入，损坏被冻结）；抹盘等于把双副本变成单副本。`diskutil secureErase` 是整个流程里唯一不可逆的动作。
