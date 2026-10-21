# 修复显式 `bold_font` 字体名匹配

## 计划

- 在 CoreText 字体索引中同时支持 `display_name` 和现有 `family + style` 键。
- 增加本地化 style 与英文 display name 的粗体选择回归场景。
- 更新字体选择文档和未发布 changelog。

## 影响文件

- `kitty/fonts/core_text.py`
- `kitty_tests/fonts.py`
- `docs/kittens/choose-fonts.rst`
- `docs/changelog.rst`

## 完成标准

- `bold_font Lilex Nerd Font Mono Regular` 能命中 Regular face，即使请求来自 ANSI 粗体。
- 既有 family/style 匹配、ANSI 状态和颜色行为保持不变。
- 相关构建、字体测试和差异检查通过。

## 验证结果

- CoreText full/display name alias and localized-style regression test added。
- `env HOMEBREW_NO_AUTO_UPDATE=1 PATH=/tmp/kitty-ci-venv/bin:/opt/homebrew/bin:/usr/bin:/bin /tmp/kitty-ci-venv/bin/python .github/workflows/ci.py build` 通过。
- `./test.py coretext_full_name_matches_localized_style` 通过。
- `./test.py --module fonts`：23 个测试通过，4 个环境相关场景跳过。
- `./test.py --module options`：7 个测试通过。
- `kitty/launcher/kitty +runpy` 真实配置验证返回 `LilexNFM-Regular Lilex Nerd Font Mono Regular`。
- `setup.py kitty.app --update-check-interval=0 --verbose` 完整 app bundle 构建通过。
- `/Applications/kitty.app` 已替换为本地 arm64 构建，旧包备份为 `/Applications/kitty.app.backup-20261021-191519`。
- `/opt/homebrew/bin/kitty --version`、`kitten --version` 和真实 CoreText 匹配验证通过；配置文件显式加载后返回 `Lilex Nerd Font Mono Regular`，匹配到 `LilexNFM-Regular`。
- 修复了显式系统字体与 medium face 相同而被 `ignore_face` 跳过的问题；`get_font_files()` 现在为 `bold_font Lilex Nerd Font Mono Regular` 保留 `LilexNFM-Regular`。
- 新增 `coretext_explicit_regular_bold_font` 回归测试；字体模块测试更新为 24 个测试通过。
- 修复后的 app 已重新安装，上一版本地构建备份为 `/Applications/kitty.app.backup-20261021-194008`。
- 新 app 已 ad hoc 重签名，`codesign --verify --deep --strict` 通过。
- `git diff --check` 通过；前一轮字体修复阶段按要求未提交 Git，本轮 fork/Actions 改动将随当前修复一起提交。
- `sphinx-build -b dummy -q docs /tmp/kitty-docs` 在 Sphinx 启动阶段退出码 139，未完成文档构建验证。
- `spctl` 未通过，因为本地构建没有 Developer ID 公证；当前包依赖本机 Homebrew 的 Python 3.14 和动态库。
- `origin` 已切换为 `git@github.com:majorcheng/kitty.git`，本地 `master` 跟踪 fork 的 `master`。
- `.github/workflows/ci.yml` 的现有 macOS `brew` job 已增加 `workflow_dispatch`、`.app` zip、SHA256 和 GitHub artifact 上传；本机用 `ditto` 归档测试通过。
