# 把仓库推到 GitHub 的步骤

## 1. 在 GitHub 上新建仓库

1. 打开 https://github.com/new
2. **Repository name** 填一个名字，例如 `darwin-hpc-skill`
3. 选 **Public**
4. **不要**勾选 "Add a README file"（保持空仓库）
5. 点 **Create repository**

## 2. 在本地初始化并推送

在终端里进入本目录后执行：

```bash
cd "/Users/minqiu/Desktop/Huamiao's Wiki/darwin-hpc-skill"

# 初始化 Git
git init

# 添加所有文件
git add README.md darwin/

# 第一次提交
git commit -m "Initial commit: DARWIN HPC skill + README"

# 添加你的 GitHub 仓库地址（把 YOUR_USERNAME 和 REPO_NAME 换成你的）
git remote add origin https://github.com/YOUR_USERNAME/REPO_NAME.git

# 推送到 GitHub（主分支用 main）
git branch -M main
git push -u origin main
```

若 GitHub 提示分支是 `master`，把最后一行改成：`git push -u origin master`

## 说明

- 仓库里的 `darwin/SKILL.md` 已改成占位符版本（`your_username`、`your_workgroup`），没有你的真实账号信息，可以放心公开。
- 你本地的 `Huamiao's Wiki/darwin/SKILL.md` 仍保留你的真实配置，不会被覆盖。
