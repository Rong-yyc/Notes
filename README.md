# 学习笔记
## Rong
这里是小镇同学的学习笔记

## Pollyanna

这里是小女友的学习笔记

---

## Git 操作指南

### 常用命令

```bash
# 初始化仓库
git init

# 添加文件到暂存区
git add .

# 提交更改
git commit -m "your commit message"

# 查看状态
git status

# 推送到远程仓库
git push
```

### 同时推送到 Gitee 和 GitHub

1.  **添加多个远程仓库地址**

    假设你已经有了一个名为 `origin` 的远程地址（通常是 GitHub），现在需要为 Gitee 添加一个新的远程地址。

    ```bash
    # 查看当前的远程仓库地址
    git remote -v
    
    # 添加 Gitee 远程仓库
    git remote add gitee git@gitee.com:your_username/your_repo.git
    # 添加 GitHub 远程仓库
    git remote add github git@github.com:your_username/your_repo.git
    ```

2.  **设置别名实现一键推送**

    为了方便，你可以设置一个 Git 别名，用一条命令同时推送到两个远程仓库。

    ```bash
    # 设置别名 pushall
    git config --global alias.pushall '!git push gitee && git push github'
    ```

    现在，你只需要运行以下命令，就可以将所有分支的修改同时推送到 GitHub 和 Gitee：

    ```bash
    git pushall
    ```
