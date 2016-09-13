---
layout: post
title: "git config 一种推荐方案"
comments: true
description: "展示我常用的 git config，也算一种推荐的方案吧"
keywords: "git, cofig"
category: "NOTE"
tag: "storm"
---

* git config --global           # 修改当前用户的配置
* sudo git config --system      # 修改系统全局的配置

###### git config --list
```bash
user.email=mtide@xxx.com
user.name=mtide
core.ignorecase=false
core.autocrlf=input
core.filemode=false
core.safecrlf=true
core.editor=vim
core.repositoryformatversion=0
core.bare=false
core.logallrefupdates=true
core.precomposeunicode=true
push.default=simple
alias.lg=log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
pull.rebase=true
```

###### git config 解析
```bash
user.email=mtide@xxx.com
user.name=mtide
core.ignorecase=false            # 不许忽略文件名大小写
core.autocrlf=input              # 换行模式为 input，即提交时转换为LF，检出时不转换
core.filemode=false              # 不检查文件权限
core.safecrlf=true               # 拒绝提交包含混合换行符的文件
core.editor=vim
core.repositoryformatversion=0   # Internal variable identifying the repository format and layout version
core.bare=false                  # 默认不创建裸仓库
core.logallrefupdates=true       # log 所有 ref 的更新
core.precomposeunicode=true      # Mac专用选项，开启以便文件名兼容其他系统
push.default=simple                    # 只推送本地当前分支，且与上游分支名字一致
alias.lg=log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit
pull.rebase=true                 # 强制开启 rebase 模式
```

###### git config 配置样例
```bash
git config --global user.email “mtide@xxx.com"
git config --global user.name=mtide
sudo git config --system core.ignorecase false
sudo git config --system core.autocrlf input
sudo git config --system core.filemode false
sudo git config --system core.safecrlf true
sudo git config --system core.editor vim
sudo git config --system core.repositoryformatversion 0
sudo git config --system core.bare false
sudo git config --system core.logallrefupdates true
sudo git config --system core.precomposeunicode true
sudo git config --system push.default simple
sudo git config --system alias.lg "log --color --graph --pretty=format:'%Cred%h%Creset -%C(yellow)%d%Creset %s %Cgreen(%cr) %C(bold blue)<%an>%Creset' --abbrev-commit"
sudo git config --system pull.rebase true
```

