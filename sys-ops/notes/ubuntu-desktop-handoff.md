# Ubuntu 桌面新装 handoff：语言、目录、输入法、portal 弹窗、字体

> 来源：slaanesh（Windows 游戏本双系统里的 Ubuntu）上实际排查过的问题，2026-09-17 整理。
> 读者：刚从头装好 Ubuntu、命令行会用、但不熟悉 Linux 桌面机制的人。
> 每条命令都标了 **[只读]** 或 **[有副作用]**。有副作用的命令都附回滚方法和失败判据。**一次只执行一条，对照预期输出再继续。**

---

## 0. 先搞清楚你的环境（全部 [只读]）

```bash
echo $XDG_SESSION_TYPE          # 预期: wayland
echo $XDG_CURRENT_DESKTOP       # 预期: ubuntu:GNOME
locale | head -3                # 看 LANG=，是 zh_CN.UTF-8 还是 en_US.UTF-8
cat ~/.config/user-dirs.dirs    # 家目录下「下载/文档」这类标准目录的真实映射
echo "GTK=$GTK_IM_MODULE QT=$QT_IM_MODULE XMOD=$XMODIFIERS"   # 当前输入法框架
```

本手册基于的环境：
- Ubuntu 新版，GNOME 桌面，**只有 Wayland 会话**（新版 GNOME 已去掉 X11 会话），默认终端是 Ptyxis
- 硬件和使用方式：游戏本，NVIDIA RTX 4070 Max-Q，用 nvidia 专有驱动；平时合上笔记本盖子，只用外接屏幕和键盘；Ubuntu 用的是独立的 ESP 分区，不和 Windows 共用
- 浏览器是 Google Chrome（deb 包）；其他常用应用是 Cursor、Tabby、Claude 桌面版，都是 Electron 应用

几个术语：
- **ibus / fcitx5**：两套「输入法框架」，负责把按键交给输入引擎
- **引擎**：真正做拼音转汉字的组件，例如 `ibus-libpinyin`（智能拼音）、fcitx5 的 `pinyin`
- **portal**：`xdg-desktop-portal-gnome`，GNOME 替应用弹出「打开方式」「选择文件」这类系统对话框的服务

---

## 快速索引：症状对应章节

| 症状 | 章节 |
|---|---|
| 家目录里同时有 `下载` 和 `Downloads`；文件管理器侧栏显示中文目录 | §2 |
| 想把系统界面改成英文 | §1 |
| 用 apt 装了 ibus，但设置里找不到「智能拼音」 | §3 |
| fcitx 不能切换中英文 / 默认打出繁体 / 界面难看 | §4 |
| Chrome、Cursor 里打不了中文 | §5 |
| 程序栏莫名出现「portal」窗口：Open with / No apps available / `xxx://` | §6 |
| 查看或安装字体；改成英文后汉字显示成日文字形 | §7 |
| 合盖、代理、显卡 | 附录 A（**没有在本机验证过**） |

---

## 1. 把系统语言改成英文

**为什么要改**：界面语言决定了新建标准目录的名字、很多程序配置文件里注释的语言，以及报错信息的语言。报错是英文的话，搜索起来容易得多。

**[有副作用]** 操作：Settings → System → Region & Language → Language 改成 **English (United States)**。Formats（日期、货币格式）可以继续用 China。改完后注销，再重新登录。

- 会改变什么：当前用户的界面语言。这个设置保存在 AccountsService 里，优先级高于 `/etc/default/locale`，所以只用 `sudo update-locale` 可能不生效。
- 重新登录时如果弹出「Update standard folder names?」：
  - 目录已经是英文名：选 **Keep Old Names**
  - 目录还是中文名，想改成英文：选 **Update Names**。这一步会**重命名**目录，里面的文件不会丢，但其他程序里记住的旧路径会失效（见 §2）。
- 回滚：在同一个地方改回中文，再注销、登录。
- 验证 **[只读]**：重新登录后运行 `locale | head -1`，预期输出 `LANG=en_US.UTF-8`。

---

## 2. 家目录里残留中文目录（`下载`、`文档`……）

### 原因

这台机器上实际查到了三个互相独立的来源，要分别处理：

