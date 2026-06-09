# Luoj's Markdown

基于 **Hugo + PaperMod** 的个人博客，托管在 **Cloudflare Pages**。

- 访问地址：<https://luojmarkdown.pages.dev/>

---

## 项目结构

```
LuojMarkdown/
├── layouts/                        # 自定义模板（覆盖 PaperMod 默认布局）
│   ├── index.html                  # 首页：分层卡片式导航
│   ├── _default/
│   │   └── baseof.html             # 全局布局：顶栏/侧边栏/大纲/搜索/主题
│   └── partials/
│       ├── sidebar.html            # 文章列表侧边栏（三层嵌套）
│       ├── toc_sidebar.html        # 文章右侧目录大纲
│       └── extend_head.html        # 默认护眼模式修正
├── content/
│   ├── _index.md                   # 首页内容（个人信息/关于/联系）
│   └── posts/                      # 所有文章（按分类分目录）
│       ├── 安卓工具/               # ADB相关、实用命令
│       ├── 安卓应用/               # 含子分类 1-5
│       │   ├── 1.Java基础/
│       │   ├── 2.Kotlin基础/
│       │   ├── 3.Android基础/
│       │   ├── 4.Android核心组件/
│       │   └── 5.Android进阶/
│       ├── 安卓系统/
│       └── 项目问题/
├── static/
│   └── pic/                        # 静态资源（头像、图标）
├── themes/
│   └── PaperMod/                   # 主题（git submodule）
├── hugo.toml                       # Hugo 站点配置
├── README.md
├── ISSUES.md
├── .gitignore
└── CLAUDE.md
```

---

## 功能清单

### 集成顶栏（sticky 固定）

- **文件**：`layouts/_default/baseof.html`
- Logo + 主题选择 + 全局搜索 + GitHub 图标，合为一条固定栏
- 滚动时始终悬浮在页面最上方

### 全局搜索

- **文件**：`layouts/_default/baseof.html`
- 所有页面顶部居中搜索栏，`Ctrl+K` 快速聚焦
- 搜索文章标题 + 正文，显示匹配片段并高亮关键词
- 键盘 ↑↓ 导航，Enter 跳转，Esc 关闭

### 主题选择器（👕）

- **文件**：`layouts/_default/baseof.html`
- 三主题：☀️ 浅色 / 🌙 深色 / 🌿 护眼
- 默认进入护眼模式
- 选择保存到 localStorage，下次自动恢复

### 左侧文章列表侧边栏（三层嵌套）

- **文件**：`layouts/partials/sidebar.html`
- 支持三层嵌套分类（一级分类 → 子分类 → 文章）
- 折叠/展开，点击分类标题跳转首篇文章
- 滚动位置记忆（sessionStorage）
- 当前文章高亮

### 右侧文章目录大纲

- **文件**：`layouts/partials/toc_sidebar.html`
- 自动提取 h1-h4 标题生成层级大纲
- IntersectionObserver 滚动高亮当前标题
- 点击平滑跳转，窗口 < 1200px 自动隐藏

### 首页分层布局

- **文件**：`layouts/index.html`、`content/_index.md`
- 个人信息卡片 → 关于本站 → 博客统计 → 内容导航 → 联系我
- 分类卡片简化，只显示名称+篇数，点击跳转第一篇
- 统计自动计算（文章数/字数/分类数）
- 内容全部由 `content/_index.md` 驱动

### 多层分类支持

- 目录命名：`1.Java基础` `2.Kotlin基础`（数字.名称）
- 子目录需 `_index.md` 定义分类标题
- 顶级目录（安卓应用/安卓工具等）保持不变

---

## 文章编写规范

### Frontmatter 格式

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
| `+++` 必须成对 | 开头末尾缺一不可 |
| `draft = false` | 否则生产环境不渲染 |
| `date` 用过去时间 | 未来时间会被跳过 |
| UTF-8 无 BOM | VS Code 等编辑器默认即可 |
| 不要用 PowerShell `Set-Content` | 会自动加 BOM |

### 新建分类

```bash
mkdir content/posts/新分类
echo '+++' > content/posts/新分类/_index.md
echo "title = '新分类名称'" >> content/posts/新分类/_index.md
echo '+++' >> content/posts/新分类/_index.md
```

---

## 部署

1. 编辑文章，`git push` 到 `master` 分支
2. Cloudflare Pages 自动构建部署
3. 1-2 分钟后刷新

## 本地预览

```bash
hugo server -D
```

访问 `http://localhost:1313/`
