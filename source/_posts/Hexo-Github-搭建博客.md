---
title: Hexo + Github 搭建博客
date: 2019-08-25 23:19:21
categories:
- Tools
tags: 
- Hexo
---

# Hexo

一种静态博客框架

常用命令

```shell
hexo new <title>

# hexo generate
hexo s # hexo server

hexo deploy
```

分类和标签

```
categories:
- [Diary, PlayStation]
- [Diary, Games]
- [Life]
tags:
- PS3
- Games
```



## 安装和使用

[Hexo官方文档]( https://hexo.io/zh-cn/docs/) 有详细的描述

遇到问题可以看看下面tips目录下有无解决方案

## 主题

默认主题是landscape

[主题官方网页](https://hexo.io/themes/) 查看喜欢的主题，点名称转到github地址，clone到themes目录下，并将**站点配置文件**_config.yml中的主题改为对应的主题名。

这里记录下了一些我觉得好看的主题

https://theme-next.org/   	  https://github.com/theme-next

http://lotabout.me/hexo-theme-noise/

https://colmugx.github.io/

https://zalando-incubator.github.io/hexo-theme-doc/

https://caichenghan.github.io/

https://blog.zhangruipeng.me/hexo-theme-minos/

https://www.haomwei.com/

https://ice.gs/

http://www.codeblocq.com/assets/projects/hexo-theme-clean-blog/

---

# Github

代码托管网站

提供Git Pages 功能，每个用户可以拥有一个仓库名为`用户名 + .github.io`，用来托管网页内容

用的比较多的博客框架有Jekyll和Hexo

网上均有比较多的教程

---

# Tips

搭建过程中虽然流程简洁，但仍然遇到了不少问题

## 本地预览博客效果

```
hexo s
```

s表示server，用浏览器打开 [localhost:4000](localhost:4000) 就可以看到博客的效果了，<kbd> Ctrl </kbd> + <kbd>C</kbd>结束

## npm install慢的问题

**npm install非常慢，可能超时安装失败**

用 get命令查看registry

```
npm config get registry
```

原版结果为

```
http://registry.npmjs.org
```

用set命令换成阿里的镜像

```
npm config set registry https://registry.npm.taobao.org
```

再执行命令

```
npm install
```

或者直接执行

```
npm install --registry=https://registry.npm.taobao.org
```

## 安装yarn导致hexo文件夹混乱

> 如无必要，勿增实体

对前端几乎没啥了解，一通瞎整后，成功地让文件变得混乱起来

卸载yarn也失败，删掉bin文件夹后，它自己又回来了

另外报错code-js缺失，执行`npm install code-js@3`重新装回来就好了

## 设置语言不起效

比如我按照nextT官网在站点配置文件中配置`zh-Hans`没有起效（这个锅是官方文档的，文件名是`zh-CN`，文档里写的是`zh-Hans`）

解决方案：

打开对应主题的languages文件夹，查找自己想要的语言配置，保证站点配置文件中的语言配置和languages文件夹下的文件名一致

我这里将站点配置改成`zh-CN`就可以了

## Hexo是静态的博客框架

展示页面和内容编辑的关系

## 标签和分类

需要自己创建tag页面（取消注释，创建相关页面）

```
hexo new page "tags"
```

创建的页面中：加`type: "tags"`

next主题配置文件中取消menu对tags的注释

（分类等相同操作）

Hexo 中两者有着明显的差别：分类具有顺序性和层次性，也就是说 `Foo, Bar` 不等于 `Bar, Foo`；而标签没有顺序和层次。

## github push报错443

检查是否开了代理，导致ssh连接失败

## hexo d 的git失败问题

这里是个相当大的坑，整了好久，同时网慢误以为安装depoyler失败，结果是站点配置文件，我写的是`repo`，改成`repository`就好了

远程库的地址为 git@... 不能用https等

网上看到有人和我的相反，总之发布找不到git出错可以考虑**repository**和**repo**两个换着试试

## 多台计算机编辑

io的master分支只有public的静态文件，如果想要从仓库中拿到源文件并编辑，那么仓库中需要存放生成hexo静态页面用的文件

两种方案，一是将源文件放到另一个仓库中；二是将源文件放在io库的另一个分支上，master用来展示静态页面，另一个分支用来开发和写博客

## 提交源文件

需要注意一个问题，主题文件夹也受git版本控制，有两种方案处理

一是将主题的git版本控制文件删掉，直接提交主题内容，这样相当于在hexo源文件中放了一份主题的当前备份，弊端是从作者的源库中获取最新版本特别不方便；

二是利用git的submodule功能，相当于记录一个theme主题repo的地址，弊端是自己做修改不能同步到远端；为了解决这个问题，可以在自己的仓库中fork一份主题，不过如果不需要和作者源库做同步，fork一份会稍微繁琐一点，不如第一种方法直接。

（我采用的是第一种方式）

## git允许仓库嵌套

git中可以直接加如子项目

即git仓库可以嵌套

相当于在项目中包含了一个子模块的版本控制指针，项目中实际上并没有这个内容

![](/img/images/1566724273431.png)

![](/img/images/1566725941936.png)

![](/img/images/1566725971896.png)

# 阿里云图床+PicGo

与这个博客操作类似，不再详细写

https://www.cnblogs.com/Kylin-lawliet/p/16446536.html

注意：

- Typora中用PicGo app上传
- PicGo app中设置时间戳命名，经过测试，同名不同内容的文件不会再次上传，直接返回已有url