| 来源 | 文件 | 当时的状态 |
|---|---|---|
| XDG 标准目录 | `~/.config/user-dirs.dirs` | 已经是英文（`XDG_DOWNLOAD_DIR="$HOME/Downloads"`），**不是它的问题** |
| 文件管理器侧栏书签 | `~/.config/gtk-3.0/bookmarks` | 还指向 `文档/音乐/图片/视频/下载`，其中 4 个目录早已不存在 |
| Chrome 记住的旧路径 | `~/.config/google-chrome/Default/Preferences` | `savefile.default_directory = ~/下载`，Chrome「另存为」时又把 `~/下载` 建了出来 |

也就是说，把系统改成英文、或者用 `xdg-user-dirs-update` 改好标准目录，都**不会**同步修改书签和各个程序里记住的路径。

### 诊断（全部 [只读]）

```bash
cat ~/.config/user-dirs.dirs
cat ~/.config/gtk-3.0/bookmarks
ls -la ~/下载
python3 -c "import json,os;p=json.load(open(os.path.expanduser('~/.config/google-chrome/Default/Preferences')));print(p.get('savefile'),p.get('download'))"
```

书签里的中文会显示成 URL 编码：`%E4%B8%8B%E8%BD%BD` 就是「下载」。

### 处理

**2.1 如果 `user-dirs.dirs` 里还是中文，先改成英文 [有副作用]**

```bash
LC_ALL=C xdg-user-dirs-update --force && cat ~/.config/user-dirs.dirs
```
- 预期：配置里全部变成 `$HOME/Desktop`、`$HOME/Downloads` 这样的英文路径。
- 注意：这条命令会**新建**英文目录，但**不会**把中文目录里的文件搬过去，要自己用 `mv -n` 搬。
- 回滚：手动编辑 `~/.config/user-dirs.dirs`，改回原来的路径。

**2.2 把中文目录里的文件搬走，再删掉空目录**

先看清楚里面有什么 **[只读]**：

```bash
ls -la ~/下载
```

搬文件 **[有副作用]**：

```bash
mv -n ~/下载/* ~/Downloads/ && ls -A ~/下载
```
- 预期：没有任何输出，说明目录已经空了。
- `-n` 的作用是遇到同名文件时不覆盖，而是留在原处。所以如果还有输出，说明有同名文件，要手动处理。
- 回滚：把文件 `mv` 回去。

删空目录 **[有副作用]**：

```bash
rmdir ~/下载
```
- 成功判据：没有输出。
- 失败判据：出现 `rmdir: failed to remove '...': Directory not empty`，说明里面还有东西，先停下来检查。
- `rmdir` 只能删除空目录，不会误删文件，比 `rm -r` 安全。

**2.3 修正文件管理器侧栏书签 [有副作用]**

```bash
cp ~/.config/gtk-3.0/bookmarks ~/.config/gtk-3.0/bookmarks.bak
printf 'file://%s\n' "$HOME/Documents" "$HOME/Music" "$HOME/Pictures" "$HOME/Videos" "$HOME/Downloads" > ~/.config/gtk-3.0/bookmarks
cat ~/.config/gtk-3.0/bookmarks
```
- 预期：输出 5 行 `file:///home/<你的用户名>/...`，重新打开文件管理器后侧栏显示英文名。
- 回滚：`mv ~/.config/gtk-3.0/bookmarks.bak ~/.config/gtk-3.0/bookmarks`

**2.4 修正 Chrome 的路径 [有副作用]**
- 下载位置：Chrome → Settings → Downloads → Location，改成 `~/Downloads`。
- `savefile.default_directory` 只是「另存为」对话框上次用过的目录，下次另存为时手动选一次 `Downloads` 就会被覆盖。`~/下载` 已经删掉的话，Chrome 会自动退回到默认目录。
- 其他程序（网盘客户端、IDE 等）如果也记住了中文路径，要到各自的设置里去改。

---

## 3. 装了 ibus，却找不到「智能拼音」

### 原因（按实际遇到的顺序）

1. `apt install ibus` 只装了**框架**。智能拼音是另一个包 **`ibus-libpinyin`**。
2. 引擎装好以后，还要把它加进 **GNOME 输入源列表**，否则按 Super+Space 没有东西可以切换。这台机器当时就卡在这一步：`ibus list-engine` 里能看到引擎，但 `gsettings` 的输入源列表里只有 `('xkb', 'us')`。
3. GNOME Settings 的「Add Input Source」对话框默认只列出少数几种语言，要先点底部的 **⋮（More）**，才能搜到 Chinese。

### 诊断（全部 [只读]）

