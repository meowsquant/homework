# 配置 Quarto 网站部署到 GitHub Pages

本文档记录了将 `homework` 仓库中的 Quarto(`.qmd`)作业文档配置为一个网站并自动部署到 GitHub Pages 的完整操作步骤。

- **仓库**: `meowsquant/homework`
- **操作日期**: 2026-07-19
- **部署地址**: https://meowsquant.github.io/homework/

---

## 背景

仓库中包含四个 Quarto 文档(`task1.qmd` ~ `task4.qmd`),内容为量化交易入门课程作业(数据模拟、行情获取、趋势识别、均线策略回测)。目标是将其组织为一个带导航栏的网站,推送到 `main` 分支时自动构建并部署到 GitHub Pages。

## 前置条件

- [Quarto](https://quarto.org/) 已安装(本机版本 1.9.38)
- Python 3.x(本机使用 uv 管理的 3.14)
- 仓库已托管于 GitHub 并配置了 SSH 远程(`git@github.com:meowsquant/homework.git`)
- 代码依赖:`numpy`、`pandas`、`matplotlib`、`akshare`

---

## 操作步骤

### 1. 创建虚拟环境并安装依赖

由于本机 Python 由 uv 管理(externally managed),不能直接 `pip install`,需先创建虚拟环境:

```bash
/Users/sissiyang/.local/share/uv/python/cpython-3.14-macos-aarch64-none/bin/python -m venv .venv
.venv/bin/python -m pip install numpy pandas matplotlib akshare
```

Quarto 执行 Python 代码块还需要 Jupyter 内核和 jupyter-cache:

```bash
.venv/bin/python -m pip install jupyter pyyaml ipykernel jupyter-cache
.venv/bin/python -m ipykernel install --user --name python3 --display-name "Python 3"
```

### 2. 创建 Quarto 网站配置

在仓库根目录新建 `_quarto.yml`:

```yaml
project:
  type: website
  output-dir: _site
  render:
    - task1.qmd
    - task2.qmd
    - task3.qmd
    - task4.qmd
    - index.qmd
  resources:
    - "*.png"
    - "*.jpg"

execute:
  freeze: auto
  cache: true

website:
  title: "量化交易入门 — 作业"
  page-navigation: true
  navbar:
    background: primary
    left:
      - text: "Task 1 — 模拟数据涨跌盘"
        href: task1.html
      - text: "Task 2 — 股票数据获取"
        href: task2.html
      - text: "Task 3 — 趋势模拟与识别"
        href: task3.html
      - text: "Task 4 — 双均线策略回测"
        href: task4.html
    right:
      - icon: github
        href: https://github.com/meowsquant/homework
        aria-label: GitHub

lang: zh
```

**关键设计说明**:

- `freeze: auto` — 本地执行一次代码,结果缓存到 `_freeze/` 目录。CI 渲染时直接复用缓存,无需重新执行代码或访问 akshare API。这对依赖网络数据源(akshare)的文档尤其重要。
- `cache: true` — 启用 Jupyter 缓存(需安装 `jupyter-cache`)。

### 3. 创建首页 `index.qmd`

新建 `index.qmd` 作为网站入口,包含作业列表表格、运行环境说明和 GitHub 链接。

### 4. 创建 `.gitignore`

```gitignore
.Rhistory
.DS_Store
.venv/
__pycache__/
*.quarto_ipynb*
quarto_ipynb*
```

### 5. 清理旧的渲染产物

仓库中存在此前单独渲染 `.qmd` 为 HTML 时生成的附属文件(`task1_files/`、`task2_files/` 等),这些在网站模式下会导致冲突(Quarto 尝试将 `task1_files/index.html` 当作目录读取,报 `NotADirectory` 错误):

```bash
rm -rf *_files
```

同时,旧的 `task1.html` 等独立 HTML 文件也不再需要(网站模式输出到 `_site/`),一并删除。

### 6. 本地渲染生成 freeze 缓存

```bash
QUARTO_PYTHON=.venv/bin/python quarto render
```

这会执行所有代码块、生成图表,并将结果缓存到 `_freeze/` 目录。输出网站在 `_site/`。

### 7. 创建 GitHub Actions 工作流

新建 `.github/workflows/quarto-publish.yml`:

```yaml
on:
  push:
    branches: main
  workflow_dispatch:

name: Quarto Publish

permissions:
  contents: read
  pages: write
  id-token: write

jobs:
  build-deploy:
    runs-on: ubuntu-latest
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    steps:
      - name: Check out repository
        uses: actions/checkout@v4

      - name: Set up Quarto
        uses: quarto-dev/quarto-actions/setup@v2

      - name: Render site
        uses: quarto-dev/quarto-actions/render@v2

      - name: Upload artifact
        uses: actions/upload-pages-artifact@v3
        with:
          path: _site

      - name: Deploy to GitHub Pages
        id: deployment
        uses: actions/deploy-pages@v4
```

**说明**:由于使用了 `freeze: auto`,工作流中不需要安装 Python 依赖或 Jupyter — Quarto 渲染时会直接读取 `_freeze/` 中的缓存结果。

### 8. 提交并推送

```bash
git add _quarto.yml index.qmd .gitignore .github/workflows/quarto-publish.yml _freeze/
git add -A   # 同时暂存删除的旧渲染产物
git commit -m "配置 Quarto 网站并部署到 GitHub Pages"
git push origin main:main
```

### 9. 在 GitHub 仓库设置中开启 Pages

这是唯一需要手动操作的步骤:

1. 打开 https://github.com/meowsquant/homework/settings/pages
2. 在 **Build and deployment** 部分,将 **Source** 设置为 `GitHub Actions`(不是 "Deploy from a branch")
3. 保存后,工作流会自动触发

构建进度可在 https://github.com/meowsquant/homework/actions 查看。部署完成后访问 https://meowsquant.github.io/homework/ 。

---

## 遇到的问题与解决

| 问题 | 原因 | 解决方法 |
| --- | --- | --- |
| `pip install` 报 `externally-managed` 错误 | uv 管理的 Python 不允许直接安装包 | 创建 `.venv` 虚拟环境 |
| Quarto 渲染报 `No module named 'yaml'` | 虚拟环境缺少 jupyter 相关包 | 安装 `jupyter pyyaml ipykernel` |
| Quarto 渲染报 `jupyter-cache package is required` | 缓存执行依赖 jupyter-cache | 安装 `jupyter-cache` |
| Quarto 渲染报 `NotADirectory: task1_files/index.html` | 旧的单独渲染产物(`*_files/`)与网站模式冲突 | `rm -rf *_files` 清理旧产物 |
| `git push` 被拒绝(non-fast-forward) | 本地与远程分叉(远程有创建/删除 index.html 的提交) | `git pull origin main` 合并后再推送 |
| Pages 显示 README 而非 Quarto 站点 | GitHub 内置 `pages-build-deployment` 覆盖了 Quarto Publish 部署 | 安装 `gh` CLI,`gh run rerun` 重新触发 Quarto Publish;详见 [fix-pages-deployment.md](./fix-pages-deployment.md) |

---

## 文件清单

| 文件 | 说明 |
| --- | --- |
| `_quarto.yml` | Quarto 网站配置 |
| `index.qmd` | 网站首页 |
| `.gitignore` | 忽略虚拟环境、缓存等 |
| `.github/workflows/quarto-publish.yml` | GitHub Actions 部署工作流 |
| `_freeze/` | 本地执行缓存(提交到仓库,CI 复用) |

---

## 后续维护

修改 `.qmd` 文件后,需要本地重新渲染以更新 `_freeze/` 缓存,然后推送:

```bash
QUARTO_PYTHON=.venv/bin/python quarto render
git add _freeze/
git commit -m "更新内容"
git push origin main:main
```

推送后 GitHub Actions 会自动重新部署。
