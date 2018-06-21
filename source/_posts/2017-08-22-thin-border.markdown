---
layout: post
title: "移动端html5 1px border的实现"
date: 2017-08-22 20:11
author: "Gubaidan"
header-img: ""
cdn: 'header-on'
tags: 
  - css3 
---



<!-- more -->
### 一、真正的1px边框

做完APP前端后大师兄就告诉我h5上的边框线太粗,把整站都给拉low了. 当时工期紧就没太在意1px粗细, 后面的版本针对这个问题做了些尝试, 这里总结下1px细线的处理方法，因为这已是最细的边框，电脑上清楚的显示，我已经设置了1px的border。

于是为了寻找在移动端看起来像真正1px 的border，搜集一堆资料后还真找到了方案：

- **父元素设置**：scale(0.5,0.5)                 
- **子元素设置**：scale(2,2) 还原缩放，origin都是基于左上角（0,0）/left top

这样父元素的border其实被缩放了，无疑更细。

### 二、通用方案

用一个css类去为block元素添加更细的border

```css
css
.border-1px{
  position: relative;
  &:before, &:after{
    border-top: 1px solid #c8c7cc;
    content: ' ';
    display: block;
    width: 100%;
    position: absolute;
    left: 0;
  }
  &:before{
    top: 0;
  }
  &:after{
    bottom: 0;
  }
}
```

适应移动设备：


```css
css
@media (-webkit-min-device-pixel-ratio:1.5), (min-device-pixel-ratio: 1.5){
  .border-1px{
    &::after, &::before{
      -webkit-transform: scaleY(.7);
      -webkit-transform-origin: 0 0;
      transform: scaleY(.7);
    }
    &::after{
      -webkit-transform-origin: left bottom;
    }
  }
}

@media (-webkit-min-device-pixel-ratio:2), (min-device-pixel-ratio: 2){
  .border-1px{
    &::after, &::before{
      -webkit-transform: scaleY(.5);
      transform: scaleY(.5);
    }
  }
}
```


### 三、简易方案

```css
css
.line {position:relative;}
.line:after {
	width:200%;
	height:200%;
	position:absolute;
	top:0;
	left:0;
	z-index:0;content:"";
	-webkit-transform:scale(0.5);
	-webkit-transform-origin:0 0;
	transform:scale(0.5);
	transform-origin:0 0;
	box-sizing:border-box;
}
.list {
	width:100%;
	margin:auto;
	list-style:none;
	padding:5px 20px;
}
.list:after {
	border:1px solid #ccc;
	border-radius:10px;
}
.item {padding:0px;}
.item:after {border-bottom:1px solid #ccc;}
.item:last-child:after {display:none;}
```

如此调用

```css
<div class="item line"></div>
```

这种方式适合某个元素下方需要有一条极细的border

### 四、对比

上图是原生1px，下图是解决后的效果                
![与原生方案对比](http://p9n2j0ewi.bkt.clouddn.com/blogimg/border1px.jpg)  

### 五、产生差别的原因

> 为什么移动端css里面写了1px, 实际看起来比1px粗. 其实原因很好理解:这2个'px' 的含义是不一样的. 移动端 html 的 header 总会有一句


```css
<meta name="viewport" content="width=device-width, initial-scale=1.0, maximum-scale=1.0, user-scalable=no">
```


> 这句话定义了本页面的 viewport 的宽度为设备宽度,初始缩放值和最大缩放值都为1,并禁止了用户缩放. viewport 通俗的讲是浏览器上可用来显示页面的区域, 这个区域是可能比屏幕大的。


>手机存在一个能完美适配的理想viewport, 分辨率相差很大的手机的理想viewport的宽度可能是一样的, 这样做的目的是为了保证同样的css在不同屏幕下的显示效果是一致的, 上面的meta实际上是设置了ideal viewport的宽度.


>以实际举例: iphone3 和 iphone4 的屏幕宽度分别是 320px, 640px, 但是它们的 ideal viewport 的宽度都是 320px, 设置了设备宽度后, 320px 宽的元素都能 100% 的填充满屏幕宽. 不同手机的 ideal viewpor t宽度是不一样的, 常见的有 320px, 360px, 384px. iphone 系列的这个值在 6 之前都是 320px, 控制 viewport 的好处就在于一套 css 可以适配多个机型.


>因此, viewport 的设置和屏幕物理分辨率是按比例而不是相同的. 移动端 window 对象有个 devicePixelRatio 属性, 它表示设备物理像素和 css 像素的比例, 在 retina 屏的 iphone 手机上, 这个值为 2 或 3, css 里写的 1px 长度映射到物理像素上就有 2px 或 3px 那么长.



