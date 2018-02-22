---
title: vue仿知乎切图组件开发手记
date: 2018-02-22 14:22:23
tags: 前端
---

在最近的开发工作中需要使用切图组件，但是发现vue中并没有特别好用的切图组件，所以就尝试自己遭了一个轮子，实现切图功能。  
**切图组件的理论依据**  
实现切图组件主要是依靠canvas的drawImage这个api,该api可以实现图片的缩放和剪切。关于该api的释义可以查看[MDN文档](https://developer.mozilla.org/zh-CN/docs/Web/API/Canvas_API/Tutorial/Using_images)。
**切图组件的基本开发步骤**

1.设定画布

canvas画布html结构如下所示
```html
 <canvas 
    :width="canvasW+'px'"
    :height="canvasH+'px'" 
    class="image-edit" id="canvas"
    :style="canvasStyl"
    @mousedown.stop="mousedownHandler"
    @mouseup.stop="mouseupHandler"
    @mousemove.stop="mousemoveHandler">
</canvas>
```
关于这里一些变量的定义如下所示

```javascript
    data () {
        return {
            //...
            //定义边框的宽度
            borderLen:38,
            //...
        }
    },
    props:{
        //...
        //图片宽度
        imageW:{
            type:Number,
            default:240
        },
        //图片高度
        imageH:{
            type:Number,
            default:160
        },
        //...
    },
    computed:{
        //canvas宽度
        canvasW () {
            return this.lenAdapter(this.imageW+(2*this.borderLen));
        },
        //canvas高度
        canvasH () {
            return this.lenAdapter(this.imageH + (2*this.borderLen));
        },
        //处理后的边框宽度
        aBorder () {
            return this.lenAdapter(this.borderLen);
        },
        //canvas的style样式
        canvasStyl () {
            return {
                'width': (this.imageW + 2 * this.borderLen) + 'px',
                'height': (this.imageH + 2 * this.borderLen) + 'px'
            }
        }
    }
```
这里可以看出canvas画布宽度由图像的宽度和边框构成，canvas画布高度由图像的高度和边框构成,而这里的lenAdapter函数用于根据屏幕dpr不同，长度为一倍或者二倍，具体实现如下所示
```javascript
//根据dpi的长度转换器
lenAdapter(len){
    if(devicePixelRatio>=2){
        len = len * 2; 
    }
    return len;
},
```
>注意在javascript中dpr为window对象上的一个属性，可以通过window.devicePixelRatio获取，也可以略写window，就是上面代码中的devicePixelRatio获取。

综上，在我们使用canvas画布时，需要同时设定width,height和style，width和height根据设备dpr的不同会放大一定的倍数，而style控制画布为规定的大小，这样就可以保证在retina屏幕上提供像素更多的canvas画布。

2.在画布上添加基本元素  
知乎切图组件的样式很简朴，主要为底部的图片和上层半透明边框组成。这里我使用canvas绘制两个正方形，并且在两个正方形的重叠位置绘制半透明的边框，绘制的代码如下所示。

```javascript
let context = document.getElementById('canvas').getContext('2d');
context.beginPath();
context.moveTo(0,0);
context.lineTo(this.canvasW,0);
context.lineTo(this.canvasW,this.canvasH);
context.lineTo(0,this.canvasH);
context.lineTo(0,0);
context.moveTo(this.aBorder,this.aBorder);
context.lineTo(this.canvasW - this.aBorder,this.aBorder);
context.lineTo(this.canvasW - this.aBorder,this.canvasH-this.aBorder);
context.lineTo(this.aBorder,this.canvasH - this.aBorder);
context.lineTo(this.aBorder,this.aBorder); 
context.fillStyle='rgba(230,224,224,0.6)';
//渲染图形重叠部分
context.fill("evenodd");    
```

3.在画布上渲染图片  
边框中间区域作为图片展示区域，图片最初的渲染会根据图片展示位置进行缩放，缩放规则为总是以图片比较小的长度（宽或者高）和图片展示区域的缩放比例为基准缩放，这样总是可以保证图片的宽和高一种是和图片区域完全重合的，另一种可以拖动，初始位置是位于图片展示区域居中。具体内容如下所示
```javascript
//获取上下文绘画对象
let ctx = document.getElementById('canvas').getContext('2d');
//创建图片对象
let image = new Image();
//设置图像源
image.src = src;
//图片加载成功的回调函数
image.onload = () => {
    //获取图片的原始宽度
    this.imageNatrueW = image.naturalWidth;
    //获取图片的原始高度
    this.imageNatrueH = image.naturalHeight;
    //设定初始横坐标
    this.imageX = this.aBorder;
    //设定初始纵坐标
    this.imageY = this.aBorder;
    //若图片比较宽，则高度跟视口重合
    if(this.imageNatrueW>this.imageNatrueH){
        //计算缩放比例
        this.scale = this.imageH / this.imageNatrueH;
        //计算出x坐标所处位置
        this.imageX = this.lenAdapter(-(this.scale*this.imageNatrueW - this.imageW)/2+this.borderLen);
    //若图片比较高，则宽度跟视口重合
    }else{
        //计算缩放比例
        this.scale = this.imageW / this.imageNatrueW;
        //计算出y坐标所处位置
        this.imageY = this.lenAdapter(-(this.scale*this.imageNatrueH - this.imageH)/2+this.borderLen);
    }
    this.scaleBase = this.scale;
    //绘制图片，参数含义分别为图像源，图片初始的x轴坐标，图片初始的y轴坐标，图片缩放后的宽度，图片缩放后的高度
    ctx.drawImage(image,this.imageX,this.imageY,this.lenAdapter(this.scale*this.imageNatrueW),this.lenAdapter(this.scale*this.imageNatrueH));
    this.renderBg(); 
}
this.image = image;
```

4.实现图片移动的事件  
图片的拖动操作由鼠标按下，鼠标移动，鼠标放下一系列操作实现。图片的边界检测规则为图片左侧坐标不能大于左侧边框，图片右侧坐标不能小于右侧边框，图片上侧坐标不能大于上侧边框，图片下侧坐标不能小于下侧边框，具体的代码实现如下所示。
```javascript
//鼠标按下事件
mousedownHandler (event) {
    //获取基准的x坐标
    this.baseX = event.pageX;
    //获取基准的y坐标
    this.baseY = event.pageY;
    //设定鼠标为按下状态
    this.mouseDownFlag = true;
},
//鼠标松开事件
mouseupHandler (event) {
    //设定鼠标为松开的状态
    this.mouseDownFlag = false;
},
mousemoveHandler(event) {
    //鼠标为按下状态时才执行移动
    if(this.mouseDownFlag){
        let ctx = document.getElementById('canvas').getContext('2d');
        //获取x轴的移动距离
        let offsetX = event.pageX - this.baseX;
        //获取y轴移动距离
        let offsetY = event.pageY - this.baseY;
        //这里的积数为移动灵活度的补正（补正参数可以控制移动图片的容易度）
        this.imageX += offsetX*1.5;
        this.imageY += offsetY*1.5;
        //图片左侧边缘检测 
        if(this.imageX>=this.aBorder){
            this.imageX = this.aBorder;
        }
        //图片右侧边缘检测
        if(this.imageX<=this.lenAdapter(this.borderLen + this.imageW - this.scale * this.imageNatrueW)){
            this.imageX=this.lenAdapter(this.borderLen+this.imageW - this.scale * this.imageNatrueW);
        }
        //图片上侧边缘检测
        if(this.imageY>=this.aBorder){
            this.imageY = this.aBorder;
        }
        //图片下侧边缘检测
        if(this.imageY <= this.lenAdapter(this.borderLen + this.imageH - this.scale * this.imageNatrueH)){
            this.imageY = this.lenAdapter(this.borderLen + this.imageH - this.scale * this.imageNatrueH);
        }
        this.baseX = event.pageX;
        this.baseY = event.pageY;
        //清除画布
        ctx.clearRect(0,0,this.canvasW,this.canvasH);
        //重新绘制图片
        ctx.drawImage(this.image,this.imageX,this.imageY,this.lenAdapter(this.scale*this.imageNatrueW),this.lenAdapter(this.scale*this.imageNatrueH));
        //绘制背景，后绘制的内容会显示在上面
        this.renderBg();
    }
}
```
5.控制图片缩放

```javascript
 changeScale (val) {
    let ctx = document.getElementById('canvas').getContext('2d');
    const precent = val / 100;
    this.precent = precent;
    //获取图片未放大前的宽度
    let beforeImageW = this.lenAdapter(this.scale * this.imageNatrueW);
    //获取图片未放大前的高度
    let beforeImageH = this.lenAdapter(this.scale * this.imageNatrueH);
    //获取放大比例
    this.scale = (1+precent) * this.scaleBase;
    //获取图片放大后的宽度
    let afterImageW = this.lenAdapter(this.scale * this.imageNatrueW);
    //获取图片放大后的高度
    let afterImageH = this.lenAdapter(this.scale * this.imageNatrueH);
    //设定x坐标为以图片中心放大
    this.imageX = this.imageX - (afterImageW - beforeImageW) / 2;
    //设定y坐标为以图片中心放大
    this.imageY = this.imageY - (afterImageH - beforeImageH) / 2;
    //图片左侧边缘检测 
    if(this.imageX>=this.aBorder){
        this.imageX = this.aBorder;
    }
    //图片右侧边缘检测
    if(this.imageX <= this.lenAdapter(this.borderLen + this.imageW - this.scale * this.imageNatrueW)){
        this.imageX = this.lenAdapter(this.borderLen+this.imageW - this.scale * this.imageNatrueW);
    }
    //图片上侧边缘检测
    if(this.imageY>=this.aBorder){
        this.imageY = this.aBorder;
    }
    //图片下侧边缘检测
    if(this.imageY <= this.lenAdapter(this.borderLen + this.imageH - this.scale * this.imageNatrueH)){
        this.imageY = this.lenAdapter(this.borderLen + this.imageH - this.scale * this.imageNatrueH);
    }
    //清除画布
    ctx.clearRect(0,0,this.canvasW,this.canvasH);
    //绘制图片
    ctx.drawImage(this.image,this.imageX,this.imageY,this.lenAdapter(this.scale*this.imageNatrueW),this.lenAdapter(this.scale*this.imageNatrueH));
    //绘制背景
    this.renderBg();
}

```
6.保存图片

```javascript
//保存图片
saveImg () {
    //上传图片为二倍图
    let canvas = document.createElement('canvas');
    //设置宽度为图片二倍大小
    canvas.setAttribute('width',this.imageW*2+'px');
    //设置高度为图片二倍大小
    canvas.setAttribute('height',this.imageH*2+'px');
    //样式仍然为一倍大小
    canvas.style.height = this.imageW + 'px';
    canvas.style.width = this.imageH + 'px';
    let context = canvas.getContext('2d');
    //获取裁剪掉的x坐标
    let sx = this.aBorder - this.imageX;
    //获取裁剪掉的y坐标
    let sy = this.aBorder - this.imageY;
    if(devicePixelRatio>=2){
        sx = sx / 2;
        sy = sy / 2;
    }
    let sWidth =  this.imageW;
    let sHeight = this.imageH;
    let dx = 0;
    let dy = 0;
    let dWidth = this.imageW;
    let dHeight = this.imageH;
    let precent = this.scalePrecent/100;
    //这里参数分别为图片源,在原图中裁剪掉的x坐标，在原图中裁剪掉的y坐标,裁剪框的宽度，裁剪框的高度，新生成图片距离画布的横坐标，新生成图片距离画布的纵坐标，画布的宽度，画布的高度
    context.drawImage(this.image,sx/(this.scale),sy/(this.scale),(sWidth)/(this.scale),(sHeight)/(this.scale),dx,dy,dWidth*2,dHeight*2);
    let data = canvas.toDataURL();
    let imageSrc = data;
    data=data.split(',')[1];
    data = atob(data);
    let ia = new Uint8Array(data.length);
    for (let i = 0; i < data.length; i++) {
        ia[i] = data.charCodeAt(i);
    };
    let blob=new Blob([ia],{type:"image/png"});
    //组装成表单
    let fd = new FormData();
    fd.append('file',blob);
    this.$emit('submitImg',{'isSuccess':true,image:{imageSrc,imgW:this.imageW,imgH:this.imageH}});
            this.scalePrecent = 1;
            this.$emit('closeDialog');
        }
```



