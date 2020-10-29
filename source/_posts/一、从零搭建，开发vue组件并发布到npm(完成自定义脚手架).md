---
title: 一、从零搭建，开发vue组件并发布到npm(完成自定义dev环境脚手架)
date: 2019-10-20 16:35:15
tags:
---

###  使用webpack打包工具，初始化项目。
本博客从零开始搭建项目，使用的打包工具是webpack，webpack官方文档地址[https://webpackjs.com](https://webpackjs.com)

```
mkdir vue-component-lyc
cd ./vue-component-lyc
npm init -y
```
初始化完成后，文件夹会生成一个pagejson文件
```
{
  "name": "vue-component-lyc", // 项目名称
  "version": "1.0.0", // 项目版本
  "description": "",
  "main": "index.js", // 默认入口文件
  "scripts": {
    "test": "echo \"Error: no test specified\" && exit 1"
  },
  "keywords": [],
  "author": "",
  "license": "ISC"
}
```
### 确定目录结构，编辑webpack打包配置。
首先，我们先创建一个build文件夹，用于放置webpack相关配置，build里分别创建webpack.base.js、webpack.dev.js和webpack.prod.js文件。

然后创建一个src文件夹，src里面再创建一个App.vue文件和一个mian.js打包入口文件。到此文件目录架构如下

{% asset_img QQ20191020-0.png This is an image %}

然后，我们需要考虑需要引入哪些依赖、插件。使用webpack，肯定需要引入webpack、webpack-cli、webpack-dev-server。
loader方面，先引入几个常用的loader: url-loader、css-loader、style-loader、babel-loader。这个项目需要打包处理.vue文件，所以需要引入vue-loader和vue, css处理器我比较喜欢使用sass，所以我还要引入sass-loader、node-sass。
插件方面，引入常用的html-webpack-plugin、clean-webpack-plugin, 因为我们项目打算把webpack开发环境配置webpack.dev.js和生产环境配置webpack.prod.js中的公用配置部分提取出来, 所以需要引入webpack-merge, 再引入webpack-simple-progress-plugin查看打包进程。

```
npm install webpack webpack-cli webpack-dev-server url-loader css-loader style-loader 
babel-loader vue-loader sass-loader node-sass html-webpack-plugin clean-webpack-plugin 
webpack-merge webpack-simple-progress-plugin --save-dev
npm install vue --save
```
在根目录下创建一个public文件夹，用于放置一些静态文件，然后建立一个模版文件index.html, 配合插件html-webpack-plugin使用

{% asset_img photo1026-2.jpg This is an image %}

```
<!DOCTYPE html>
<html lang="en">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <meta http-equiv="X-UA-Compatible" content="ie=edge">
  <title>Document</title>
</head>
<body>
  <div id="app"></div>
</body>
</html>
```
接下来开始参考webpack官方文档撸配置了

build/webpack.base.js中添加配置
```
const { CleanWebpackPlugin } = require('clean-webpack-plugin')
const SimpleProgressPlugin = require('webpack-simple-progress-plugin')
module.exports = {
  resolve: {
    extensions: ['.js', '.vue', '.json'],
  },
  module: {
    rules: [
      {
        test: /\.vue$/,
        loader: 'vue-loader'
      },
      {
        test: /\.js$/,
        loader: 'babel-loader'
      },
      {
        test: /\.css$/,
        loader: "style-loader!css-loader"
      },
      {
        test: /\.scss$/,
        use: [
          "style-loader",
          "css-loader",
          "sass-loader"
        ]
      },
      {
        test: /\.(png|jpg|gif)$/i,
        use: [
          {
            loader: 'url-loader'
          }
        ]
      }
    ]
  },
  plugins: [
    new CleanWebpackPlugin(),
    new SimpleProgressPlugin()
  ]
}
```
开发环境配置 webpack.dev.js
```
const merge = require('webpack-merge')
const common = require('./webpack.base.js')
const path = require('path')
const HtmlWebpackPlugin = require('html-webpack-plugin')

module.exports = merge(common, {
  mode: 'development',
  entry: path.resolve(__dirname, '../src/main.js'),
  output: {
    filename: "[name].[hash].main.[id].js",
    path: path.resolve(__dirname, '../dist/')
  },
  devServer: {
    hot: true,
    port: 9001,
    stats: 'minimal'
  },
  plugins: [
    new HtmlWebpackPlugin({
      template: './public/index.html'
    })
  ]
})
```
### 添加测试App.vue，本地运行测试。
将App.vue文件引入main.js中，对vue源码不了解的小伙伴，可以看这个文档[https://ustbhuangyi.github.io/vue-analysis/](https://ustbhuangyi.github.io/vue-analysis/)

测试页面App.vue
```
<template>
  <div class="app">
    <p>测试页面</p>
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
开发环境入口文件 src/main.js
```
import Vue from 'vue'
import App from './App.vue'

new Vue({
  el: '#app',
  render: h => h(App)
})
```
pageage.json添加运行命令
```
"scripts": {
  "serve": "webpack-dev-server --config build/webpack.dev.js"
},
```
目前文件目录

{% asset_img QQ20191020-1.png This is an image %}

接下来, npm run serve跑一下， 出来的肯定是一堆报错，哈哈哈哈哈哈，我们一个个去解决它吧
```
Error: Cannot find module '@babel/core'
 babel-loader@8 requires Babel 7.x (the package '@babel/core'). 
 If you'd like to use Babel 6.x ('babel-core'), 
 you should install 'babel-loader@7'
```
根据报错提示，我们可以选择npm install @babel/core --save-dev, 继续跑，又报错了
```
RROR in ./src/App.vue
Module Error (from ./node_modules/vue-loader/lib/index.js):
[vue-loader] vue-template-compiler must be installed as a peer dependency, 
or a compatible compiler implementation must be passed via option
s.
```
提示中表明，安装vue-loader的同时也要安装vue-template-compiler这个依赖，继续 npm install vue-template-compiler --save-dev， 继续跑，又报错了。
```
ERROR in ./src/App.vue
Module Error (from ./node_modules/vue-loader/lib/index.js):
vue-loader was used without the corresponding plugin. Make sure to include 
VueLoaderPlugin in your webpack config.
```
vue-loader必须在相应插件VueLoaderPlugin下才能正常使用，需要在webpack配置中加上它, Vue-Loader使用文档中也有特别提到[https://vue-loader.vuejs.org/zh/guide/#%E6%89%8B%E5%8A%A8%E8%AE%BE%E7%BD%AE](https://vue-loader.vuejs.org/zh/guide/#%E6%89%8B%E5%8A%A8%E8%AE%BE%E7%BD%AE)
```
//  webpack.base.js中添加(不要直接粘贴代码，在对应为止添加即可)
const VueLoaderPlugin = require('vue-loader/lib/plugin')

plugins: [
  new VueLoaderPlugin()
]
```
继续运行 成功了， 第一步完成。

{% asset_img QQ20191020-0.jpg This is an image %}
