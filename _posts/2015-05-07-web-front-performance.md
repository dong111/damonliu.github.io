---
layout: post
category: front
title: 前端性能优化
tags: [front]
---

# 前端性能优化

### 最少的请求

80%的响应时间花在下载网页内容(images, stylesheets, javascripts, scripts, flash等)。减少请求次数是缩短响应时间的
关键！可以通过简化页面设计来减少请求次数，但页面内容较多可以采用以下技巧。

- 捆绑文件 多个js或者css文件的合并。
- css尽量少用图片，可以把用的图片拼装成一个图片，通过css控制显示那个部分。
- inline images，通过将图片进行编码内嵌到网页中

### 减少DNS查询次数

DNS查询也消耗响应时间，如果我们的网页内容来自各个不同的domain (比如嵌入了开放广告，引用了外部图片或脚本)，
那么客户端首次解析这些domain也需要消耗一定的时间。DNS查询结果缓存在本地系统和浏览器中一段时间，所以DNS查
询一般是对首次访问响应速度有所影响。IE 默认的缓存DNS30分钟，Firefox默认的DNS缓存只有1分钟，而FasterFox
修改DNS的缓存为1小时。一般域名不要超过4个。

### 使用CDN

再次强调第一条黄金定律，减少网页内容的下载时间。提高下载速度还可以通过CDN(内容分发网络)来提升。CDN通过部署
在不同地区的服务器来提高客户的下载速度。如果你的网站上有大量的静态内容，世界各地的用户都在访问，
那CDN是必不可少的。事实上大多数互联网中的巨头们都有自己的CDN。我们自己的网站可以先通过免费的CDN供应商来分发网页资源。

- 基本方法

```javascript
    <script src="http://ajax.googleapis.com/ajax/libs/jquery/2.0.0/jquery.min.js"></script>
    <script>
        if (typeof jQuery == 'undefined') {
            document.write(unescape("%3Cscript src='js/jquery-2.0.0.min.js'%3E%3C/script%3E"));
        }
    </script>
```

- 协议省略

```javascript
    <script src="//ajax.googleapis.com/ajax/libs/jquery/2.0.0/jquery.min.js"></script>
```
上面的代码看起来奇怪，但“协议省略”的网址是引用第三方内容的最好的方式，它可以通过Http或Https引用。当页面加载时，
对于非加密请求脚本会通过Http方式引用并且缓存起来，以此同时对于加密请求脚本会根据“协议省略”方式使用Https引用内容，
所以使用“协议省略”的URL允许单个脚本更灵活地引用内容。


### 减少DOM元素数量，减少页面大小

网页中元素过多对网页的加载和脚本的执行都是沉重的负担，500个元素和5000个元素在加载速度上会有很大差别。
想知道你的网页中有多少元素，通过在浏览器中的一条简单命令就可以算出
```javascript
document.getElementsByTagName('*').length
```
比如google的html coding style就规定省略不必要的标签
```html
<ul>
    <li>1
    <li>2
</ul>
<table>
    <tr>
        <td>1
        <td>2
    </tr>
</table>
```


### 避免css表达式

CSS表达式可以动态的设置CSS属性，在IE5-IE8中支持，其他浏览器中表达式会被忽略。例如下面表达式在不同时间设置不同的
背景颜色。
```css
    background-color: expression( (new Date()).getHours()%2 ? "#B8D4FF" : "#F08A00" );
```
CSS表达式的问题在于它被重新计算的次数远比我们想象的要多，不仅在网页绘制或大小改变时计算，即使我们滚动屏幕或者移动
鼠标的时候也在计算，因此我们还是尽量避免使用它来防止使用不当而造成的性能损耗。如果想达到这样的效果可使用javascript

### 将样式表置顶

经样式表(css)放在网页的HEAD中会让网页显得加载速度更快，因为这样做可以使浏览器逐步加载已将下载的网页内容。这对内容
比较多的网页尤其重要，用户不用一直等待在一个白屏上，而是可以先看已经下载的内容。

如果将样式表放在底部，浏览器会拒绝渲染已经下载的网页，因为大多数浏览器在实现时都努力避免重绘，样式表中的内容是绘制
网页的关键信息，没有下载下来之前只好对不起观众了。

> 不要使用@import，可以使用link代替，因为@import相当于把css放在网页内容的底部

### 将javascript置低

把脚本置底，这样可以让网页渲染所需要的内容尽快加载显示给用户。

### 使用单独的js和css文件

使用外部Javascript和CSS文件可以使这些文件被浏览器缓存，从而在不同的请求内容之间重用。
同时将Javascript和CSS从inline变为external也减小了网页内容的大小。
 
 
### 精简Javascript和CSS

比如一般我们用的第三方js和css文件都会有一个product版本，去掉了里面不必要的空格，减少了
文件的大小。统计表明精简后的文件大小平均减少了21%。
用来帮助我们做精简的工具很多，主要可以参考如下，
- JS compressors
> * [Packer](http://dean.edwards.name/packer/) 
> * [JSMin](http://crockford.com/javascript/jsmin)
> * [Closure compiler](http://code.google.com/intl/pl/closure/compiler/)
> * [YUICompressor](http://developer.yahoo.com/yui/compressor/)
- CSS compressors
> * [CSSTidy](http://csstidy.sourceforge.net/)
> * [Minify](http://code.google.com/p/minify/)
> * [CSSCompressor](http://www.csscompressor.com/)

### 使用GET Ajax请求

浏览器在实现XMLHttpRequest POST的时候分成两步，先发header，然后发送数据。而GET却可
以用一个TCP报文完成请求。另外GET从语义上来讲是去服务器取数据，而POST则是向服务器发送数据，
所以我们使用Ajax请求数据的时候尽量通过GET来完成。

### 缓存Ajax

Ajax可以帮助我们异步的下载网页内容，但是有些网页内容即使是异步的，用户还是在等待它的返回结果，
例如ajax的返回是用户联系人的下拉列表。

### 避免空的图片src

空的图片src仍然会使浏览器发送请求到服务器，这样完全是浪费时间，而且浪费服务器的资源。尤
其是你的网站每天被很多人访问的时候，这种空请求造成的伤害不容忽略。