```bash
dpkg -l ibus-libpinyin | tail -1                      # 预期: 以 ii 开头，表示已安装
ibus list-engine | grep -i pinyin                     # 预期: 有一行 "libpinyin - Intelligent Pinyin"
gsettings get org.gnome.desktop.input-sources sources # 预期: [('xkb', 'us'), ('ibus', 'libpinyin')]
```

`ibus list-engine` 里可能还会出现下面这些，它们**不是**智能拼音：
- `pinyin - Pinyin`：旧的 ibus-pinyin 引擎
- `m17n:zh:pinyin`：来自 `ibus-m17n` 包，会让添加列表很乱，用不到的话可以 `sudo apt remove ibus-m17n`

### 处理

**3.1 安装引擎 [有副作用]**

```bash
sudo apt install ibus-libpinyin
```
回滚：`sudo apt remove ibus-libpinyin`

**3.2 把输入法框架设为 ibus [有副作用]**

```bash
im-config -n ibus
```
- 这条命令会写入 `~/.xinputrc`。
- 回滚：改回原来的框架，例如 `im-config -n fcitx5`，或者直接删除 `~/.xinputrc`。
- 执行后要**注销、重新登录**，因为环境变量只在登录时读取。

**3.3 添加到 GNOME 输入源 [有副作用]**（下面两种方法任选一种，效果相同）

图形界面：Settings → Keyboard → Input Sources → **+ Add Input Source…** → **⋮** → 搜索 Chinese → **Chinese (China)** → **Chinese (Intelligent Pinyin)**。

命令行：

```bash
gsettings set org.gnome.desktop.input-sources sources "[('xkb', 'us'), ('ibus', 'libpinyin')]"
```
- 正常情况下没有输出，顶栏右上角会出现 `en` 输入法图标。
- 注意写的是 **`libpinyin`**，不是 `pinyin`。
- 回滚：`gsettings set org.gnome.desktop.input-sources sources "[('xkb', 'us')]"`
- 如果引擎在列表里找不到：先运行 `ibus restart`，再注销、登录。

**3.4 使用方法**
- **Super+Space**：在 English 和智能拼音之间切换。
- 在智能拼音里按 **Shift**：临时切到英文。
- 默认就是简体。

**3.5 验证 [只读]**

```bash
sleep 5; ibus engine
```

按回车后，5 秒内按 Super+Space 切到拼音，然后什么都别动，等结果出来。预期输出：`libpinyin`

> 坑：如果直接运行 `ibus engine`，结果永远是 `xkb:us::eng`。因为你要打出这条命令，就得先切回英文。所以要用 `sleep` 留出时间窗口。

如果还是切不过去，检查快捷键是否存在 **[只读]**：

```bash
gsettings get org.gnome.desktop.wm.keybindings switch-input-source
```

预期输出里包含 `'<Super>space'`。

---

## 4. fcitx5：不能切中英文 / 默认繁体 / 界面差

### 原因（在这台机器的配置文件里逐一核实过）

| 症状 | 原因 | 证据文件 |
|---|---|---|
| 不能切换中英文 | 输入法分组里**只有 `pinyin`，没有 `keyboard-us`**。fcitx5 的 Ctrl+Space 是在「第一项」和「当前项」之间切换，只有一项就没有英文可切 | `~/.config/fcitx5/profile` |
| 默认打出繁体 | 「简繁转换」（chttrans）对 pinyin 处于开启状态；它的开关快捷键 **Ctrl+Shift+F** 和 Cursor/VS Code 的「在文件中搜索」相同，很容易误触 | `~/.config/fcitx5/conf/chttrans.conf` 里有 `[EnabledIM] 0=pinyin` |
| 界面差、候选窗位置不对 | GNOME Wayland 下 fcitx5 的候选窗没法正常显示在 gnome-shell 之上，要装 **Kimpanel** 扩展 | fcitx wiki |
| 各种奇怪行为 | **ibus 和 fcitx5 同时在运行**。GNOME 总会启动 ibus-daemon，而 im-config 又让 GTK 程序走 fcitx | `~/.config/ibus/bus/` 和 `~/.config/fcitx5/` 同时存在 |

**原则：ibus 和 fcitx5 只能留一个。**

- GNOME 用户推荐 **ibus + ibus-libpinyin**：与 GNOME 原生集成，不需要装扩展。
- fcitx5 的拼音词库和整句输入质量更好，如果更看重打字准确度，就选 fcitx5，但要按方案 B 配好。

诊断 **[只读]**：

```bash
cat ~/.config/fcitx5/profile
grep -A2 EnabledIM ~/.config/fcitx5/conf/chttrans.conf
apt list --installed 'fcitx*' 2>/dev/null
cat ~/.xinputrc 2>/dev/null
```

