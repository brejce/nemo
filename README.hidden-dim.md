# nemo-dim-hidden（中文说明）

给 Nemo 文件管理器加三个小改动：

1. **隐藏文件淡显**：以 `.` 开头的文件和文件夹，在**图标视图、紧凑视图、列表视图**里
   统一降到 50% 透明度（图标 + 文件名），像"半隐身"，但仍然可见、可点、可改名。
   列表中只用主题自带的灰色（insensitive）。
2. **隐藏文件排最后**：排序时隐藏项一律排到非隐藏项之后，正序倒序都是。
   顺序为：可见目录 → 可见文件 → 隐藏目录 → 隐藏文件。
3. **侧栏 tree view 永远不显示隐藏项**：上游行为是"跟随 Ctrl+H"，开了显示隐藏文件
   之后目录树里也会冒出 `.xxx`；这里改成目录树恒定干净，开关只作用于列表/图标视图。

`Ctrl+H` 的开关语义没变，只是不再影响侧栏的目录树。

## 仓库与分支

- fork：https://github.com/brejce/nemo
- 分支：`hidden-dim-6.6.4`，基于上游 tag `6.6.4`，上面挂 4 个代码提交（外加文档提交）
- 改动合计：5 个文件、约 +165 行

| 提交 | 内容 |
| --- | --- |
| `989d72a` icon view: dim hidden files | `libnemo-private/nemo-icon-canvas-item.{c,h}`、`nemo-icon-container.c`：canvas item 加 `is_hidden` 标志，绘制时整块 `cairo_push_group` + `cairo_paint_with_alpha(0.5)` |
| `aab0d6a` list view: dim hidden files | `src/nemo-list-view.c`：所有文字列用 cell data func 设 `sensitive=FALSE`；Name 列图标用一个内嵌 `GtkCellRendererPixbuf` 子类做 50% alpha（GTK3 的 pixbuf renderer 不认 sensitive）；带 `editable` guard 以免隐藏文件改不了名 |
| `6ec9549` sort hidden files last | `libnemo-private/nemo-file.c`：在比较函数里把 hidden 作为分组键，插在"目录优先"之前 |
| `6dc39db` tree sidebar: never show hidden files | `src/nemo-tree-sidebar.c`：`update_filtering_from_preferences()` 里把 `fm_tree_model_set_show_hidden_files()` 恒传 FALSE（传 FALSE 会同时销毁已有的隐藏行） |

## 构建与安装

装到 `/usr/local`，**不经过 pacman**，与系统包并存（靠路径优先级遮蔽它）。

```bash
# 首次需要工具链（intltool 包已从 Arch 仓库下架，脚本内置了官方 intltool-merge）
sudo pacman -S --needed gobject-introspection glib2-devel

~/Project/nemo-dim-hidden/build.sh install
```

脚本做的事：

1. clone/fetch fork 的 `hidden-dim-6.6.4` 到 `src/`
2. `meson setup --prefix=/usr/local --libdir=lib` + `ninja`
3. `sudo ninja install`
4. 把 `/usr/local/share/applications/nemo*.desktop` 里的 `Exec=nemo` 改成绝对路径
   `/usr/local/bin/nemo`
5. 复制一份到 `~/.local/share/applications/`（部分启动器按 ID 解析 desktop 文件时
   不看 `/usr/local/share`，只认用户级目录）

安装后 `nemo -q` 再打开即可生效。验证跑的是哪个二进制：

```bash
pgrep -a -x nemo          # 应该是 /usr/local/bin/nemo
/usr/local/bin/nemo --version
```

## 终端里的 `nemo` 为什么可能还是系统版

安装脚本只负责**菜单 / 双击目录 / D-Bus** 这几条路径，它们靠 desktop 文件里的绝对
`/usr/local/bin/nemo` 保证。终端里敲 `nemo` 走的是 PATH 解析，取决于你的 shell / 会话环境：

```bash
which -a nemo
# /usr/bin/nemo          ← PATH 里 /usr/bin 在前时会命中系统版（没有补丁）
# /usr/local/bin/nemo
```

三种处理方式，任选其一：

```bash
# 1. 最省事：写绝对路径
/usr/local/bin/nemo ~/某个目录

# 2. 把 /usr/local/bin 提到 PATH 前面（Arch 默认 /etc/profile 本来就是这么排的）
fish_add_path --move-to-front /usr/local/bin      # fish
export PATH=/usr/local/bin:$PATH                  # bash/zsh

# 3. 只给 nemo 单独指路，不动全局 PATH
# fish: ~/.config/fish/config.fish
function nemo
    command /usr/local/bin/nemo $argv
end
# bash: ~/.bashrc
alias nemo=/usr/local/bin/nemo
```

`build.sh install` 结束时会自动检测。如果 `command -v nemo` 不是 `/usr/local/bin/nemo`：

- 能识别出你的 shell（fish / bash / zsh）→ **问你一句**「是否让脚本加一段配置」，
  回答 y 就写进对应的 rc 文件，并带上 `# >>> nemo-dim-hidden >>>` 标记；
- 非交互运行（管道、CI）→ 不会改你的配置，只打印手动方案；
  想强制自动写，用 `NEMO_PATHFIX_YES=1 ./build.sh install`；
- 识别不出 shell，或你回答 n → 打印三条手动方案（见上）。

随时可以单独跑这段逻辑，不必重新构建：

```bash
./build.sh pathfix
```

`./build.sh uninstall` 会把这段带标记的配置一起删掉。

## 卸载（回退到 pacman 的版本）

```bash
~/Project/nemo-dim-hidden/build.sh uninstall
rm -f ~/.local/share/applications/nemo.desktop
```

## 已知边界

- **版本被钉在 6.6.4**：`/usr/local` 优先，系统 nemo 升级后你用的还是这一份，
  拿不到上游新功能。想跟新版就让 fork 基于新 tag 重建分支。
- **`libnemo-extension.so` 实际加载的是 `/usr/lib` 那份**（安装后 RUNPATH 被 meson 去了）。
  若 Arch 大版本升级把 soname 升到 `.so.2`，补丁版会启动失败，届时重建或卸载。
- **第三方扩展不加载**：编译进去的扩展目录是 `/usr/local/lib/nemo/extensions-3.0`（不存在），
  系统扩展在 `/usr/lib/nemo/extensions-3.0`。要扩展需另配（或 `nemo-python` + 用户级扩展）。
- **`org.Nemo` 的 D-Bus 激活仍指向系统版**：`/usr/share/dbus-1/services/nemo.service` 是登录时
  就被 dbus-daemon 记下的，`/usr/local` 那份要重启 session bus 才生效。菜单打开、双击目录
  不走这条路径，日常无感。
- **重命名时整列临时回亮**：`sensitive` 是 renderer 级属性，不是行级；编辑结束后恢复。
- **选中 + 淡显的对比度取决于主题**（Adwaita 下偏暗），想调就改 `paint_with_alpha` 的 0.5。
