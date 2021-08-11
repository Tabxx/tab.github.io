---
title: css变量那些事
description: css变量那些事
top_img: 'topbg.jpg'
cover: 'topbg.jpg'
date: 2020-03-18 10:35:42
tags: CSS
---


  css变量也不是一个新词，但是说起css变量可能很多同学都不太熟悉，因为它的使用场景确实不太多，做过皮肤切换功能的同学估计是了解css变量带来的好处（变量是个好东西）。
   
   
###  浏览器支持  
   开始之前先了解下css变量的兼容性，避免一顿操作猛如虎，一看兼容泪满面。现在市面上主流的浏览器基本上都支持（IE6这玩意别跟我说主流），全球网站流量的 77％ 支持CSS变量，而美国已经接近90％。
   ![e1b75cac7515da90a7ebdfe923f8f104.jpeg](./Screenshot0119-1141.jpg)
   
   

### 声明你的第一个css变量
既然是变量，那么可以大胆的猜测它一定有作用域(后面会讲到)。我们先在全局也就是 `:root`  上定义，也意味着所有的DOM节点都能使用到全局的变量。
```css
:root {
  --title-color: #1b1b1b;  
}
```
声明变量的语法是 ` --*` ，至于命名规则就比较有意思了，css变量命名除了不能包含 `$` ，`[` ，` ^`，`(`，`%`等字符之外都是可以，甚至中文字也行。


### 使用你的第一个css变量
刚刚我们已经在全局声明了一个`--title-color`，那么现在我们可以在任意位置去使用到它。
```css
.card-title {
    color: var(--title-color)
}
```
 使用css变量的语法是`var(--*)`，怎么样，是不是很简单呢
 
 
 ### css变量权重or作用域
 我们在全局声明一个`--color` ， 并且在 `div` 选择器下面再声明一个`--color`
```css
:root { 
    --color: red;
}
div {
    --color: blue;
}
div.box {
    color: var(--color);
}
```
根据以往写css的经验，box这个div容器的字体应该是蓝色。事实证明确实是这样(⑉• •⑉)‥♡
```html
<div class="box">
我的蓝色来自于div
</div>
<div class="box">
我的红色来自于根元素
</div>
```
上面这个简单的例子我们可以看出：当存在多个命名相同的变量时，还是会遵循**CSS选择器的权重**规则来取值，但是 `!important`  不能用在变量这里。


### css变量缺省语法
其实` var() `可以传递两个参数，它的完整语法是`var( <自定义属性名> [, <默认值 ]? )`.
```css
:root { 
    --size: 12px;
}
div {
    font-size: var(--title-size, 16px); 
    background-color: var(--size, #ddd);
 }
```
看完上面这段代码是不是想说，so easy，`div` 最后编译的结果是肯定是这样子的：
```css
div {
    font-size: 16px; 
    background-color:  #ddd;
 }
```
有这种想法的同学还是太年轻了，结果是这样的
```css
div {
    font-size: 16px; 
    background-color:  transparent;
 }
```
纳尼？
![4a69f941957fa542c2f37d379b1f0d4b.png](./question.png)


var第二个参数默认值，当且仅当使用的**变量未定义**时才能生效；如果是变量值无效的情况，第二个参数也是没有什么作用的。

上面这例子`--size` 已经被定义了，也就是说会变成`background-color: 12px;` ，一看就知道这语法是错误的，那么最终会取到浏览器默认的背景色也就是`transparent`。

 ### 应用场景
到这里css变量的基本使用方法已经差不多了，下面来看看它的应用场景，加深点印象。
##### 主题皮肤切换
拿一个最简单的例子来理解一下主题皮肤切换的效果。需求是这样的：有一个容器的字体颜色，在白色主题下显示黑色，在黑色主题下显示白色，默认白皮肤（够简单吧）；
直接上代码：
```html
<style>
:root {
    --card-title: #000;
}
body[data-theme="black"] {
    --card-title: #fff;
}
.box {
    color: var(--card-title);
}
...
</style>

<body>
   <div class="box"> 我是一个会变色的标题 </div>
</body>
```
剩下的我们就只需要动态修改`body` 的 `data-theme` 属性的值来切换主题。有没有同学想说，那我直接js修改`box`的`color`属性不就完事了？？？这种想法很危险，万一是网页全局主题修改，涉及到的配色有很多呢，用js动态的一个个改不合适吧。

##### 响应式布局
需求：根据不同的屏幕尺寸显示不同的字体大小
还是直接上代码吧
```css
:root { 
    --main-font-size: 16px; 
} 
media all and (max-width: 1024px) {
    :root { 
        --main-font-size: 14px;
     } 
}
```
几行代码就能够实现，是不是很刺激。



最后在补充好几点：
1、css变量**不能在属性名上使用**
```css
:root { --bgc: background-color;  }
div { var(--bgc): red; }
```
2、不要想着和`px` 这种单位搞个拼接， 不支持
```css
:root { --size: 20;  }
div { font-size: var(--size)px; }
```
3、变量之间可以相互定义
```css
:root { 
    --color: red;
    --bg-color: var(--color); // 红色
 }
div { background-color: var(--bg-color); } 
```

<div class="aplayer no-destroy" data-id="热门" data-server="tencent" data-type="search" data-fixed="true" data-mini="true" data-listFolded="false" data-order="random" data-preload="none" data-autoplay="true" muted></div>

