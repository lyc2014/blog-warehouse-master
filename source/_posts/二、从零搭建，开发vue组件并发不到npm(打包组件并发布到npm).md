---
title: 二、从零搭建，开发vue组件并发不到npm(打包组件并发布到npm)
date: 2019-10-22 00:07:55
tags:
---

### 开发vue插件
如何开发vue插件呢，可以直接参考vue官方文档[https://cn.vuejs.org/v2/guide/plugins.html#开发插件](https://cn.vuejs.org/v2/guide/plugins.html#开发插件)。Vue.js的插件应该暴露一个install方法。这个方法的第一个参数是Vue构造器，第二个参数是一个可选的选项对象：
```
MyPlugin.install = function (Vue, options) {
  // 1. 添加全局方法或属性
  Vue.myGlobalMethod = function () {
    // 逻辑...
  }

  // 2. 添加全局资源
  Vue.directive('my-directive', {
    bind (el, binding, vnode, oldVnode) {
      // 逻辑...
    }
    ...
  })

  // 3. 注入组件选项
  Vue.mixin({
    created: function () {
      // 逻辑...
    }
    ...
  })

  // 4. 添加实例方法
  Vue.prototype.$myMethod = function (methodOptions) {
    // 逻辑...
  }
}
```
看完了可以开始整活了，我们在src下面建立一个lib文件夹，在文件夹里创建index.js和test.vue。

创建一个test.vue测试组件，无任何功能，仅为了测试如何开发插件。
```
<template>
  <div class="test">
    <p class="red">{{name}}</p>
  </div>
</template>
<script>
export default {
  name: 'test',
  data () {
    return {
      name: '测试组件'
    }
  }
}
</script>
<style lang="scss" scoped>
.test{
  .red{
    color: red;
  }
}
</style>
```
创建一个index.js， 引入test.vue，并且添加install方法。
```
import test from './test.vue'
const testComponent = {
  install(Vue, options) {
    Vue.component(test.name, test)
  }
}
if (typeof window !== 'undefined' && window.Vue) {
  window.Vue.use(testComponent)
}
export default testComponent
```
内容写好，接下来我们需要添加生产环境下的打包配置, 往build/webpack.prod.js中添加配置
```
const merge = require("webpack-merge")
const common = require('./webpack.base.js')
const path = require('path')

module.exports = merge(common, {
  entry: path.resolve(__dirname, '../src/libs/index.js'),
  output: {
    filename: 'vue-component-lyc.js',
    path: path.resolve(__dirname, '../dist/'),
    libraryTarget: 'umd'
  },
  stats: 'minimal',
  mode: 'production'
})
```
在package.json添加命令
```
"scripts": {
  "build": "webpack --config build/webpack.prod.js"
 }
```
npm run build 进行打包后，dist文件夹下输出文件vue-component-lyc.js

{% asset_img 11571678038_.pic_hd.jpg This is an image %}

### dev环境下对插件进行本地测试

我们main.js中全局引入dist目录下的vue-component-lyc.js，在app.vue中使用这个插件。
```
import Vue from 'vue'
import App from './App.vue'
import test from '../dist/vue-component-lyc'
Vue.use(test)

new Vue({
  el: '#app',
  render: h => h(App)
})
```
然后在App.vue中使用这个插件。
```
<template>
  <div class="app">
    <p>测试页面</p>
    <test></test>
  </div>
</template>
<script>
export default {
  data () {
    return {

    }
  }
}
</script>
<style lang="scss" scoped>

</style>

```
npm run serve运行一下可以看到这个组件成功的引入使用了

{% asset_img 21571678433_.pic.jpg This is an image %}

好了，本地测试是ok的。

### 发布到npm
修改、补充package.json内容准备发布，主要是入口这里要改好。
```
{
  "main": "dist/vue-component-lyc.js"
}
```
然后登录账号
```
$ npm adduser
Username: YOUR_USER_NAME
Password: YOUR_PASSWORD
Email: YOUR_EMAIL@domain.com
```
发布
```
npm publish
```
完事儿。