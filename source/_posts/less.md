---
title: less相关
date: 2018-02-22 14:18:27
tags: 前端
---

1.less中的变量

less中定义变量
```
@fontBig: 18px;
@fontNormal: 16px;
@fontSmall: 14px;
@fontSmaller: 12px;
@spacing: 20px;
@padding: 20px;
@theme-color:#ea9235;
```
less中应用变量
```
.container-m-title{
        border-bottom:1px solid @border-color;
        color:@main-font-color;
    }
```
```
/*由于less中自带数学计算，使用下面这种非less中的计算时需要使用~进行转义*/
width:calc(~"100% - 28px");
```

2.less中的层级关系  
我们知道在css中使用下面的方式表示层级关系
```
.outer .inner { text-align:center; }
```
而我们在less中使用嵌套的方式表示层级关系
```less
.outer{
    /*这里给外部设置样式*/
    width:100%;
    .inner{
        /*这里给内部设置样式*/
        text-align:center;
    }
    /*&表示父级本身，这里可以父级的before伪类*/
    &:before{
    }
    /*与上同理，设置hover样式*/
    &:hover{
        
    }
    /*表示有父级类的基础上包含该类名才生效的样式*/
    &.outer-font{
        font-size:16px;
    }
    /*直属子类的样式*/
    & > .child {
        
    }
}
```
3.less中的转义  
**计算的转义**  
less中是支持数学计算的，比如：
```less
width:100px + 200;
```
所以我们在使用比如说calc这种css3新特性时需要进行转义
```less
width:calc(~'100% - 20px');
```
**别名的转义**  
在我们使用@import引入使用webpack定义的别名时需要进行转义，写法如下：
```
@import '~assets/style/var.less';
```
4.less中的mixin  
mixin比较类似与我们平常写的函数比较类似，可以传入参数
```less
.bg-img(@urlStr) {
  background-image: url("@{urlStr}.png");
  @media only screen and (-webkit-min-device-pixel-ratio:2) {
    background-image: url("@{urlStr}@2x.png");
  }
  @media only screen and (min-device-pixel-ratio:2) {
    background-image: url("@{urlStr}@2x.png");
  }
}
```
当然mixin可以存在默认值
```less
.border(@border_width:3px;) {
    border:@border_width solid red;
}
```





