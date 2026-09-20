<!-- BEGIN notion-pointer -->
> **状态只在 Notion 改，本地不要建同名文件。**
> 规则真相源：[AI-Infra 工作台](https://app.notion.com/p/3e1854271cf781dfb804dd88945d6283)
> 掌握度 / 能力阶梯 / inbox / kata 四个 database 都在该页下。
> 本仓库只放笔记正文与 skill；`00-index.md`、`inbox.md`、`ROADMAP.md`、`STATUS.md` 已废弃，不要重建。
> 写完 note 用 `/note` 更新掌握度与复查日期。
<!-- END notion-pointer -->

# sys-ops（原 mac-ops）：设备、存储、网络与 Linux 桌面运维

## 目标
把一次性的排障过程沉淀成可重跑的 runbook，避免同一类故障重新推导一遍。
这个方向不是「学习 ladder」，是**可执行手册 + 事故档案**。

## 硬约束（数据安全优先于修复速度）
- 任何可能写入用户数据卷的命令，先给 dry-run 或只读版本，并说明失败后的回滚路径
- 对疑似故障盘：优先「只读挂载 + 整卷镜像」，不要先跑修复工具
- 删除/格式化类操作永远由我本人执行，Claude 只给命令与校验步骤
- 校验优先于信任：拷贝完成 = 文件数 + 字节数 + 抽样哈希三者一致，不是「Finder 没报错」

## 目录
- `runbooks/` 可重跑的操作手册，一个故障类型一篇（模板见 runbooks/TEMPLATE.md）
- `incidents/` 按 `YYYY-MM-DD-<slug>.md` 记录具体事故的时序与原始命令输出
- `inventory.md` 设备、卷、文件系统、网盘与代理的清单（唯一事实源）
- `notes/` 非故障类的配置手册与 handoff（如 Ubuntu 桌面的语言、输入法、字体），入口是 `notes/00-index.md`
- `inbox.md` 待整理成 runbook 的历史 chat

## 写作要求
命令要能直接粘；每条命令注明**只读还是有副作用**；引用输出时保留原始退出码与错误串（如 `fsck_hfs` exit 8、`Invalid sibling link`），不要改写。
