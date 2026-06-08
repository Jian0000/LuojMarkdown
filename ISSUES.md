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