### 方案 A（推荐）：停用 fcitx5，改用 ibus

1. 先按 §3 把 ibus 和智能拼音配好。
2. **[有副作用]** 卸载 fcitx5。要卸载的包以上面 `apt list` 的实际输出为准，例如：

   ```bash
   sudo apt remove --autoremove fcitx5 fcitx5-chinese-addons fcitx5-config-qt
   ```

   - 这一步只删软件包，`~/.config/fcitx5` 会保留。
   - 回滚：用 `sudo apt install` 装回同样的包，再执行 `im-config -n fcitx5`。
3. 确认 `im-config -n ibus` 已经执行过，然后注销、重新登录。

### 方案 B：保留 fcitx5，修好它

1. 运行 `fcitx5-configtool`，把 **Keyboard - English (US)** 加进输入法列表，并**放在第一位**，pinyin 放第二位。
2. 按一次 **Ctrl+Shift+F** 切回简体，然后到「简繁转换」附加组件的设置里把这个快捷键清空，避免和 IDE 冲突。
3. 从 extensions.gnome.org 安装 **Kimpanel**（Input Method Panel）扩展。
4. **[有副作用]** 执行 `im-config -n fcitx5`，再把 GNOME 的输入源设成只剩英文：

   ```bash
   gsettings set org.gnome.desktop.input-sources sources "[('xkb', 'us')]"
   ```

   这样 ibus 那边就不会再切换。
5. 注销、重新登录。

---

## 5. Chrome / Electron 应用里打不了中文

**只有遇到问题时才需要做。**

- **Chrome**
  - 如果以 Wayland 原生方式运行，需要加启动参数：`--enable-wayland-ime --wayland-text-input-version=3`
  - 另一种办法：打开 `chrome://flags`，把 **Preferred Ozone Platform** 设成 **X11**，改走 XWayland，这时用的是传统的 IM 模块。
- **Electron 应用**（Cursor、VS Code、Tabby、Claude 桌面版等）
  - 它们只支持 text-input-v1，而 GNOME 只实现了 v3。
  - 所以在 GNOME 下**保持默认的 XWayland 模式**，不要强制加 `--ozone-platform=wayland`。

---

## 6. 程序栏莫名出现 portal 窗口：「Open with — No apps available — `bitbrowser://cc/`」

### 原因

这台机器上的调用链是：

```
抖音网页 (www.douyin.com) 里的脚本尝试打开 bitbrowser://…
  → Chrome 里存着「始终允许 douyin.com 打开 bitbrowser 链接」，于是不再询问
  → 链接被交给系统
  → 系统里没有处理 bitbrowser:// 的程序
  → GNOME portal 弹出「No apps available」
```

- 证据在 Chrome 的 `Default/Preferences` 里：

  ```json
  "protocol_handler": {"allowed_origin_protocol_pairs": {
    "https://www.douyin.com": {"bitbrowser": true}
  }}
  ```

- BitBrowser（比特浏览器）是一种指纹浏览器。**推测**网站是在做风控检测，看你有没有装这个客户端。这一点没有核实。
- 那条「始终允许」多半是以前在 Chrome 的「Open xdg-open?」提示框里顺手勾上的。

**通用排查方法**：弹窗里出现的是别的协议（`xxx://`）也一样，去 Chrome 配置里找是哪个网站被允许了 **[只读]**：

```bash
python3 -c "import json,os;p=json.load(open(os.path.expanduser('~/.config/google-chrome/Default/Preferences')));print(json.dumps(p.get('protocol_handler'),indent=1))"
```

- 检验判据：关掉所有来自该网站的标签页后，弹窗不应再出现。
- 如果还会出现，说明另有来源。可以看 Chrome 扩展，或者运行 `dbus-monitor --session "interface='org.freedesktop.portal.OpenURI'"`，看是谁在调用 portal。

### 方案 A（推荐）：撤销这条授权

Chrome 的设置界面里没有地方能删这一项，只能改配置文件，**而且必须先彻底退出 Chrome**。

**1. 确认 Chrome 已完全退出 [只读]**

```bash
pgrep -x chrome
```

预期：没有任何输出。如果有输出，先在 Chrome 菜单里点 Exit。

**2. 备份 [有副作用：只新建一个备份文件]**

```bash
cp ~/.config/google-chrome/Default/Preferences ~/.config/google-chrome/Default/Preferences.bak-bitbrowser
```

