---
title: 澳牧会员中心前端开发总结
date: 2018-01-23 16:23:02
tags: worklog
---
1. less定义变量   
使用less这种声明变量的方式可以将一些常用的变量提前定义，这里定义了系统使用的主体色，以便更好的管理整个项目的样式文件。需要注意的当使用@import 引入less文件时，**文件路径使用webpack的别名时需要在文章路径前加上~**，当想使用某个变量时，需要在具体的vue文件中的style标签中引入该定义变量的文件.
2. 布局  
后台系统的单页面布局，通常头部包含导航栏，左侧为侧边栏，这些都为固定不动的内容，而右侧的视口的宽度和高度都会发生变化。总体上可以分为左右布局，右侧再分为上下布局，absoulte布局实现下方视口，为其设置top,right,bottom,left,具体结构如下所示


<html>
<div style="width:260px;height:200px;background-color:#555555;display:flex;font-size:12px;margin-left:80px;"><div style="background-color:#777777;float:left;width:50px;height:200px;line-height:200px;text-align:center;">侧边栏</div><div style="width:210px;height:200px;"><div style="height:40px;line-height:40px;background-color:#DDDDDD;text-align:center;color:#000;">顶部导航栏</div><div style="height:160px;line-height:160px;background-color:#666666;text-align:center;">视口</div></div>

</div>
</html>

**注意:若想设定视口存在一个min-width使其内部产生一个滚动条，需要在视口内嵌套一层div，将该div设置一个min-width对外部不会有影响，想设置一个元素宽度超过视口从而触发底部滚动条这种情况必须要给外部容器设置一个具体的宽度，有时候想用内部列表给外部容器撑开，而不给外部容器设置宽度，这种做法通常情况下是不可取的，需要计算内部列表的宽度赋值给外部容器**

