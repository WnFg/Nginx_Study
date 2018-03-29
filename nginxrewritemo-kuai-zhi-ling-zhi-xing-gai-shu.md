## ![](http://wiki.baidu.com/plugins/servlet/confluence/placeholder/macro?definition=e3RvY30&locale=zh_CN&version=2)

## 1. rewrite模块简介

ngx\_http\_rewrite\_module属于Http模块，常用功能是通过正则匹配来修改请求的URI或有选择的使某些配置生效。

该模块的常用指令集：

* break
* if
* return
* rewrite
* set

指令的执行顺序：

The [break](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#break), [if](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#if), [return](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#return), [rewrite](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite), and [set](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#set) directives are processed in the following order:

* the directives of this module specified on the [server](http://nginx.org/en/docs/http/ngx_http_core_module.html#server) level are executed sequentially;
* repeatedly:
  * a [location](http://nginx.org/en/docs/http/ngx_http_core_module.html#location) is searched based on a request URI;
  * the directives of this module specified inside the found location are executed sequentially;
  * the loop is repeated if a request URI was [rewritten](http://nginx.org/en/docs/http/ngx_http_rewrite_module.html#rewrite), but not more than [10 times](http://nginx.org/en/docs/http/ngx_http_core_module.html#internal).

## 2. nginx配置文件解析简介

很多时候，nginx的配置文件被看作是特定语言定义的一个脚本，但是这种视角会给人一种错觉——当一个请求被Nginx处理时，指令是按配置文件中的顺序作用于该请求。实质上，配置文件中的指令在Nginx启动阶段，将被其所属的模块解析，并组合成一个特定的结构体，并在处理请求时被使用。例如，当配置文件解析完成后，所有的http模块根据配置文件内容将生成如下结构：

![](http://wiki.baidu.com/download/attachments/481109760/image2018-3-26%2022%3A21%3A12.png?version=1&modificationDate=1522074073000&api=v2)

                                                 图1

\(以下内容仅讨论nginx中的http模块\)

其中main\_conf为配置文件中的http块对应的解析结果，srv\_conf对应server块，loc\_conf则对应Location块。每个xxx\_conf对应了一个数组，该数组中存储了Nginx中所有http模块定义的结构体地址，该结构体一般被用来存储配置项。至于为何location或server对应的ngx\_http\_conf\_ctx\_t需要有指向更上一级的配置的原因是，一个相同的配置项可能在多个层次中出现，那如何决定哪个层次中的该配置内容生效呢——需要定义某种合并策略，来对其进行处理，所以需要对上层配置进行遍历。

从上述的描述中可以看到，配置文件中的指令更多表现出一种静态概念，先于请求处理前已被nginx解析，并决定了Nginx中模块的处理规则。而指令真正的作用顺序则由两个内容决定，1. 定义该指令的模块介入了nginx的11个http阶段\(图2\)中的哪一个，2. 编译nginx时，该指令所属模块在所有模块中的排位。

## 3. rewrite模块是如何工作的

![](http://wiki.baidu.com/download/attachments/481109760/image2018-3-26%2022%3A58%3A32.png?version=1&modificationDate=1522076312000&api=v2)

                                                            图2

Nginx为了实现对uri的重写，以及有条件的使部分配置生效等功能，需要引入一些流程控制结构\(例如If\)，这就是rewrite模块做的事了。如图2所示，rewrite模块在NGX\_HTTP\_SERVER\_REWRITE\_PHASE和NGX\_HTTP\_REWRITE\_PHASE两个阶段起作用，作用的方法简单来说就是注册一个处理函数到Nginx框架中，当框架进行到某一阶段时，按顺序调用所有模块注册的处理函数。

根据内容2，rewrite所定义的指令也将在nginx启动阶段被解析并放入相应的结构体内，但与大多数模块不同的是，rewrite模块定义的指令集构成一个简单的脚本语言\(参见ngx\_http\_script.c\)，其指令在被解析后将成为中间指令，这些中间指令，将在http的NGX\_HTTP\_REWRITE\_PHASE\(或NGX\_HTTP\_SERVER\_REWRITE\_PHASE\)阶段被一个简单的解释器执行，而rewrite模块在该阶段的注册函数即为解释器的入口。

从以上内容可以看到，rewrite模块定义的指令相比于其它模块的指令更具有被执行的感觉\(毕竟要被解释器执行\)，那么它是如何有选择的使一些配置生效呢？看下面的例子：

location / {

root html;

echo $test;

if  \($uri ~ .\*jpg$\) {

          root jpg;

          set $test i\_am\_jpg;

          break;

}

if  \($uri ~ .\*png$\) {

          root png;

          set $test i\_am\_png;

          break;

}

set $test i\_am\_default

}

           例1

上面的Location中，root指令将被http\_core模块解析，echo指令被http\_echo模块解析，其余指令被rewrite模块解析，大致形成如图3的location级别的配置结构，对应图4红框内的结构

![](http://wiki.baidu.com/download/attachments/481109760/image2018-3-27%2012%3A37%3A6.png?version=1&modificationDate=1522125426000&api=v2)

                                                              图3

![](http://wiki.baidu.com/download/attachments/481109760/image2018-3-27%2012%3A39%3A13.png?version=1&modificationDate=1522125553000&api=v2)

                                                      图4

例1中的rewrite指令，在被解析后，大致形成如下伪代码：

start

test $uri ~ .\*jpg

check against zero

        replace location\_configure

        set  $test  i\_am\_jpg

        exit rewrite\_phase

test $uri ~ .\*png

check against zero

        replace location\_configure

        set  $test  i\_am\_jpg

        exit rewrite\_phase

set $test i\_am\_default

end

结合图3及上述伪代码，可以看出，当nginx启动后，配置文件解析如图3所示，而上述中间代码则存储于rewrite模块生成的结构体，特别的，在rewrite解析if指令时，会继承上一级的Location配置，并根据if大括号中的配置内容生成新的location配置。例如，上一级的root为html，而if \($uri ~ .\*jpg\) 对应的location结构体中root为jpg。

当开始处理http请求时，uri匹配到相应的location，并到达rewrite\_phase阶段时，rewrite模块开始执行其中的中间代码。简单来看上面的伪代码：

1. 对于if \($uri ~ .\*jpg\)结构，首先计算测试条件，然后判断若测试结果非0的话，为成功。此时，首先将原location配置，替换为该if对应的location配置
2. set指令，在当前请求的内存结构中注册名为test的变量
3. 最终执行break指令，终止后续的rewrite指令执行

当请求的uri以jpg\(png\)结尾，最终响应是i\_am\_jpg\(png\)，其它则是i\_am\_default。可以看到echo $test 在if之前出现，但是仍然可以输出下面使用set指令的$test内容。原因就是内容2所说，echo指令被http\_echo模块解析，且在NGX\_HTTP\_CONTENT\_PHASE阶段起作用，即在内容生成阶段输出变量$test的值，而该值在rewrite\_phase阶段时将被赋值为i\_am\_jpg。

## 4. rewrite模块其它指令

return 就比较简单了，语法：return **code \[text\];    **例如return 200 "ok"，向客户端返回响应，响应码为200，内容为ok。

rewrite 指令用来重写uri，语法：rewrite_`regexreplacement`_ \[_`flag`_\];

当uri匹配到regex时，替换为replacement，其中flag是可选的，flag有以下选择：

* last          终止后续rewrite指令的执行，并回到NGX\_HTTP\_CONFIG\_PHASE阶段，重新根据uri匹配location，最多循环10次
* break       终止后续rewrite指令的执行
* redirect    http 302，重定向
* permanent   http 301

特别的，若replacement以http,https或$scheme开头，则直接向客户端发送重定向响应。当replacement有参数时，uri重写前的参数将后缀到replacement之后，如果不想这样，可以在replacement最后放一个问号——'?'，例如：rewrite ^/users/\(.\*\)$ /show?user=$1? last;

## 5. 小结

到这里，可以知道，若想要了解一个nginx配置项的作用，需要首先知道其可以出现的上下文\(context\)，语法，及大致作用，这个可以从nginx官方文档中快速找到，例如

![](http://wiki.baidu.com/download/thumbnails/481109760/image2018-3-27%2020%3A3%3A39.png?version=1&modificationDate=1522152219000&api=v2)

但若想详细了解该配置项的作用规则，则需要了解它是哪个模块定义的，该模块如何介入Nginx的处理框架中，该模块属于哪一类型的模块，例如http模块，过滤模块，事件处理模块等，每种类型的模块都有Nginx框架规定的一套接口，用来完成不同的功能，不过我们接触最多的应该是http模块。

  


