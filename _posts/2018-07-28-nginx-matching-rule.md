---
layout: post
title: '理解Nginx服务器及其路径匹配算法'
date: 2018-07-28
author: lijing
# cover: 'http://on2171g4d.bkt.clouddn.com/jekyll-banner.png'
tags: nginx
---


>Nginx是世界上最流行的Web服务器之一。它能够很好的处理高并发的场景，并且很容易配置成一个Web服务器，或者邮件服务器，或反向代理服务器。这篇文章中我们将讨论Nginx处理请求过程中的细节。把你从靠猜来设计和配置Nginx的困境中解救出来。

### Nginx 模块化配置：

Nginx将具有不同功能的配置划分成不同的模块，并且按照不同的层次将它们组织在一起。当收到请求时，Nginx会使用一套处理逻辑来决定使用哪个配置块来处理这个请求。我们的文章将详细讨论这个处理逻辑。  我们主要会讨论server和location这两种配置模块。  
    
模块server是Nginx用来定义虚拟服务器的配置模块，以实现特定的功能。服务器管理员通常会在Nginx中配置多个server模块，通过不同的域名（domain）,端口（port）和IP地址来决定使用哪个server模块来处理请求。  

模块location是用于server模块内部的模块，它用来定义这个server模块如何处理来自不同来源和URI的请求。管理员可以随意设定URI的匹配规则，这是一个非常灵活的模块。

### Nginx是如何决定使用哪个Server模块来处理请求的？

由于Nginx允许使用多个server模块来配置多个虚拟服务器，它提供了一套流程来决定使用哪个虚拟机来处理请求。
Nginx是通过一个决策系统来实现这个匹配过程的。server中与此相关的最关键的两个指令是listen和server_name.


#### 1、通过解析listen来匹配请求

首先，Nginx先获取请求的IP地址和端口，与所有的server模块中的listen指令进行匹配，找到所有可能会处理这个请求的模块。
指令listen一般用来定义这个server模块要响应的请求的的IP地址和端口。若server模块中如果没有定义listen指令，默认会定义为0.0.0.0:80（若Nginx是非root用户启动则定义为0.0.0.0:8080）。它允许这类server模块响应发送到本机任何IP上端口为80的请求，默认值在匹配server模块的算法中优先级很低。
指令listen允许的值如下：

```text
* IP地址+端口。
* 一个单独的IP地址，默认监听的端口为80
* 一个单独的端口号，会监听所有IP地址
* 一个指向Unix socket服务的路径（这个配置一般用于不同服务器之间发送请求）
```

当决定要用哪个server模块来处理请求时，Nginx会遵循以下原则：
Nginx会自动补全不完整的listen配置，例如：

```text
* 没配置listen时会使用默认值 0.0.0.0:80
* 只配置了IP如 111.111.111.111 时，会自动转化为 111.111.111.111:80
* 只配了端口如8888，自动补全为0.0.0.0:8888
```

Nginx接着会找出IP地址和端口与接到的请求匹配的server模块，如果有设定了IP地址的server模块，则所有没有设定IP地址或设定为0.0.0.0的server模块将 不会被选中 。端口号必须被精确匹配。

如果此时只选中了一个server模块，则将会使用这个模块来处理请求。如果有多个，则Nginx开始匹配这些模块中的server_name指令。

要特别注意的是只有当Nginx匹配到的listen配置的权重相同时，才会去匹配server_name指令，例如：example.com指向192.168.1.10的80端口，则example.com将会由下面的第一个模块来处理。

```code
server {
    listen 192.168.1.10;
    . . .
}

server {
    listen 80;
    server_name example.com;
    . . .
}
```
在匹配到多个相同权重的server模块的场景中，下一步Nginx是检查server_name指令


#### 2、通过解析server_name指令来匹配模块

为了从匹配到的多个server模块找到最终处理请求的模块，Nginx从请求头获取“Host”字段，这个字段中传输的是客户端真正想请求的服务域名或IP。
Nginx尝试从备选的模块中通过server_name指令来匹配是，将会按照下面的规则：

