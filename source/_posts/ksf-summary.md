---
title: 康师傅红包系统开发总结
date: 2018-02-22 14:14:08
tags: worklog
---

1.关于前后端分离  
在本次项目中采用了前后端分离方式，在项目部署上彻底分离，前端作为一个独立的项目单独存在，前端项目使用vue-cli生成。  
2.使用vue-cli的webpack方式生成项目需要注意的事情  
若项目中使用css框架，如less,sass等，需要自己手动安装
```bash
npm install less less-loader --save-dev
```
3.在webpack中使用别名  
若想在webpack中使用别名，需要在build文件夹中的webpack.base.conf.js中resolve.alias中增加配置以下内容
```javascript
 resolve: {
    //可以给常用的文件目录配置别名
    alias: {
      'vue$': 'vue/dist/vue.esm.js',
      '@': resolve('src'),
      'assets':resolve('src/assets'),
      'components':resolve('src/components'),
      'config':resolve('src/config'),
      'api':resolve('src/api')
    }
}
```
在js中使用别名引入文件
```javascript
//使用别名快速引入文件
import api from 'api/activity.js';
import addBtn from 'components/common/addBtn.vue';
import aListForm from 'components/activity/aListForm.vue';
import comTable from 'components/common/comTable.vue';
import comData from 'assets/data/com_table.json';
import giftPannel from 'components/activity/giftPannel.vue';
```
在样式中使用别名引入文件
```html
<style lang="less" scoped>
/**这里使用~表明使用别名**/
@import '~assets/style/var.less';
</style>
``` 
4.IE不支持ES6的原生promise
由于IE不支持原生的promise，所以引入babel-ployfill模块对其进行转义处理。
```bash
npm install babel-ployfill --save-dev
```
然后在build文件夹中的webpack.base.conf.js中entry.app中增加babel-ployfill处理
```javascript
entry: {
    app: ['babel-polyfill','./src/main.js']
 }
```
5.关于使用element-ui自定义主题色  
element-ui默认提供的是蓝色的主题色，而有时候我们需要使用其他颜色的主题色，关于主题色相关的内容可以查看[官方文档](http://element-cn.eleme.io/#/zh-CN/component/custom-theme) ，这里我使用了element-ui官方提供的[在线主题生成工具](https://elementui.github.io/theme-chalk-preview/#/zh-CN)，在此处生成好主题色后即可下载。我将下载后的css文件放在assets/style中。最后在main.js中引入主题色
```javascript
//main.js
import './assets/style/element-#EA9235/index.css';
```
6.公共css样式策略  
在我们做项目时需要使用全局的css样式，其中我创建一个文件存放公共样式变量，一个文件存储公共样式类。
公共变量文件只需在需要使用的文件引用
```
<style lang="less" scoped>
@import '~assets/style/var.less';
    .form-group{
        margin:30px 0 30px @form-indent;
        width:540px;
    }
    .complex-input{
        color:@sub-font-color;
        margin-bottom:12px;
        &:last-child{
            margin-bottom:0;
        }
    }
</style>
```
公共类文件需要全局引入，这样在每个文件都可以设定公共类名引入全局样式
```javascript
//main.js
//引入全局样式文件
import './assets/style/index.less';
```
7.系统布局中需要注意的事情 

在系统布局中，我想在右侧视口实现有一个背景边框，并且最小高度为100%。最开始我的想法是做一个嵌套的div,在外层设置padding,内部容器为主视口，主要的样式如下所示
```
.outer{
    box-sizing:border-box;
    padding:10px;
    min-height:100%
    .inner{
        min-height:100%
    }
}
```
但是经过尝试发现min-height无法使用该方式继承。

经过实验，最终使用了一个div解决了该问题，具体样式如下所示
```
.viewport{
    /*防止出现横向滚动条时最右侧边框消失*/
    display:inline-block;
    /*设置最小宽度*/
    min-width:1150px;
    /*设置最大宽度*/
    max-width:1700px;
    /*～用于避免less转义*/
    /*使用css3新属性calc设置宽度*/
    width:calc(~"100% - 28px");
    margin:14px;
    background:#fff;
    min-height:calc(~"100% - 28px");
    box-shadow:#ccc 1px 1px 8px;
    border-radius:6px;
    /*防止出现横向滚动条时最右侧边框消失*/
    overflow:hidden;
}
```
注意这里解决防止出现横向滚动条时最右侧边框消失的方式。

8.vue中几个小的知识点  
**关于emit**  
emit并不会阻止后续的执行内容，这一点和return有所区别，请注意。  
**关于自定义事件的外部传参**    
有时我们需要遍历自定义组件，并给他们的自定义事件传入索引值，而这个事件本身有返回值，这时就需要用到自定义组件的外部传参，具体用法如下所示：  
组件内部定义自定义事件
```javascript
//inner.vue
this.$emit('myEvent',true);  
```
外部传参
```html
<template>
    <!--注意这里的写法,$event即为内部传出的参数，再后面可以跟上外部传参的参数 -->
    <inner @myEvent="eventHandler($event,key)"></inner>
</template>
<script>
import inner from './inner.vue';
export default {
    data () {
      return {
          key:10
      }  
    },
    components:{
        inner
    },
    methods:{
        eventHandler (flag,key) {
            //true
            console.log(flag);
            //10
            console.log(key);
        }
    }
}
</script>
```
**slot的用法**  
slot即为插槽，当我们定义一个组件希望有一部分html结构希望由外部传入时我们就可以使用slot,详细内容可以查看[官方文档](https://cn.vuejs.org/v2/guide/components.html#%E4%BD%BF%E7%94%A8%E6%8F%92%E6%A7%BD%E5%88%86%E5%8F%91%E5%86%85%E5%AE%B9),在本案例中我在自定义表格插件时使用到了插槽功能，使其可以自定义表格中的内容。  
9.element-ui需要注意的问题  
**element-ui表单验证**
element-ui的基本表单验证可以查看[官方文档](http://element-cn.eleme.io/#/zh-CN/component/form)，这里着重说明一下动态生成的表单验证
```html
<el-form :model="dynamicValidateForm" ref="dynamicValidateForm" label-width="100px" class="demo-dynamic">
  <el-form-item
    prop="email"
    label="邮箱"
    :rules="[
      { required: true, message: '请输入邮箱地址', trigger: 'blur' },
      { type: 'email', message: '请输入正确的邮箱地址', trigger: 'blur,change' }
    ]"
  >
  <el-input v-model="dynamicValidateForm.email"></el-input>
  </el-form-item>
  <!--这里注意动态生成的表单验证最重要的是需要有prop,rule使用内联的方式定义 -->
  <el-form-item
    v-for="(domain, index) in dynamicValidateForm.domains"
    :label="'域名' + index"
    :key="domain.key"
    :prop="'domains.' + index + '.value'"
    :rules="{
      required: true, message: '域名不能为空', trigger: 'blur'
    }"
  >
    <el-input v-model="domain.value"></el-input><el-button @click.prevent="removeDomain(domain)">删除</el-button>
  </el-form-item>
  <el-form-item>
    <el-button type="primary" @click="submitForm('dynamicValidateForm')">提交</el-button>
    <el-button @click="addDomain">新增域名</el-button>
    <el-button @click="resetForm('dynamicValidateForm')">重置</el-button>
  </el-form-item>
</el-form>
```

在此次项目中想添加的表单验证部分结构如下所示

```html
<el-form ref="giftForm" :model="formData">
    <div :key="key" v-for="(item,key) in formData.price">
        <!--...-->
        <!--红包奖 -->
        <template v-if="item.type === 0">
            <div class="item-group">
                <div class="item-content">
                    <el-form-item label-width="60px" :key="key" :prop="'price.' + key + '.money'" :rules="{required: true, message: '金额不能为空', trigger: 'blur'}">
                        <h3 class="item-label" slot="label">金额</h3>
                        <el-input size="small" type="number" required v-model="item.money" style="width:60px;margin:0 6px;"></el-input>元
                    </el-form-item>
                </div>
            </div>
        </template>
    </div>
</el-form>
```

需要注意的是，prop属性写法为element-ui提供的官方写法，还有一点需要注意的是<el-form-item></el-form-item>不能嵌套，**嵌套后会使表单验证失效**。  
**el-dialog插件需要注意的内容**  
在使用el-dialog经常会将控制dialog显隐的flag作为参数传入，但是我们要注意外部传入的参数是不允许在内部改变，需要交给外部改变，这里我们要注意所有交给外部改变的时机。关于dialog的配置如下所示：
```html
<el-dialog :visible.sync="isShow" append-to-body width="920px" :before-close="dialogCloseBefore" @close="closeDialog">
</el-dialog>
```
需要注意的是before-close设置的是关闭前回调的函数，close相应的是关闭dialog的事件，在这两个函数都需要交给外部关闭dialog,具体内容如下：
```javascript
dialogCloseBefore (done) {
        this.$confirm('确定要放弃对奖品的操作吗','提示',{
            confirmButtonText: '确定',
            cancelButtonText: '取消',
            type: 'warning'
        }).then(()=>{
            //这里不使用默认的done()，因为会在内部改变参数从而报错，这里手动暴露到外面关闭
            this.$emit('close',false);
        });
    },
    closeDialog () {
        this.$emit('close',false);
    },
```
其他部分需要自定义关闭表单也需要手动暴露到外面改变。  
**element-ui函数作为属性传参问题**  
element-ui里有很多组件的参数是把函数作为属性传入是不能使用上述所说的外部传参的方式的，这里可以使用返回一个函数解决该问题，实例如下所示：
```
<!--before-upload为上传前调用的函数，这里传入索引 -->
<el-upload class="upload" v-else drag action="" :before-upload="uploadImage(key)"
    <i class="el-icon-upload"></i>
    <div class="el-upload__text">将文件拖到此处，或<em>点击上传</em></div>
    <div class="el-upload__tip" slot="tip">只能上传jpg/jpeg/png文件，且不超过2mb</div>
</el-upload>
```
定义的具体函数
```javascript
uploadImage (key) {
    //返回具体的函数，在函数中可以使用传入的key值
    return (file) => {
        if(file.type !== 'image/jpeg' &&  file.type !== 'image/jpg' && file.type !== 'image/png'){
            this.$message.error('上传头像图片只能是 JPG/JPEG/PNG 格式!');
            return false;
        }
        if(file.size / 1024 / 1024 > 2){
            this.$message.error('图片不能超过2MB!');
            return false;
        }
        let reader = new FileReader();
        reader.readAsDataURL(file);
        reader.onload = () => {
            this.$nextTick(()=>{
            this.imageSrc = reader.result;
            this.index = key;
            this.dialogVisible = true; 
            });
        }
        return false;
    }
}
```
10.一些其他知识点  
**上传文件相关**  
自定义上传文件
若我们想定义一个上传图片功能的按钮，可以将按钮上覆盖一个隐藏的上传图片的input框，关于input框样式设计如下所示：
```less
.upload-file{
    position:absolute;
    left:0;
    top:0;
    right:0;
    bottom:0;
    opacity:0;
    z-index:101;
    filter:alpha(opacity=0);
    cursor:pointer;
}
```
获取上传文件的参数  
这里我们以案例中的上传文件的函数为例分析
```javascript
changeFile (event,key) {
    //获取file文件
    let file = event.target.files[0];
    //判断file文件类型
    if(file.type !== 'image/jpeg' &&  file.type !== 'image/jpg' && file.type !== 'image/png'){
        this.$message.error('上传头像图片只能是 JPG/JPEG/PNG 格式!');
        return false;
    }
    //判断file文件大小
    if(file.size / 1024 / 1024 > 2){
        this.$message.error('图片不能超过2MB!');
        return false;
    }
    let reader = new FileReader();
    //读取图片base64的url
    reader.readAsDataURL(file);
    //读取完毕后调用的回调函数
    reader.onload = () => {
        this.$nextTick(()=>{
        //获取图片base64的url
        this.imageSrc = reader.result;
        this.index = key;
        this.dialogVisible = true;
        });
    } 
}
```

**隐藏数字输入框的向上，向下按钮**  
我们在使用type=number的input框时会有默认的向上，向下按钮，而我们并不需要这种按钮，需要reset这种样式，重置样式如下所示
```less
/*chrome清除样式*/
input::-webkit-outer-spin-button,
input::-webkit-inner-spin-button{
    -webkit-appearance: none !important;
    margin: 0; 
}
/*火狐清除样式*/
input[type="number"]{-moz-appearance:textfield;}
```
**始终使滚动条位于最底部**  
当我们添加动态项时希望滚动条始终跟随新出现的动态项出现在最底部，实现该功能的代码如下所示
```javascript
//获取scroll的dom
let sc = this.$refs.scrollContainer;
//使滚动条总是位于最下方
sc.scrollTop = sc.scrollHeight;
```
