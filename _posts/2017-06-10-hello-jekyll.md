---
layout: post
title: 在Github上使用Jekyll搭建自己的博客
categories: [jekyll]
tags: [jekyll, github]
---


> 作为一个码农，总是不喜欢各种博客平台的约束，总是希望把一切(`code`)都掌握在自己手中，而`Github`为我们提供了`免费的午餐`

#### 1. 在github上创建博客仓库
![]({{ site.url}}/public/images/2017-06-10-hello-jekyll-1.png)

> 注意这里的`xx`一定要和用户名相同，最终生成的网站地址才是`https://xx.github.io`，有自己的域名则可随意

#### 2. 修改仓库Settings

找到`GitHub Pages`设置：
![]({{ site.url}}/public/images/2017-06-10-hello-jekyll-2.png)

选择`源码分支`和`theme`
![]({{ site.url}}/public/images/2017-06-10-hello-jekyll-3.png)

#### 3. 拉取仓库到本地

```
git clone https://github.com/xx/xx.github.io.git
```

#### 4. 搭建jekyll环境

##### 4.1 安装ruby
> 下载并安装`ruby`

##### 4.2 安装jekyll
通过`gem`安装`jekyll`
```
gem install jekyll
```

##### 4.3 创建博客
```
jekyll new myblog
```

##### 4.4 本地运行

```
jekyll serve
```

##### 4.5 目录结构
`jeklly`的目录结构一般如下：
```
    |-_data // 存放数据文件
            |-favorites.yaml // 比如这里存放的是yaml格式的favorites数据
    |-_drafts // 草稿存放目录
    |-_includes // 存放公共html模板
            |-head.html
            |-sidebar.html
    |-_layouts // 存放布局模板
            |-default.html
            |-page.html
            |-post.html
    |-_posts // 这里放具体博客的md文件
            |-2017-06-10-hello-jeklly.md
    |-_site // 本地运行时生成的网站目录
    |-public // 存放js/css等静态资源
            |js
            |css
    |-_config.yml // 配置文件
    |-index.html // index
```

> 不同的`theme`目录结构也不同

#### 5. 写博客
在`_post`目录中新建`markdown`文件，例如`2017-06-10-hello-jeklly.md`，然后按照`markdown`语法写就可以了：

```
---
layout: post
title: 在Github上使用Jekyll搭建自己的博客
categories: [jekyll]
tags: [jekyll, github]
---


> 作为一个码农，总是不喜欢各种博客平台的约束，总是希望把一切(`code`)都掌握在自己手中，而`Github`为我们提供了`免费的午餐`

...

```

> 注意：`markdown`文件头部使用一组`---`定义了一些元数据

* layout: 使用的布局
* title: 标题
* categories: 分类，可有多个
* tags: 标签，可有多个

#### 参考

[https://jekyllrb.com/](https://jekyllrb.com/)