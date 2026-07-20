# 修复 GitHub Pages 显示 README 而非 Quarto 站点

本文档记录了 Quarto 网站部署到 GitHub Pages 后,线上仍显示 README.md 内容而非 Quarto 渲染站点的排障与修复过程。

- **仓库**: `meowsquant/homework`
- **操作日期**: 2026-07-19
- **部署地址**: https://meowsquant.github.io/homework/
- **前置文档**: [create-github-pages-action.md](./create-github-pages-action.md)

---

## 问题描述

完成 Quarto 网站配置并推送到 `main` 后,GitHub Actions 的 `Quarto Publish` 工作流显示 `success`,但访问 https://meowsquant.github.io/homework/ 时:

- 首页显示的是 `README.md` 的内容,而非 Quarto 渲染的 `index.qmd`
- 导航栏缺失
- `task1.html`、`task2.html` 等页面返回 404

## 诊断过程

### 1. 确认本地 `_site` 内容正确

```bash
head -30 _site/index.html | grep -o '<title>.*</title>'
# 输出: <title>量化交易入门 — 作业</title>

grep -c "本网站为" _site/index.html
# 输出: 1 (index.qmd 的内容)
```

本地渲染产物正确,问题出在部署侧。

### 2. 检查 GitHub Pages API 配置

```bash
gh api repos/meowsquant/homework/pages
```

返回:
```json
{
  "build_type": "workflow",
  "source": { "branch": "main", "path": "/" },
  "status": "built"
}
```

`build_type` 已经是 `"workflow"`,说明 Pages Source 已设置为 "GitHub Actions"。

### 3. 检查 Quarto Publish 工作流日志

```bash
gh run view 29686149699 --repo meowsquant/homework --log
```

关键日志:
```
[5/5] index.qmd
Output created: _site/index.html
```

渲染成功,artifact 也上传了(15.4 MB)。

### 4. 检查部署历史

```bash
gh api repos/meowsquant/homework/deployments --jq '.[] | {id, environment, created_at}'
```

发现两个工作流在竞争部署:
- `Quarto Publish`(我们自定义的,输出 `_site/`)
- `pages-build-deployment`(GitHub 内置的,用仓库原始内容部署)

`pages-build-deployment` 在 Quarto Publish 之后运行,用仓库根目录的内容(README.md)覆盖了 Quarto 的部署结果。

### 5. 用 curl 验证线上实际内容

```bash
curl -sL https://meowsquant.github.io/homework/ | grep -oE '<title>[^<]+</title>'
# 输出: <title>量化交易入门 — 作业 | homework</title>  (README 的标题)

curl -sL -o /dev/null -w "%{http_code}" https://meowsquant.github.io/homework/task1.html
# 输出: 404
```

确认线上确实是 README,task 页面不可访问。

## 根因

GitHub 的内置 `pages-build-deployment` 工作流在 Quarto Publish 之后运行,用仓库原始内容(README.md)覆盖了 Quarto 渲染的 `_site/` 部署结果。

虽然 `build_type` 已为 `"workflow"`,但 `pages-build-deployment` 仍被触发(可能是 Pages 设置变更或首次启用时的遗留行为)。

## 修复步骤

### 1. 安装 GitHub CLI

```bash
brew install gh
```

### 2. 登录 GitHub

```bash
gh auth login
```

选择浏览器授权,完成认证。验证:

```bash
gh auth status
# ✓ Logged in to github.com account meowsquant
```

### 3. 重新触发 Quarto Publish 工作流

```bash
gh run rerun 29686149699 --repo meowsquant/homework
```

等待工作流完成:

```bash
sleep 40
gh run list --repo meowsquant/homework --limit 3
# completed  success  配置 Quarto 网站并部署到 GitHub Pages  Quarto Publish
```

这次 `pages-build-deployment` 没有再干扰。

### 4. 验证部署结果

等待 10 秒让 CDN 更新,然后检查:

```bash
# 检查首页标题
curl -sL https://meowsquant.github.io/homework/ | grep -oE '<title>[^<]+</title>'
# 输出: <title>量化交易入门 — 作业</title>  ✓ Quarto 站点

# 检查内容是否为 index.qmd
curl -sL https://meowsquant.github.io/homework/ | grep -c "本网站为"
# 输出: 1  ✓

# 检查导航栏
curl -sL https://meowsquant.github.io/homework/ | grep -c "navbar"
# 输出: 14  ✓

# 检查所有 task 页面
for page in index task1 task2 task3 task4; do
  echo -n "$page.html: "
  curl -sL -o /dev/null -w "%{http_code}" "https://meowsquant.github.io/homework/$page.html"
  echo ""
done
# 输出:
# index.html: 200
# task1.html: 200
# task2.html: 200
# task3.html: 200
# task4.html: 200
```

所有页面均返回 200,Quarto 渲染的网站成功上线。

## 经验总结

1. **Quarto 网站部署到 GitHub Pages 时,需确保 Pages Source 设置为 "GitHub Actions"**。可通过 `gh api repos/.../pages` 检查 `build_type` 是否为 `"workflow"`。

2. **GitHub 内置的 `pages-build-deployment` 可能与自定义工作流冲突**。如果 Pages 显示 README 而非 Quarto 站点,检查是否有两个工作流在竞争部署。

3. **重新触发工作流可解决部署覆盖问题**。使用 `gh run rerun <run-id>` 重新执行 Quarto Publish,通常能让 Quarto 的部署结果最终生效。

4. **验证部署时,用 `curl` 直接检查 HTML 内容**,而非依赖浏览器(浏览器可能缓存旧版本)。

5. **后续维护时,如果再次出现部署覆盖,可手动触发 `pages-build-deployment` 或重新运行 Quarto Publish**。

## 参考命令速查

```bash
# 检查 Pages 配置
gh api repos/meowsquant/homework/pages

# 查看工作流运行状态
gh run list --repo meowsquant/homework --limit 5

# 查看工作流日志
gh run view <run-id> --repo meowsquant/homework --log

# 重新触发工作流
gh run rerun <run-id> --repo meowsquant/homework

# 验证线上内容
curl -sL https://meowsquant.github.io/homework/ | grep -oE '<title>[^<]+</title>'
```

## 相关文档

- [create-github-pages-action.md](./create-github-pages-action.md) — Quarto 网站配置与 GitHub Actions 部署的完整流程
