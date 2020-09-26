---
title: 利用Github Pages和Hexo搭建静态博客
date: 2016-02-14 10:10:10
tags: [Github, Hexo]
categories: Hexo
---
# 搭建流程
## 创建仓库
在[GitHub](https://github.com)上创建仓库，名为`YourUserName.github.io`
## 新建hexo分支
在刚刚创建的仓库中新建一个`hexo`分支，并在仓库的`setting`中将其设为默认分支，
这样做的目的是将hexo的原始文件和生成的静态文件分开放置
<!-- more -->
## 建立本地仓库
``` bash
$ git clone git@github.com:YourUserName/YourUserName.github.io
```
## 在本地初始化博客
进入仓库文件夹
```bash
$ cd YourUserName.github.io
```
依次执行如下代码
```bash
$ npm install hexo
$ hexo init
$ npm install
$ npm install hexo-deployer-git --save
```
## 修改_config.yml
将`_config.yml`中的`depoly`字段修改为如下所示：
```bash
deploy:
  type: git
  repository: git@github.com:YourUserName/YourUserName.github.io.git
  branch: master
```
## 将博客同步至Github
依次执行如下代码
```bash
$ git add --all
$ git commit -m "blabla"
$ git push origin hexo
```
**可能遇到的问题**
如果提示
``fatal: Not a git repository (or any of the parent directories): .git``
可通过如下方式解决
```bash
$ git clone --no-checkout git@github.com:YourUserName/YourUserName.github.io.git tmp
$ mv tmp/.git .
$ rmdir tmp
$ git reset --hard HEAD
```
## 将博客部署至Github Pages
```bash
hexo g -d
```
---
# 更新博客
## 一般的流程
```bash
$ git add --all
$ git commit -m "blabla"
$ git push origin hexo
$ hexo g -d
```
## 偷懒的做法
每次都敲上面那些太麻烦惹……写成脚本好啦
```bash
$ nano hexo_git_deploy.sh
```
将这些代码粘贴进去
```bash
#!/bin/bash

git add --all
git commit -m "blabla"
git push origin hexo
hexo g -d
```
按`ctrl+X`退出编辑状态，按`y`确定，按`Enter`保存
将其设置为可执行
```bash
$ chmod +x hexo_git_deploy.sh
```
以后每次都只要执行这个就好啦
```bash
$ ./hexo_git_deploy.sh
```