**3. 删除这一项 [有副作用：改写 Preferences]**

```bash
python3 - <<'E'
import json, os
f = os.path.expanduser('~/.config/google-chrome/Default/Preferences')
p = json.load(open(f))
print(p['protocol_handler']['allowed_origin_protocol_pairs'].pop('https://www.douyin.com'))
json.dump(p, open(f, 'w'), separators=(',', ':'))
E
```
- 预期输出：`{'bitbrowser': True}`
- 失败判据：出现 `KeyError: 'https://www.douyin.com'`，说明这一项已经不在了，文件不会被改动。

**4. 验证 [只读]**

```bash
grep -c bitbrowser ~/.config/google-chrome/Default/Preferences
```

预期输出：`0`。这条 grep 的退出码是 1，属于正常情况。

**回滚**：在 Chrome 退出的状态下，把备份拷回去：

```bash
cp ~/.config/google-chrome/Default/Preferences.bak-bitbrowser ~/.config/google-chrome/Default/Preferences
```

之后网站再尝试打开这类链接，Chrome 会先在标签页里询问：点 **Cancel**，**不要勾**「Always allow」。

### 方案 B：注册一个什么都不做的处理程序

适用情况：Chrome 的询问框也出现得太频繁。

**[有副作用]**

```bash
cat > ~/.local/share/applications/bitbrowser-null.desktop <<'E'
[Desktop Entry]
Type=Application
Name=bitbrowser null handler
Exec=true %u
NoDisplay=true
MimeType=x-scheme-handler/bitbrowser;
E
xdg-mime default bitbrowser-null.desktop x-scheme-handler/bitbrowser
xdg-mime query default x-scheme-handler/bitbrowser
```

- 预期：最后一行输出 `bitbrowser-null.desktop`。
- 代价：网站检测脚本拿到的信号会变，这一点没有验证。
- 回滚：删掉那个 `.desktop` 文件，再删掉 `~/.config/mimeapps.list` 里 `x-scheme-handler/bitbrowser=` 那一行。

---

## 7. 查看与安装字体

### 查看（全部 [只读]）

图形界面用 **Fonts** 应用（`gnome-font-viewer`），能预览，也能直接安装字体。系统里没有的话，先装：`sudo apt install gnome-font-viewer`。

命令行：

```bash
fc-list : family | sort -u | less           # 所有字体族
fc-list :lang=zh family | sort -u           # 支持中文的字体
fc-match sans-serif; fc-match monospace     # 系统默认的无衬线字体、等宽字体
gsettings get org.gnome.desktop.interface font-name
gsettings get org.gnome.desktop.interface monospace-font-name
```

字体文件放在这几个位置：

| 路径 | 来源 |
|---|---|
| `/usr/share/fonts/` | apt 安装的 |
| `/usr/local/share/fonts/` | 手动给所有用户装的 |
| `~/.local/share/fonts/` | 只给自己装的（推荐） |

### 安装

**方式 A：用 apt 装 [有副作用]**

```bash
sudo apt install fonts-noto-cjk fonts-noto-color-emoji fonts-jetbrains-mono
```

- 这三个包分别是：中文字体、彩色 emoji、编程用的等宽字体。
- 回滚：`sudo apt remove <包名>`

**方式 B：手动安装 `.ttf`/`.otf` [有副作用]**

```bash
mkdir -p ~/.local/share/fonts
cp ~/Downloads/SomeFont/*.{ttf,otf} ~/.local/share/fonts/ 2>/dev/null
fc-cache -f
fc-list | grep -i somefont
```

- 最后一条 `fc-list` 能列出新字体，才算装好。
- 另一种装法：在 Fonts 应用里双击字体文件，点 Install，效果一样。
- 回滚：删掉 `~/.local/share/fonts/` 里对应的文件，再运行 `fc-cache -f`。
- 已经打开的程序要重启后才能看到新字体。

装好之后在哪里换字体：
- 界面字体：装 **GNOME Tweaks**（`sudo apt install gnome-tweaks`）。
- Tabby：Settings → Appearance → Font。

### 改成英文后，汉字可能显示成日文字形（⚠️ 没有在本机验证过）

Noto CJK 在非中文 locale 下，可能把汉字优先匹配到日文版字形（JP）。

**诊断 [只读]**

```bash
fc-match sans-serif:lang=zh-cn
fc-match sans-serif
```

- 两行都是 `... CJK SC`：没有这个问题。
- 第二行是 `... JP`：确认有问题，按下面修复。

