---
layout: post
title: 'Node nvm & npm nrm 工具介绍'
date: 2019-04-21
author: 易风
cover: ''
tags: nvm nrm 包管理 镜像源管理
---

> 开篇词: 在开发小程序使用wepy框架中时出现过node版本依赖问题，由于packjson Node版本与本地版本不一致问题导致依赖包安装失败，最终使用nvm node版本管理工具解决 。


# Mac包管理工具 — brew

**brew是Mac一个包管理工具，类似于Ubuntu下的 apt，在mac系统上进行开发，brew可以很方便地进行安装/卸载/更新各种软件包，可以大大提升工作效率。**


###  安装 Homebrew

打开mac 终端 Terminal, 执行以下代码:
```code
/usr/bin/ruby -e "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/master/install)"
```

安装完成执行 brew -v 检测是否安装成功


# node包管理工具 — nvm

### 安装 nvm

```code
brew install nvm
```

查看版本是否安装成功
```code
nvm --version
```

nvm 查看可用的node版本
```code
nvm -ls
```

nvm 安装 node版本
```code
nvm install v10.15.3
```

nvm 切换node版本
```code
nvm use 10.15.3   
```

nvm 设置默认版本
```code
nvm use default 10.15.3   
```
***注意 nvm use 命令只是在当前终端窗口有效，相当临时版本, 需要设置默认版本还需使用nvm use version**


## Uninstall node

nvm删除node版本
```code
nvm uninstall 10.15.3   
```

# npm镜像管理工具 — nrm

### 全局安装 
```code
npm instal -g nrm
```

### 查看镜像源

```code
nrm ls

npm ---- https://registry.npmjs.org/
cnpm --- http://r.cnpmjs.org/
taobao - https://registry.npm.taobao.org/
nj ----- https://registry.nodejitsu.com/
npmMirror  https://skimdb.npmjs.com/registry/
edunpm - http://registry.enpmjs.org/
```


### 切换镜像

```code
nrm use cnpm

verb config Skipping project config: /Users/liyi/.npmrc. (matches userconfig)
Registry has been set to: http://r.cnpmjs.org/
```

### nrm命令

```code
nrm help // 查看帮助
nrm list // 查看所有镜像源
nrm use cnpm // 切换镜像
nrm  current // 查看当前镜像
```


