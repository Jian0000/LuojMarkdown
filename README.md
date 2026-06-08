# Luoj's Markdown

基于 **Hugo + PaperMod** 的个人博客，托管在 **Cloudflare Pages**。

- 访问地址：<https://luojmarkdown.pages.dev/>

---

## 项目结构

```
LuojMarkdown/
├── layouts/                        # 自定义模板（覆盖 PaperMod 默认布局）
│   ├── index.html                  # 首页：卡片式分类导航
│   ├── _default/
│   │   └── baseof.html             # 全局布局：控制侧边栏显隐、自定义样式
│   └── partials/
│       └── sidebar.html            # 文章列表侧边栏（分层大纲）
├── content/
│   └── posts/                      # 所有文章
│       ├── first.md                # 根目录文章
│       ├── second.md               # 根目录文章
│       ├── test/                   # 分类：测试
│       │   ├── _index.md           # 分类页标题配置
│       │   ├── test1.md
│       │   ├── test2.md
│       │   └── test3.md
│       ├── test2/                  # 分类：测试2
│       │   ├── _index.md
│       │   ├── test1.md
│       │   ├── test2.md
│       │   └── test3.md
│       └── notes/                  # 分类：笔记
│           ├── _index.md
│           ├── note1.md
│           └── note2.md
├── themes/
│   └── PaperMod/                   # 主题（git submodule）
├── hugo.toml                       # Hugo 站点配置
├── .gitignore
└── CLAUDE.md                       # Claude Code 项目配置
```

---

## 功能改动记录

### 1. 左侧文章列表侧边栏

- **文件**：`layouts/_default/baseof.html`、`layouts/partials/sidebar.html`
- 文章页面左侧显示固定侧边栏，列出所有文章
- 侧边栏 `sticky` 定位，跟随滚动
- 当前文章高亮显示
- 手机端自动折叠到顶部

### 2. 分类大纲结构

- **目录**：`content/posts/test/`、`content/posts/notes/` 等
- 文章按文件夹分组，每个文件夹是一个分类
- 分类名由 `_index.md` 中的 `title` 字段定义
- URL 格式：`/posts/<分类>/<文章slug>/`

### 3. 分层侧边栏

- **文件**：`layouts/partials/sidebar.html`
- 根目录文章直接列出
- 分类下文章以可折叠分组展示（►/▼ 图标）
- 分类标题可点击跳转到该分类第一篇文章
- 默认展开所有分类

### 4. 首页卡片式导航

- **文件**：`layouts/index.html`
- 首页去除侧边栏，居中展示分类卡片
- 每个卡片显示：分类图标、名称、文章数量、首篇文章
- 点击卡片跳转到该分类的第一篇文章
- 根目录文章单独展示为一个卡片，列出所有直接链接

### 5. 配置优化

- **文件**：`hugo.toml`
- `baseURL` → 实际 Cloudflare 域名
- `locale` → `zh-cn`（中文）
- `title` → 自定义博客名

---

## 文章编写规范

### Frontmatter 格式

每篇 `.md` 文章必须以 `+++` 开头和结尾包裹元数据（TOML 格式）：

```markdown
+++
title = '文章标题'
date = '2026-06-05T12:00:00+08:00'
draft = false
+++

正文内容（Markdown 格式）...
```

### 注意事项

| 规则 | 说明 |
|------|------|
| `+++` 必须成对 | 开头一个 `+++`，结尾一个 `+++`，缺一不可 |
| `draft = false` | 必须设为 `false`，否则生产环境不会渲染 |
| `date` 用过去时间 | Hugo 将 `date` 作为发布时间，未来时间会被跳过 |
| 文件编码 | 保存为 **UTF-8 无 BOM**（VS Code 等现代编辑器默认如此） |
| 不要用 PowerShell `Set-Content` 创建文件 | 它会自动加 BOM 头，Hugo 可能识别异常 |

### 新建文章

直接在 `content/posts/` 下新建 `.md` 文件：

```bash
# 根目录文章
content/posts/我的文章.md

# 分类文章（需先在分类目录下创建 _index.md）
content/posts/笔记/笔记三.md
```

`git push` 后 Cloudflare Pages 自动构建部署。

### 新建分类

```bash
mkdir content/posts/新分类
echo '+++' > content/posts/新分类/_index.md
echo 'title = "新分类名称"' >> content/posts/新分类/_index.md
echo '+++' >> content/posts/新分类/_index.md
```

---

## 部署

1. 编辑文章，`git push` 到 `master` 分支
2. Cloudflare Pages 自动检测变更并构建
3. 1-2 分钟后刷新博客即可看到更新

## 本地预览

```bash
hugo server -D    # -D 包含草稿
```

访问 `http://localhost:1313/`

---

## 问题排查记录

### 问题 1：网站显示 Hugo 默认页面，没有自己写的文章

| 项目 | 内容 |
|------|------|
| **现象** | 浏览器打开博客，显示 Hugo 初始化的空页面，而非测试文章 |
| **原因** | 文章 frontmatter 中 `draft = true`，Hugo 生产构建默认跳过草稿 |
| **解决** | 将 `draft = true` 改为 `draft = false` |

### 问题 2：新建 `.md` 文章不渲染

| 项目 | 内容 |
|------|------|
| **现象** | 新建文章 push 后，Cloudflare 构建成功但页面不存在 |
| **原因** | 文章 frontmatter 缺少开头的 `+++`，Hugo 未能识别文章元数据 |
| **解决** | 确保每篇文章以 `+++` 开头、`+++` 结尾包裹 frontmatter |

### 问题 3：文章 `date` 设为未来时间导致不渲染

| 项目 | 内容 |
|------|------|
| **现象** | 文章文件格式正确，`draft = false`，但 Hugo 跳过不渲染 |
| **原因** | Hugo 将 `date` 作为 `publishDate`，如果时间还没到（哪怕只差 1 分钟），视为"未发布"而跳过 |
| **解决** | `date` 使用**过去时间**，不要设为未来。特殊情况可构建时加 `--buildFuture` |

### 问题 4：PowerShell 创建的文件带 BOM，Hugo 识别异常

| 项目 | 内容 |
|------|------|
| **现象** | 用 `Set-Content -Encoding utf8` 创建的文章不渲染 |
| **原因** | PowerShell `Set-Content -Encoding utf8` 自动在文件头部添加 UTF-8 BOM（`EF BB BF`），部分 Hugo 版本对此敏感 |
| **解决** | 用 VS Code 等编辑器创建文件，或用 `Write` 工具创建、保存为 UTF-8 无 BOM |

### 问题 5：Cloudflare Pages 修改不生效（最坑）

| 项目 | 内容 |
|------|------|
| **现象** | GitHub 代码已更新，本地构建正常，但线上页面无变化 |
| **根本原因** | 两个原因叠加：① PaperMod 主题是 **git submodule**，Cloudflare Pages 默认不拉取子模块，导致 `hugo` 构建失败；② 之前能成功是因为 `public/` 目录（构建产物）也提交到了仓库中，Cloudflare 直接用现成文件，根本没跑 Hugo |
| **解决** | 1. Cloudflare Pages → Settings → Build & Deploy → 勾选 **Include Git Submodules** 2. 将 `public/` 加入 `.gitignore`，不再提交构建产物到仓库 |

### 问题 6：侧边栏分类目录不显示子文章

| 项目 | 内容 |
|------|------|
| **现象** | 侧边栏看到分类名（测试、笔记），但展开后没有文章 |
| **原因** | 同问题 3 —— 子目录文章 `date` 设在了当前时间之后 |
| **解决** | 将所有文章 `date` 改为已过去的时间点 |

