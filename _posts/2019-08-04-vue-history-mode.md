---
layout: post
title: 'Vue Router History 模式渲染配置'
date: 2018-08-04
author: liyi
tags: Vue history vue-router
---


>前言：写这篇文章，源于最近自己开发vue项目，项目使用Vue-router, 在配置 HTML5 History 模式下遇到一些问题，写篇文章简单总结一下，如有描述有不对的地方欢迎各位大神提issues 


## Vue history 页面渲染空白前端配置:

1、修改vue-cli config/index.js 配置文件 找到如下文件

静态文件修改为相对路径 ： `assetsPublicPath: '/'` 修改为 `assetsPublicPath: './'` 
 ```code
 build: {
        // Template for index.html
        index: path.resolve(__dirname, '../dist/index.html'),

        // Paths
        assetsRoot: path.resolve(__dirname, '../dist'),
        assetsSubDirectory: 'static',
        //assetsPublicPath: '/',
        assetsPublicPath: './',
        
        ...
}
 ```

2、修改路由base值

```code
const router = new VueRouter({
    mode: 'history',
    #base: '/',    
    base: '/build/',   
    ...
})
```
上述代码片段： base 默认 "/" 为服务器根目录,  假如开发项目Vue Span 服务器根目录在  `public/`, 生产文件在 '/public/build/' 下面, 
此时应该设置为 `base: '/build/' `

**注意：如果Web server 根目录不在dist下(或者其他目录下面), base：vaule 需要修改为基于服务器根目录下  `/xxx/` 来指定 vue Spa入口文件**


## Vue Router 服务端配置 案例:

#### Nginx配置分为两种情况:

1、发布文件在服务器根目录如下所示：
``` code
location / {
    root  /；
    index index.html;
    try_files $uri $uri/ /index.html;
}
.....
``` 

2、发布文件在不在服务器根目录，举个例子，比如服务器根目录在 `/public/` ,项目发布文件在 build 目录下

```	html
# 服务器目录结构

├── public/ # 服务器根目录
|   |
|   └── build/ # 生产环境发布文件目录
└── ****/ 
    
    
# Nginx server config:

root /www/public/;

location /build/ {
    index index.html index.htm;

    try_files $uri $uri/ /build/index.html;
}
```

#### Apache:
Apache服务器配置 Vue Router History模式，需要在生产文件目录(和vue index.html发布入口文件在同一个目录) 新建一个 `.htaccess` 文件，配置如下图,

##### 1、首先需要配置Apache httpd.conf 
* 开启rewrite_module功能， `LoadModule rewrite_module libexec/apache2/mod_rewrite.so` 去掉注释
* `AllowOverride: All`是为了使apache支持 `.hatccess`,

```code
DocumentRoot "/www/public/"（设置apache默认指向目录）
<Directory "/www/public/">
    Options Indexes FollowSymLinks Multiviews
    MultiviewsMatch Any
    AllowOverride All
    Require all granted
</Directory>
```

##### 2、**.hatccess 配置文件**

```code
 <IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /index.html [L]
</IfModule>
```

配置参数:  
`RewriteBase`：默认在 "/" 根目录查找  
`RewriteRule ^index\.html$ - [L]`: 匹配 index.html 文件

#### 注意：
如果生产文件不在根目录, 而是在 '/public/build/index.html' 那么:
`RewriteBase: /` 修改为为 `RewriteBase: /build/`  
`RewriteRule . /index.html [L]` 修改为： `RewriteRule . /build/index.html [L]` 如下所示

```code
 <IfModule mod_rewrite.c>
  RewriteEngine On
  RewriteBase /build/
  RewriteRule ^index\.html$ - [L]
  RewriteCond %{REQUEST_FILENAME} !-f
  RewriteCond %{REQUEST_FILENAME} !-d
  RewriteRule . /build/index.html [L]
</IfModule>
```


### 备注：
以上举例服务器根目录都是基于 `/public` 演示, 开发中依据实际情况替换对应目录即可





















