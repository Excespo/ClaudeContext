# inbox

**2026-09-13：6 条历史 chat 已全部读完，全部蒸馏进 runbooks/ 与 incidents/，见 _triage.md。**

剩下的是**照片库抢救这件事本身的未决项**（见 incidents/2026-09-13-external-hfs-catalog.md）：

- [ ] Pan 网页端已用空间是否 ≈ 325 G
- [ ] Pan 客户端是否有「挂载 / 按需下载」模式（有则可零流量对云端做 find 计数）
- [ ] 下载 `database/`（三个文件齐全）做 `PRAGMA integrity_check` + `ZASSET` 计数交叉验证
- [ ] 接 Windows 跑 CrystalDiskInfo 拿 SMART 四项 + 通电时间/次数
- [ ] 看网盘客户端传输列表的失败条目数（零成本，可能直接终结流程）

**硬规则：以上未完成前，源盘一个字节都不动。**

其他：
- [ ] 换一根线/硬盘盒再挂载一次——若 199 UDMA_CRC 偏高，这一步可能直接解决问题
