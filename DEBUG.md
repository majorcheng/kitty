# `bold_font` 仍显示粗体的调试记录

## Observations

- 用户截图显示 `LS_COLORS` 产生的 ANSI 粗体属性仍使用较粗字形。
- 当前安装包 `/Applications/kitty.app` 能运行，版本为 kitty 0.49.2；其资源中的 `kitty/fonts/core_text.py` 已包含 `display_name` 索引修复。
- 当前配置文件 `/Users/major/.config/kitty/kitty.conf` 第 10 行为 `bold_font Lilex Nerd Font Mono Regular`。
- 通过当前安装包执行 `load_config('/Users/major/.config/kitty/kitty.conf')` 后，`font_family` 为 `Lilex Nerd Font Mono`，`bold_font` 的 `FontSpec` 为系统字体值 `Lilex Nerd Font Mono Regular`。
- CoreText `all_fonts_map(True)['full_map']` 中存在 `lilex nerd font mono regular`，对应 `LilexNFM-Regular`。
- 通过当前安装包执行 `get_font_files()` 后，medium 为 `LilexNFM-Regular`，bold 为 `Menlo-Bold`，并输出 `The font Lilex Nerd Font Mono Regular was not found, falling back to Menlo`。
- 直接调用 `find_best_match('Lilex Nerd Font Mono Regular', bold=True)` 能返回 `LilexNFM-Regular`。

## Hypotheses

### H1: 显式系统字体的精确匹配被 `ignore_face` 错误跳过（ROOT HYPOTHESIS）

- Supports: 精确 full-name 索引存在；直接匹配成功；`get_font_files()` 只有在传入 medium face 作为 `ignore_face` 后才回退到 Menlo。
- Conflicts: 尚未确认 `ignore_face` 是否只应对 `auto` 生效。
- Test: 用相同的 Regular descriptor 调用 `find_best_match(..., ignore_face=regular)`，并对比不传 `ignore_face` 的结果。

### H2: 安装包仍在使用旧的 Python 字体索引代码

- Supports: 用户在替换后仍看到旧行为。
- Conflicts: 当前安装包源码包含 `display_name` 索引；运行时 `full_map` 也能看到新键。
- Test: 从 `/Applications/kitty.app` 直接打印模块路径、索引键和匹配结果。

### H3: 已选中的 Regular face 在 C 渲染层被 ANSI bold 属性合成加粗

- Supports: ANSI SGR 1 会把 cell 标记为 bold，截图表现为粗体。
- Conflicts: `get_font_files()` 当前实际 bold descriptor 是 `Menlo-Bold`，在进入渲染层前已经选错了字体。
- Test: 先确认 `set_font_family()` 提供给 bold 槽位的 descriptor，再检查 C 层 `font_for_cell()` 的槽位选择。

## Experiments

- `find_best_match('Lilex Nerd Font Mono Regular', bold=True)` 返回 `LilexNFM-Regular`。
- `find_best_match('Lilex Nerd Font Mono Regular', bold=True, ignore_face=<LilexNFM-Regular>)` 会跳过唯一的 full-name 候选，随后因查询值不是 family key 而回退到 Menlo；这与 `get_font_files()` 的结果一致。
- `get_font_files()` 的 medium/bold 输出确认问题发生在字体槽位初始化之前，尚未进入像素渲染。
- 将 `ignore_face` 限定为 `auto` 系统字体后，针对 `bold_font Lilex Nerd Font Mono Regular` 的回归测试通过，源码运行时 bold face 为 `LilexNFM-Regular`。

## Root Cause

`bold_font` 的显式值没有被识别为 `auto`，但 `get_font_from_spec()` 仍把 medium face 作为 `ignore_face` 传给平台匹配器；当用户明确指定的 full/display name 正好就是 Regular face 时，唯一精确候选被跳过并错误回退到 Menlo-Bold。

## Fix

在 `get_font_from_spec()` 中只对 `auto` 系统字体传递 `ignore_face`，并添加覆盖真实 `get_font_files()` 路径的回归测试。

## Delivery

- `./test.py coretext_explicit_regular_bold_font`、`./test.py coretext_full_name_matches_localized_style`、`./test.py --module fonts` 和 `./test.py --module options` 通过。
- 新 app 已重新构建并安装到 `/Applications/kitty.app`；旧本地构建备份为 `/Applications/kitty.app.backup-20261021-194008`，原官方包备份为 `/Applications/kitty.app.backup-20261021-191519`。
- `/opt/homebrew/bin/kitty +runpy` 从真实配置加载后输出 `bold_runtime: LilexNFM-Regular`。
