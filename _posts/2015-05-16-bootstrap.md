---
layout: post
category: linux
title: Bootstrap
tags: [bootstrap]
---

### 使用HTML5文档类型

```html
<!DOCTYPE html>
<html lang="zh-CN">
  ...
</html>
```

### meta 可以使浏览器默认使用极速模式

```html
<!DOCTYPE html>
<html>
    <head lang="zh-CN">
        <meta name="renderer" content="webkit">
    </head>
    <body>
    </body>
</html>
```

### 布局容器

所有的右边布局的宽度是：1000px，在装饰器里面的右边的div添加class .container

```html
<div class="container">
  ...
</div>
```

### 栅格系统

我们用的是.col-md-* 默认12格，每格的间距为30px, 必须放在.row下面, 列偏移 .col-md-offset-*

参考 <http://v3.bootcss.com/css/#grid-example-basic>

### 常用的其它class

```html
<div class="clearfix">                      <!--清除浮动-->
    <div class="pull-left">                 <!--向左浮动-->
    </div>
    <div class="pull-right">                <!--向右浮动-->
    </div> 
</div>

<div class="hidden">...</div>               <!--隐藏元素-->

<div class="center-block">...</div>         <!--让内容块居中-->
```

### 颜色和大小

<div class="row">
    <div class="bg-primary col-md-4">背景颜色</div>
    <div class="text-primary col-md-4">字体颜色</div>
    <button class="btn btn-primary  col-md-4">按钮颜色</button>
</div>
<div class="row">
    <div class="bg-info col-md-4">背景颜色</div>
    <div class="text-info col-md-4">字体颜色</div>
    <button class="btn btn-info col-md-4">按钮颜色</button>
</div>
<div class="row">
    <div class="bg-success col-md-4">背景颜色</div>
    <div class="text-success col-md-4">字体颜色</div>
    <button class="btn btn-success col-md-4">按钮颜色</button>
</div>
<div class="row">
    <div class="bg-warning col-md-4">背景颜色</div>
    <div class="text-warning col-md-4">字体颜色</div>
    <button class="btn btn-warning col-md-4">按钮颜色</button>
</div>
<div class="row">
    <div class="bg-danger col-md-4">背景颜色</div>
    <div class="text-danger col-md-4">字体颜色</div>
    <button class="btn btn-danger col-md-4">按钮颜色</button>
</div>
