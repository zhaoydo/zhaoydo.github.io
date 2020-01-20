---
title: git常用命令
categories:
- git
date: 2018-04-12 17:15:00
tags:
- git
---
# git常用命令  
参考：  
[常用Git命令清单](http://www.ruanyifeng.com/blog/2015/12/git-cheat-sheet.html)  
[Git远程操作详解](http://www.ruanyifeng.com/blog/2014/06/git_remote.html)  
![](http://www.ruanyifeng.com/blogimg/asset/2014/bg2014061202.jpg)
## git的几个区  
* Workspace:工作区
* Stage:暂存区
* Repository:仓库区
* Remote：远程仓库  
<!--more-->
## 常用操作  
1. 克隆项目到本地，并切换到需要修改的分支
2. 修改代码后提交到本地仓库，这里建议在git图形客户端操作（TortoiseGIT/SourceTree），方便查看修改的内容（替代了git diff/add/commit）
3. 拉取远端代码合并到本地分支，可能会有冲突，后面说如何解决
4. 合并之后push到远程仓库
```
# 从remote克隆项目到本地
git clone url
# 切换到master分支
git checkout master 
# 查看本地工作区代码与暂存区差异
git diff
# 将本地工作区代码提交到暂存区
git add .
# 提交到本地仓库区
git add -m 修复bugxxx
# 从远端master分支拉取代码并合并到本地分支
git pull origin master
# 从远端master分支拉取代码并合并到本地分支(rebase方式)
git pull --rebase origin master
# 提交到远程仓库
git push origin master
# 下班。。。
```
## 解决冲突  
git pull 实际有2个操作，1、拉取远端分支，2、合并到本地分支  
相当于：  
```
git fetch origin master
git merge origin/master
```
在merge时候可能会有冲突，推荐用TortoiseGIT客户端解决：
* 项目根目录右键:Git Commit->... 
* 双击红色的文件，解决冲突后关闭，点击commit按钮保存到本地仓库
* 操作完成，可以push到远程仓库了  

对于**rebase**方式pull的代码，解决文件冲突后不用点击commit按钮，命令行直接执行:git rebase --continue来完成  