* 如果有server模块中的server_name可以精确匹配请求头中的“Host”字段，则会使用这个模块。如果匹配到多个模块，将会使用第一个模块来处理请求。
* 如果没有精确匹配的模块，将采用后缀匹配的模式（即查找server_name的配置以*开头）。如果匹配到多个模块，则遵守最长后缀匹配规则。
* 如果使用后缀匹配没有找到模块，将采用前缀匹配的模式（即查找server_name的配置以*结尾）。如果匹配到多个模块，则遵守最长前缀缀匹配规则。
* 如果仍然没有找到模块，Nginx开始查找使用正则定义server_name的模块（在表达式前面会有~符号）。第一个表达式能够匹配Host字段的模块将被用来处理请求。
* 如果以上算法都没有匹配到模块，将采用配置了default_server的模块。单个IP/端口组合只能指定一个default_server声明。


**如果有server模块中的server_name可以精确匹配请求头中的“Host”字段，则会使用这个模块。如果匹配到多个模块，将会使用第一个模块来处理请求。**
在下面示例中，请求头中的Host字段设置为host1.example.com，将匹配第二个server模块：

```text
server {
    listen 80;
    server_name *.example.com;
    . . .
}

server {
    listen 80;
    server_name host1.example.com;
    . . .
}
```

**如果没有精确匹配的模块，将采用后缀匹配的模式（即查找server_name的配置以*开头）。如果匹配到多个模块，则遵守最长后缀匹配规则。**
在这个示例中，如果请求头Host字段为www.example.org，将会匹配到第二个模块：

```text
server {
    listen 80;
    server_name www.example.*;
    . . .
}

server {
    listen 80;
    server_name *.example.org;
    . . .
}

server {
    listen 80;
    server_name *.org;
    . . .
}
```

**如果使用后缀匹配没有找到模块，将采用前缀匹配的模式（即查找server_name的配置以*结尾）。如果匹配到多个模块，则遵守最长前缀缀匹配规则。**
在这个示例中，如果请求头Host字段为www.example.com，将会匹配到第三个模块：

```text
    server {
    listen 80;
    server_name host1.example.com;
    . . .
}

server {
    listen 80;
    server_name example.com;
    . . .
}

server {
    listen 80;
    server_name www.example.*;
    . . .
}
```

**如果仍然没有找到模块，Nginx开始查找使用正则定义server_name的模块（在表达式前面会有~符号）。第一个表达式能够匹配Host字段的模块将被用来处理请求。**
在这个示例中，如果请求头Host字段为www.example.com，将会匹配到第二个模块：

```text
server {
    listen 80;
    server_name example.com;
    . . .
}

server {
    listen 80;
    server_name ~^(www|host1).*\.example\.com$;
    . . .
}

server {
    listen 80;
    server_name ~^(subdomain|set|www|host1).*\.example\.com$;
    . . .
}
```
如果以上步骤都没有匹配到任何模块，则请求将由该IP/端口中配置了default_server的模块处理。


### 匹配location模块

在讲解Nginx如何决定使用哪个location模块前，让我们先看一下location模块定义时的相关语法。location模块使用在server模块或其它location模块中，用来指定处理URI的方式。（URI指的是请求url中域名和端口之后的部分）

我们定义`location`模块时会采用以下方式：

```text

location optional_modifier location_match {
    . . .
}
```

在上面的模板中，location_match定义了Nginx匹配URI的内容。modifier的存在于否，决定了Nginx匹配location模块的模式。以下列出不同modifier的定义方式对location的影响：

* (none):如果没有声明modifier，则location采用前缀匹配。这种情况下location模块中的定义将去匹配URI的开头部分。
* =:如果使用等号，这个模块将精确匹配请求的URI。
* ~:对大小写敏感的正则模式。
* ~*:对大小写不敏感的正则模式。
* ^~:如果这个模块已经在非正则的模块定义中被匹配到，则不会再去匹配使用了正则的模块。

**译者注：** 之前经常搞不清楚为啥老是跑到正则模式的模块中，用^~可以解决。

##### 示例（演示location模块的语法）
1、前缀匹配模式（未定义modifier）下，下面的模块将用来处理URI为/site, /site/page1/index.html,或/site/index.html的请求：

```text
location /site {
    . . .
}
```

