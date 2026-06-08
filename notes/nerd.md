# JetBrainsMono Nerd Font 压缩参数

基于本地 LazyVim 插件源码实际使用的 697 个符号分析生成。

## --unicodes 最终参数

```
U+0020-007F,\
U+2500-259F,\
U+2665,U+26A1,\
U+E0A0-E0D7,\
U+E200-E2A9,\
U+E300-E3E3,\
U+E5FA-E6B7,\
U+E700-E8EF,\
U+EA60-EC1E,\
U+ED00-EFCE,\
U+F000-F2FF,\
U+F300-F381,\
U+F400-F533,\
U+F0001-F1AF0
```

## 与原始命令对比

| # | 范围 | 图标集 | 类型 | 说明 |
|---|---|---|---|---|
| 1 | `U+0020-007F` | ASCII | 不变 | 基本拉丁字母 |
| 2 | `U+2500-259F` | Box Drawing + Block Elements | 不变 | webssh 终端渲染需要 |
| 3 | `U+2665,U+26A1` | Octicons 标准位置 | **新增** | `♥` 和 `⚡` 两个符号 |
| 4 | `U+E0A0-E0D7` | Powerline + Powerline Extra | **扩展** | 原 E0A0-E0C8，补全到 E0D7 |
| 5 | `U+E200-E2A9` | Font Awesome Extension | **新增** | 5 个符号被引用 |
| 6 | `U+E300-E3E3` | Weather Icons | 不变 | 仅 E348 被用 |
| 7 | `U+E5FA-E6B7` | Seti-UI + Custom | 不变 | 56 个符号被用 |
| 8 | `U+E700-E8EF` | Devicons | **新增** | 29 个符号被 mini.icons 使用 |
| 9 | `U+EA60-EC1E` | Codicons | **扩展** | 原 EA60-EBEB，补到 EC1E |
| 10 | `U+ED00-EFCE` | Font Awesome v3 | **新增** | Nerd Font v3 新版位置，关键缺失！ |
| 11 | `U+F000-F2FF` | Font Awesome v2 | **扩展** | 原 F000-F2E0，略扩到 F2FF |
| 12 | `U+F300-F381` | Font Logos | **新增** | 7 个符号被引用 |
| 13 | `U+F400-F533` | Octicons | 不变 | 43 个符号被用 |
| 14 | `U+F0001-F1AF0` | Material Design v3 | 不变 | 399/6896 符号被用（SMP 区域） |

## 排除的未使用图标集

| 范围 | 图标集 | 排除原因 |
|---|---|---|
| `U+23FB-U+23FE` | IEC Power Symbols | 插件中 0 使用 |
| `U+2B58` | IEC Power Symbols | 插件中 0 使用 |
| `U+E000-E00A` | Pomicons | 插件中 0 使用 |
| `U+F500-U+FD46` | Material Design v2 | Nerd Font v3 已废弃该区域 |

## 全量 pyftsubset 命令

```bash
pyftsubset "$HOME/tools/anaconda3/envs/dev/lib/python3.14/site-packages/webssh/static/css/fonts/JetBrainsMonoNerdFontMono-Regular.ttf" \
  --unicodes="\
U+0020-007F,\
U+2500-259F,\
U+2665,U+26A1,\
U+E0A0-E0D7,\
U+E200-E2A9,\
U+E300-E3E3,\
U+E5FA-E6B7,\
U+E700-E8EF,\
U+EA60-EC1E,\
U+ED00-EFCE,\
U+F000-F2FF,\
U+F300-F381,\
U+F400-F533,\
U+F0001-F1AF0" \
  --flavor=woff2 \
  --layout-features="calt,liga,kern" \
  --no-hinting \
  --no-glyph-names \
  --drop-tables+=PfEd,FFTM,LTSH,VDMX,hdmx,gasp \
  --output-file="$HOME/tools/anaconda3/envs/dev/lib/python3.14/site-packages/webssh/static/css/fonts/JetBrainsMonoNerdFontMono-Regular.woff2"
```

## 使用统计

| 范围 | 图标集 | 使用/总数 | 使用率 |
|---|---|---|---|
| `U+0020-007F` | ASCII | 96/96 | 100.0% |
| `U+2500-259F` | Box Drawing + Block | 0/160 | 0.0% (webssh 需要) |
| `U+2665,U+26A1` | Octicons 标准 | 2/2 | 100.0% |
| `U+E0A0-E0D7` | Powerline 全套 | 12/56 | 21.4% |
| `U+E200-E2A9` | Font Awesome Extension | 5/170 | 2.9% |
| `U+E300-E3E3` | Weather Icons | 1/228 | 0.4% |
| `U+E5FA-E6B7` | Seti-UI + Custom | 56/190 | 29.5% |
| `U+E700-E8EF` | Devicons | 29/496 | 5.8% |
| `U+EA60-EC1E` | Codicons | 61/447 | 13.6% |
| `U+ED00-EFCE` | Font Awesome v3 | 5/719 | 0.7% |
| `U+F000-F2FF` | Font Awesome v2 | 74/768 | 9.6% |
| `U+F300-F381` | Font Logos | 7/130 | 5.4% |
| `U+F400-F533` | Octicons | 43/308 | 14.0% |
| `U+F0001-F1AF0` | Material Design v3 | 399/6896 | 5.8% |
| **总计** | | **790/10896** | **7.2%** |

## 贡献最多的插件源文件

| 符号数 | 文件 |
|---|---|
| 386 | mini.icons/lua/mini/icons.lua |
| 53 | snacks.nvim/.../picker/config/defaults.lua |
| 47 | LazyVim/lua/lazyvim/config/init.lua |
| 33 | catppuccin/.../integrations/lsp_saga.lua |
| 27 | which-key.nvim/.../config.lua |
| 26 | trouble.nvim/.../config/init.lua |
| 26 | grug-far.nvim/.../opts.lua |
| 25 | which-key.nvim/.../icons.lua |
| 19 | blink.cmp/.../appearance.lua |
| 19 | noice.nvim/.../icons.lua |