3.在页面上存在弹出层时，在弹出层上滑动时仍旧会触发底部的滑动，这种情况在弹出层中也存在滑动层时当滑动弹出层上的滑动层是就会导致弹窗上的滑动层和下部的弹出层都会滑动，所以就很不友好，这里先使用当出现弹出层时会将viewprot下第一级div的overflow设置为hiddn，从而使底层不能滚动，实现的关键代码如下所示:
```javascript
document.getElementById('routerViewPort').style = 'overflow:hidden';
```
这样实现以后由于突然从overflow为scroll状态变为hidden状态会导致某些浏览器中的右侧滚动条突然消失，这样会导致页面有明显的15像素左右（不同浏览器下的滚动条的宽度会有差别）的右移，这样的用户体验仍然说不上好，经过资料查询查到了一种解决方案（[传送门](http://yujiangshui.com/review-how-to-make-popup-mask-effect/)），该解决方法为在设置不允许滑动的同时并且在页面右侧增加一个padding，长度保持和之前滚动条的宽度一致，实现的关键代码如下所示
```javascript
//获取包含滚动条部分的宽度
let offset = document.getElementById('routerViewPort').offsetWidth;
//获取不包含滚动条部分的宽度
let client = document.getElementById('routerViewPort').clientWidth; 
//获取滚动条的宽度
let scrollW = offset - client;
//由于之前右侧20像素的填充，所以这里的右侧填充为20像素加上滚动条宽度
document.getElementById('routerViewPort').style = `overflow:hidden;padding:20px ${20+scrollW}px 20px 20px;`;
```

4.关于页面组件的划分，这次的页面组件划分是我在设计前端页面时候划分的，总共划分了主表格页面部分，查询表单部分，标签部分，弹出选框层，在结构上都属于比较独立所以区分开，由于查询表单和主表格页面信息交换很多，所以使用了vuex来进行状态管理，汲取本次经验下次开发中应当先从数据层开始梳理，小型开发可以避免拆解过细导致系统复杂度提高的问题，关于使用vuex的操作如下所示。
```javascript
//member.js
//mutation方法有权直接操作state上的值,该方法中不允许包含异步操作,若在mutation之外对state的值进行更改将会报出错误
[MEMBER.ADD_TO_TAG] (state,tag) {
    state.tags.push(tag);
}
//action方法用于封装一系列的mutation方法组成一个公共的业务逻辑，注意这里只允许包含两个参数
addToTag ({commit},tag){
    //使用commit调用mutation方法
    commit(MEMBER.ADD_TO_TAG);
}
```

```javascript
//getters.js
//在getters中将需要报出给外部使用的state属性报出
export default {
    tag:state => state.member.tag
}
```

```javascript
//selectTag.vue
//在计算属性中声明该值是为了可以随时监听到该值的变化
computed:{
    ...mapGetters(['tag'])
},
methods:{
    addTag(tag){
        //在方法中就可以直接获取到该tag的值
        console.log(this.tag);
        //若想改变tag的值是不可以直接赋值的，这里我们使用已经定义好的action
        this.$store.dispatch('addToTag',tag);
    }
}
```

由此可见使用vuex进行状态管理的过程还是比较复杂的，开发者需要根据实际情况酌情进行考虑那些状态需要进行管理。关于更多有关vuex的介绍可以参考官网([传送门](https://cn.vuejs.org/v2/guide/state-management.html))

5.模版渲染需要注意的内容  
关于其模版渲染部分只改变数组中的特定值时是不会触发vue模版的重新渲染,需要使用
**Vue.set(属性名,想改变内容的索引值,改变后的内容)**
来触发模版的重新渲染,
**需要特别注意的是,computed(计算属性)中检测的属性中的值为一个数组时只改变数组中某一个值也不会引起重新渲染，需要将值重新初始化一下，比如初始化为[]**

6.vue中自定义动画  
vue自定义动画实现也是比较容易出错的地方，这里简单总结一下vue自定义动画。
```html
<!--transition标签用于自定义动画,name属性为自定义动画名称 -->
<transition name="fade">
    <!--需要设置过渡动画的容器需要设置v-show或者v-if来控制其显隐 -->
    <div v-show="flag"></div>
</transition>
```

```css
/*enter系列为在容器show时触发的动画，fade(根据自定义动画名决定)-enter用于设置动画第一帧的状态，也就是动画的初始状态*/
.fade-enter{
    opacity:0;
    transform:translateY(-30px);
}
/*fade-enter-to用于设定动画最后一帧的状态，也就是动画结束时的状态*/
.fade-enter-to{
    opacity:1;
    transform:translateY(0);
}
/*.fade-enter-active在整个动画执行过程中存在,可以在这里设置动画时长以及贝塞尔曲线等*/
.fade-enter-active{
    transition: all .4s;
}
/*leave系列为在容器hide时触发的动画，于enter系列如出一辙*/
.fade-leave{
    transform:translateY(0);
    opacity:1;
}
.fade-leave-to{
    transform:translateY(-30px);
    opacity:0;
}
.fade-leave-active{
    transition: all .4s;
}

```

7.vue中关于nextTick的使用  
有时候我们想获取经过动态渲染过的列表的高度,我们操作dom元素获取高度时会发现我们获取到的是动态渲染前的高度，这是因为我们获取高度的动作发生在动态渲染之前，为了保证获取高度的动作 在动态渲染之后发生，我们需要将该方法嵌套在nextTick中，该方法在所有同步方式执行后，所有异步方法执行前执行，事件轮询机制也是一个比较重要的基础知识，可以参考阮一峰老师的[javaScript 运行机制详解：再谈Event Loop](http://www.ruanyifeng.com/blog/2014/10/event-loop.html)
```javascript
//动态渲染后获取容器的高度
this.$nextTick(()=>{
    let el = this.$refs['list-container'];
    this.tagPannelH = el.getBoundingClientRect().height - this.marginFix;
    });
```

8.关于深浅拷贝问题       
在js中关于引用属性比如**对象和数组在使用=进行赋值时实际上是指向了同一个引用**，所以对赋值前后的对象进行操作时最终都会对引用的对象产生影响，这种赋值操作也就是浅拷贝过程，在编码过程中由于浅拷贝会产生一些匪夷所思的问题，关于该操作一定要多多注意，深拷贝可以通过以下方式实现。
```javascript
//对象的深拷贝
let newObj = Object.assign({},oldObj);
//数组的深拷贝
let newArr = oldArr.slice(0);
let newArr = oldArr.concat([]);
```

9.elementUI的一些坑  
(1).elementUI提供跟分页请求相关的事件只有@size-change（每一页显示数量变化时触发的事件），@current-change（为当前页数发生改变时触发的事件），在该系统中我在点击查询按钮时我会默认请求第一页的数据，在进行页面跳转时也会请求该页的数据，所以会产生比如我当前浏览的是第三页，当点击查询时首先由于点击事件请求一次第一页的数据，页面又从第三页跳转到了第一页又会触发current-change请求一次数据，导致二次请求的bug，这里我增加了一个flag控制变量，也就是说当点击查询按钮时将flag设置为true，在分页跳转时会判断若flag为false再查询，并且执行完毕后将该flag置为false（这里也可以将查询全部放在分页中解决这个问题），这样又出现了另一个问题，当我们在第一页时我们点击查询按钮由于这时还是第一页并没有执行current-change所以flag没有被置为false导致点击分页时不会再请求数据，为了解决这个问题我在查询时如果当前是第一页时跳转到第零页，由于分页机制的影响，又会重新跳转到第一页会触发current-change,这种方法还是不太好，需要继续关注一下这个问题。

(2).element-ui提供两种弹出面板，一个是生成在body下，另一种是生成在当前的父级下，在使用我们这种视口结构时，如果生成在body下会出现蒙版在弹窗之上的bug，若要解决这个bug需要将弹窗组件在视口外加载，如果在父级生成当存在第二级弹出层时仍然会存在这个bug,**这时需要在子父弹窗中都加入append-to-body属性**  

(3).element-ui表格当想换色时我们可以用官方提供的增加style的方法，但是始终找不到表格最后一条边框设置颜色的位置，最后找到位于table的bfore的伪类中，这一点需要注意
```less
.score-table{
    /*需要改变伪类为改变框改变颜色*/
    &::before{
        background-color:@border-gray;
    }
}
```











