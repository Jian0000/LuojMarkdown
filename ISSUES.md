# 问题排查记录

> 本文记录 Hugo + PaperMod + Cloudflare Pages 博客搭建过程中遇到的所有问题及解决方案。

---

## 1. 网站显示 Hugo 默认页面，没有自己的文章

**现象：** 浏览器打开博客，显示 Hugo 初始化的空页面（标题 `My New Hugo Project`），而非自己写的测试文章。

**原因：** 文章 frontmatter 中 `draft = true`，Hugo 生产构建默认跳过草稿。

**解决：**
```diff
- draft = true
+ draft = false
```

> 本地预览用 `hugo server -D`（`-D` 包含草稿），但 Cloudflare 构建时只执行 `hugo`，不包含草稿。

---

## 2. 新建 `.md` 文章不渲染

**现象：** 新建文章后 `git push`，Cloudflare 构建成功但页面 404。

**原因：** 文章 frontmatter 缺少开头的 `+++`，Hugo 未能识别文章元数据。

**错误示例：**
```markdown
date = '2026-06-05T10:10:40+08:00'
draft = false
title = '文章标题'

+++

正文...
```

**正确示例：**
```markdown
+++
date = '2026-06-05T10:10:40+08:00'
draft = false
title = '文章标题'
+++

正文...
```

---

## 3. 文章 `date` 设为未来时间导致不渲染

**现象：** 文章文件格式正确，`draft = false`，本地 `hugo` 能列出该页面，但生成时被跳过，`public/` 下没有对应 HTML。

**原因：** Hugo 将 frontmatter 中的 `date` 作为 `publishDate`（发布时间）。如果时间还没到（哪怕只差 1 分钟），Hugo 视为"未发布"而跳过渲染。

**调试方法：**
```bash
# 列出 Hugo 识别的所有页面（包括"未发布"的）
hugo list all

# 如果 permalink 存在但 public/ 下无文件，就是日期问题
```

**解决：**
- `date` 使用**过去时间**，不要设为未来
- 如果确实需要定时发布，构建时加 `--buildFuture` 参数

---

## 4. PowerShell 创建文件带 BOM 导致 Hugo 异常

**现象：** 用 `Set-Content -Encoding utf8` 创建的文章，Hugo 不渲染。

**原因：** PowerShell `Set-Content -Encoding utf8` 自动在文件头部添加 UTF-8 BOM（3 字节：`EF BB BF`），Hugo 解析 frontmatter 时可能受影响。

**验证方法：**
```powershell
$bytes = [System.IO.File]::ReadAllBytes('path\file.md')
# 如果前 3 字节是 EF BB BF，说明有 BOM
```

**解决：**
- 用 VS Code / Notepad++ 等编辑器创建 `.md` 文件
- 保存时选择 **UTF-8 无 BOM** 编码
- 避免用 PowerShell `Set-Content` 创建 Hugo 文章

---

## 5. Cloudflare Pages 修改不生效（最坑）

**现象：** GitHub 代码已更新，本地构建正常，但线上页面始终不变 —— 一直显示旧版。

**原因（两层叠加）：**

| 层级 | 问题 |
|------|------|
| ① | PaperMod 主题是 **git submodule**，Cloudflare Pages 默认**不拉取子模块** |
| ② | 之前 `public/`（构建产物）被提交到仓库，Cloudflare 检测到已有静态文件就**直接拿来用**，根本没运行 `hugo build` |

**为什么之前能成功？**
```
之前: GitHub(含 public/) → Cloudflare 直接用现成 HTML → 成功
现在: GitHub(无 public/) → Cloudflare 运行 hugo → PaperMod 缺失 → 失败 → 显示旧版
```

**解决：**

**步骤 1** — Cloudflare Pages → 项目 → **Settings** → **Build & Deploy** → 勾选 **Include Git Submodules**

**步骤 2** — 在仓库中添加 `.gitignore`：
```
public/
.hugo_build.lock
```

**步骤 3** — 清理已提交的 `public/`：
```bash
git rm -r --cached public/
git commit -m "chore: 移除 public/ 构建产物"
```

---

## 6. 侧边栏分类目录展开后无子文章

**现象：** 侧边栏看到分类名（测试、笔记），点击展开后子文章列表为空。

**原因：** 同问题 3 —— 子目录下的文章 `date` 设在了构建时的时间之后。

**解决：** 将文章 `date` 改为已过去的时间点。

---

## 7. Hugo `in` 函数参数错误

**现象：** 构建时报错 `wrong number of args for in: want 2 got 3`。

**原因：** Hugo 模板中 `in` 函数只接受 2 个参数（集合 + 元素），传入 3 个参数会报错。