2、精确匹配模式（使用=）中，下面的模块将用来处理URI为/page1的请求，而不响应URI为/page1/index.html的请求。特别注意，由于请求会自动补全为使用index页面（译者注：一般为index.html），会产生一次内部跳转，这时请求将会由另一个location模块来处理。

**译者注：** 即先进入下面这个模块，再进入到能处理/page1/index.html的location模块。
```text
location = /page1 {
    . . .
}
```

3、大小写敏感的正则模式（使用~），下面的的模块将可以处理URI为/tortoise.jpg的请求，但不会响应/FLOWER.PNG这个请求。

```text
location ~ \.(jpe?g|png|gif|ico)$ {
    . . .
}
```


4、大小写不敏感的正则模式（使用~*），下面的的模块将可以处理URI为/tortoise.jpg的请求，也会响应/FLOWER.PNG这个请求。

```text
location ~* \.(jpe?g|png|gif|ico)$ {
    . . .
}
```

5、下面的模块如果被匹配为最佳非正则模块，则不会再去匹配其它正则定义的模块，例如，下面的模块将处理URI为/costumes/ninja.html的请求。

```text
location ^~ /costumes {
    . . .
}
```
综上所述，modifier定义了location模块如何被解析。然后这并没有说清楚Nginx是把请求打到某个具体的location模块上的。下面我们将讲一下这部分。


### Nginx是如何决定使用哪个location模块来处理请求

Nginx选择location模块的算法与匹配server时的算法类似。也是通过一个流程来为每个请求匹配location模块。如果想精准且可靠的配置Nginx服务器，对这个流程的理解是非常必要的。
要时刻谨记我们在上面说的location模块声明的语法，Nginx会把请求URI与所有的location模块做匹配。它遵循下面这个流程：

* 首先，Nginx查找精准匹配的模块。如果一个使用=的模块被匹配到，则会使用这个模块来处理请求。
* 如果没有精确匹配（使用=）的模块被匹配到，Nginx会匹配前缀匹配模式的模块，将会按照最长前缀匹配原则为这个请求匹配到location模块，接下来的动作可能有下面几种：
    * 如果最长前缀匹配到的是使用了~^的模块，则Nginx会停止匹配其它模块，并且立即使用这个模块来处理请求。
    * 如果没有使用^~，则Nginx会先暂保存这个匹配的结果，然后进行后面的匹配行为。
* Nginx接下来会采匹配使用了正则的模块（包含大小写敏感和不敏感）。Nginx会按顺序使用正则去匹配URI，第一个可以匹配的模块将用来处理这个请求。
* 如果在匹配正则表达式的模块这步没有匹配到任何模块，且之前全用前缀匹配时匹配到了模块，则使用之前匹配到模块来处理请求。


很重要的一点是，Nginx会优先使用正则表达式匹配到的模块来提供服务，而不是前缀匹配到的。但是在分析流程中又是先进行前缀匹配，并且允许管理员用^~或=来阻止进行正则表达式模块的匹配。

还有一点很重要，尽管前缀匹配采用的是最长匹配原则，但在进行正则匹配时，将使用第一个被匹配的location模块。这意味着，如果正则表达式模块的顺序不同，将会产生不同的效果。

**译者注:** server_name的匹配也是这个原则。


### 什么时候会从一个location模块跳到另一个location模块？

一般来讲，当匹配到一个location模块时，这个请求将一直在这个location定义的环境中被处理。只有被匹配到的这个模块和它内部的指令可以指定如何处理这个请求，而不会被其它location模块中的指令干扰。

尽管这个原则可以让你预知一个你设计的location模块的处理结果，但你得知道，有的时候这个location模块中的一些指令会触发一次新的location模块的匹配行为。上面这个“只会有一个模块来处理请求”的原则可能会与实际请求在你设计的location中的行为不符。
下面这些指令产生的内部跳转会导致上面说的这种情况，它们是：

* index
* try_files
* rewrite
* error_page


#### 让我们简单看一下这些指令都做了什么：

**index** 当使用index指令来处理请求时，会导致内部跳转。我们经常会精准匹配模式的模块来提高匹配效率（因为它可以立即中断匹配流程），但如果你精准匹配的是一个目录，在实际的执行流程中就会触发一次新的匹配。
在下面的例子中，第一个location模块会被URI/exact匹配到，但在处理这个请求的过程中，继承的index指令会导致这个请求被转到第二个location模块中。