**修复 [有副作用：新建一个文件]**

```bash
mkdir -p ~/.config/fontconfig
cat > ~/.config/fontconfig/fonts.conf <<'E'
<?xml version="1.0"?>
<!DOCTYPE fontconfig SYSTEM "fonts.dtd">
<fontconfig>
  <alias><family>sans-serif</family><prefer><family>Noto Sans CJK SC</family></prefer></alias>
  <alias><family>serif</family><prefer><family>Noto Serif CJK SC</family></prefer></alias>
  <alias><family>monospace</family><prefer><family>Noto Sans Mono CJK SC</family></prefer></alias>
</fontconfig>
E
fc-cache -f && fc-match sans-serif
```

- 预期：最后输出里含有 `Noto Sans CJK SC`。
- 失败判据：出现 `Fontconfig error: ...`，说明 XML 写错了。
- 注意：这个配置会把西文的 sans-serif 也改用 Noto CJK SC 里自带的拉丁字形。
- 回滚：`rm ~/.config/fontconfig/fonts.conf && fc-cache -f`，然后重新登录。

---

## 附录 A：同类机器的其他配置点（⚠️ 通用做法，没有在 slaanesh 上实测）

### A.1 合盖只用外接屏幕，不想让机器休眠

systemd-logind 的默认行为：
- 连着外接显示器时，合盖不休眠（`HandleLidSwitchDocked=ignore`）
- 插着电源、但没有外接显示器时，合盖仍会休眠

**[只读]** 查看当前的合盖设置：

```bash
systemd-analyze cat-config systemd/logind.conf | grep -i lid
```

**[有副作用]** 插着电源时合盖也不休眠：

```bash
sudo mkdir -p /etc/systemd/logind.conf.d
printf '[Login]\nHandleLidSwitchExternalPower=ignore\n' | sudo tee /etc/systemd/logind.conf.d/lid.conf
```

- **重启后生效**。不要为了立即生效去运行 `systemctl restart systemd-logind`，那会把当前的图形会话踢掉。
- 回滚：`sudo rm /etc/systemd/logind.conf.d/lid.conf`，然后重启。

### A.2 本地 SOCKS5 代理（示例端口 127.0.0.1:10808）

- **图形应用**（包括 Chrome，它在 Linux 上读取 GNOME 的代理设置）：Settings → Network → Proxy → Manual，Socks Host 填 `127.0.0.1`，端口 `10808`。
- **终端（当前 shell）**：

  ```bash
  export ALL_PROXY=socks5h://127.0.0.1:10808
  ```

  用 `socks5h` 而不是 `socks5`，DNS 也会交给代理去解析。

- **git [有副作用]**：

  ```bash
  git config --global http.proxy socks5h://127.0.0.1:10808
  ```

  回滚：`git config --global --unset http.proxy`

- **apt [有副作用]**：

  ```bash
  echo 'Acquire::http::Proxy "socks5h://127.0.0.1:10808";' | sudo tee /etc/apt/apt.conf.d/95proxy
  ```

  回滚：删掉这个文件。

- 注意：代理客户端没启动时，上面这些配置会让对应程序完全连不上网。排错时先检查代理客户端在不在运行 **[只读]**：

  ```bash
  ss -ltnp | grep 10808
  ```

### A.3 NVIDIA 专有驱动自检（[只读]）

```bash
nvidia-smi                          # 能列出 GPU 型号和驱动版本，才说明专有驱动已加载
lsmod | grep -E '^nvidia|nouveau'   # 应该看到 nvidia 模块，而不是 nouveau
ubuntu-drivers list                 # 列出可安装的驱动版本
```

---

## 经验教训

1. **一次只排查一个层次。** 界面语言、XDG 目录、应用里记住的路径、输入法框架、输入引擎、GNOME 输入源列表，这几层互相独立。改好一层，不代表其他层会跟着变。
2. **输入法框架只能留一个。** 大部分「玄学」问题都来自 ibus 和 fcitx5 同时运行。
3. **验证步骤本身可能改变被测状态。** 例如打字运行 `ibus engine` 前，你会先切回英文，所以要用 `sleep` 留出时间窗口。
4. **「没报错」不等于「成功」。** 每一步都要对照预期输出（引擎名、gsettings 的值、`grep -c` 的计数）来确认。
5. **改应用配置文件前，先退出应用、再备份。** Chrome 运行时会用内存里的状态覆盖 `Preferences`，在运行中改文件等于白改。