**错误写法：**
```go
{{ if in (lower .Title) "test" "测试" }}  // 3 个参数，报错
```

**正确写法：**
```go
{{ $t := lower .Title }}
{{ if or (in $t "test") (in $t "测试") }}  // 每次只传 2 个
```

---

## 8. 背景渐变切换不生效

**现象：** 点击背景选择器中的渐变色，页面毫无变化。

**原因（两层）：**
1. `body::before` 伪元素设置了 `z-index: -1`，被 `body` 的背景色完全遮住
2. 渐变透明度太低（8%），即使没被遮也几乎看不见

**解决：**
- 改为直接应用到 `body`：`background: var(--bg-overlay, none), var(--theme)`
- 透明度从 8% 提高到 20%

---

## 9. 侧边栏不支持多层嵌套 section

**现象：** 三层目录结构下（如 安卓应用 → Java基础 → 文章），侧边栏所有文章混在一起显示，分类层级丢失。

**原因：**
- `.RegularPages` 返回所有后代页面（含嵌套子 section 的文章），不是直接子页面
- 没有 `_index.md` 的子目录不被 Hugo 识别为 section

**解决：**
- 创建所有子目录的 `_index.md` 文件
- 侧边栏改用 `.Pages` 过滤 `.Kind = "page"` 获取直接子页面
- 手动递归渲染子 section 的层级结构

---

## 10. `_index.md` 缺失导致 Hugo 不识别子 section

**现象：** `hugo list all` 只列出顶级 section，子目录（如 `01-Java基础`）不被识别。

**原因：** Hugo 只将含 `_index.md` 的目录视为 section。没有 `_index.md` 的目录只是"分支"，其内页面归属到最近的上级 section。

**解决：** 为每个子目录创建 `_index.md`，设置 `title` 字段。

---

## 11. 护眼模式代码块文字太淡

**现象：** 护眼模式下，代码块内的语法高亮文字对比度低，难以阅读。

**原因：** 护眼主题的 `--code-block-bg` 色值（`#f0f5ed`）与文字色（`#2d3e30`）对比度不够；PaperMod 语法高亮使用 inline style，不受 CSS 变量控制。

**解决：**
- 加深代码块底色为 `#d4dfcf`
- 在护眼主题 CSS 中追加 `pre code { color: #1a2a1d }`

---

## 12. 主题切换 flash（PaperMod 脚本覆盖自定义主题）

**现象：** 页面加载时短暂闪一下浅色模式，然后才切换到护眼模式。

**原因：** PaperMod 的 `head.html` 中主题检测脚本只识别 `dark` / `light`，遇到 `eye-care` 会 fallback 到 `light`，覆盖了自定义设置。

**解决：**
- 在 `head.html` 之前插入脚本，第一时间设 `dataset.theme`
- 创建 `layouts/partials/extend_head.html`，在 PaperMod 脚本之后再次修正主题

---

## 13. 首页卡片子文章列表过于冗长

**现象：** 安卓应用分类卡片下显示了 5 个子分类的全部文章名，卡片撑得太大。

**原因：** 模板对含子 section 的分类也列出了 `.RegularPages`（所有后代文章）。

**解决：** 首页卡片只显示分类名 + 篇数 + 子分类数，不列文章名。

---

## 14. 主页标题前出现 ▶ 符号

**现象：** 首页「关于本站」「博客统计」等标题前面出现残留的 ▶ 三角符号。

**原因：** `baseof.html` 中旧版侧边栏的 `.section-title::before { content: '▶' }` 没有限定作用域，泄漏到全站所有同类名元素。当时侧边栏已改用 `section-title-wrapper`，旧样式未清理。

**解决：** 删除 `baseof.html` 中不再使用的 `.section-title` 伪元素样式块。

---

## 15. 侧边栏文章排序反转（04→01）

**现象：** 侧边栏 `1.Java基础` 下文章显示为 `04` `03` `02` `01` 降序排列。

**原因：** 遍历 `$directPages` 时，来源 `.Pages` 默认按 Hugo 内部顺序（通常是 date），未显式排序。

**解决：** 将 `.Pages` 改为 `.Pages.ByTitle`，使文章按标题字母升序（01, 02, 03...）。

---

## 16. 首页卡片图标匹配顺序错误

**现象：** 「安卓工具」「安卓系统」卡片图标显示 📱 而非 🔧 / ⚙️。

**原因：** 图标匹配使用 `if-else` 链，`in "安卓"` 匹配顺序靠前，所有含"安卓"的分类都匹配到 📱，后续 `in "工具"` 等规则被跳过。

**解决：** 将 `工具` `系统` `项目` 的匹配放到 `安卓` 前面，最后才 fallback 到 📱。