```text
index index.html;
location = /exact {
    . . .
}

location / {
    . . .
}
```

在上面的例子中，如果你真的需要让请求在第一个模块中被处理完毕，你必须通过其它的方法来让这个请求可以匹配到这个模块，例如，你可以设置一个个不合法的index，并且打开autoindex。

```text
location = /exact {
    index nothing_will_match;
    autoindex on;
}

location  / {
    . . .
}
```
这是阻止index指令产生跳转的一种方法，但对于大多数场景来说，这是没用的。大多数场景下，精确匹配到一个目录时使用rewrite指令（也会导致一个跳转）比较有用。


**try_files:** 使用try_files会导致重新分析location模块。这个指令让Nginx检查一组文件或目录是否存在。最后一个参数可以是一个让Nginx产生内部跳转的URI。
看一下下面这个配置：

```text
root /var/www/main;
location / {
    try_files $uri $uri.html $uri/ /fallback/index.html;
}

location /fallback {
    root /var/www/another;
}
```

在上面这个例子中，
* 如果有一个URI为/blahblah的请求，第一个模块就会先被执行。
* 它会尝试在/var/www/main这个目录中查找blahblah.html这个文件。
* 当找不到时，他会去查找在/var/www/main中是否有目录叫作blahblah/。
* 当以上查找都失败时，它将跳转到/fallback/index.html。
* 这将会触发另一个匹配流程，这时请求将匹配到第二个模块。
* 此时请求将返回/var/www/another/fallback/index.html的内容。


**rewrite:** 当使用最后一个参数为last的rewrite指令时，或使用不带参数，Nginx会根据rewrite的结果来启动一次新的匹配流程。
例如，如果我们在上面这个例子中添加rewrite指令，可能会导致有的请求不会被try_files指令处理到：

```text
root /var/www/main;
location / {
    rewrite ^/rewriteme/(.*)$ /$1 last;
    try_files $uri $uri.html $uri/ /fallback/index.html;
}

location /fallback {
    root /var/www/another;
}
```

在上面这个例子中，URI为/rewriteme/hello的请求会先被第一个模块匹配。然后会被重新写为/hello并且重新匹配location，然后又会被第一个模块匹配，并且开始根据try_files的配置进行查找，当什么都没找到时，会跳到/fallback/index.html（跳转的情况跟我们在上面讲的一致）。
然而，如果请求的URI为/rewriteme/fallback/hello，第一个模块也会首先被匹配到，重写URI之后，得到的URI是/fallback/hello,这时这个请求将直接被第二个模块匹配到。
return指令返回301或302时也会产生相似的情况，不同的是这种情况下将会产生新的请求。如果rewrite指令使用redirect或permanent标志时，也会产生类似的情景。当然，由于这种重定向的请求是可见，它们应该是可控的。

<br>
**error_page:**  error_page指令也能产生与try_files类似的内部跳转。这个指令用来定义特定状态码时的动作。在try_files被定义时，这个指令可能永远不会被执行，因为try_files定义了请求的整个周期。

看一下下面这个例子：

```text
root /var/www/main;

location / {
    error_page 404 /another/whoops.html;
}

location /another {
    root /var/www;
}
```

所有请求都会被第一个模块处理（除了以/another开头的请求），将会返回/var/www/main中的内容。但如果在/var/www/main没有找到对应的文件（状态码为404），将会内部跳转到/another/whoops.html，这时将会匹配到第二个模块，实际提供服务的将是/var/www/another/whoops.html这个文件的内容。
因此，理解Nginx触发新的location匹配行为的情景，有助于预知请求中发生的行为。

### 总结:
----------------------

**理解Nginx处理客户端请求的原理，可以让你做Nginx管理员时更加轻松。你将会知道针对每个请求，Nginx将会使用哪个模块来提供服务。你也可以清楚的了解Nginx如何根据URI去匹配location模块。总之，了解Nginx匹配模块的原理，可以让你具有追踪每个请求在Nginx中的运行环境的能力。**




##### 文章来源：36kr大前端团队分享

