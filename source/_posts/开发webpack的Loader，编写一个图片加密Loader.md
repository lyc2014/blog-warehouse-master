---
title: 开发webpack的Loader，编写一个图片加密Loader
date: 2019-11-1 16:24:15
tags:
---

### 前言
为了更了解webpack的打包过程，我决定写一个小demo，写一个加密图片的打包Loader，实现的功能是打包的时候加密图片资源，网页上加载完图片资源后再进行解密，可以有效**防止网页上的图片被直接下载窃取**, [npm地址](https://www.npmjs.com/package/img-hide)

{% asset_img gif1.gif This is an image %}

### Loader开发思路

官方文档中有开发loader的[教程](https://webpack.js.org/contribute/writing-a-loader/)

```
A loader is a node module that exports a function.
This function is called when a resource should be transformed by this loader.
The given function will have access to the Loader API using the this context provided to it.
```

这里提到了webpack的loader是一个导出为函数的node模块, 函数的入参便是源资源内容，函数中可以对这些内容进行处理。所以一个最简单的loader可以这样写:

```
module.exports = function(source) {
  return source;
};
```

Loader的具体开发规范可以直接看[官网](https://webpack.js.org/contribute/writing-a-loader/), 本文章主要讲一下Loader的实现过程。

{% asset_img gif2.gif This is an image %}

思路: 利用loader的链式调用规则，在url-loader处理图片资源前，先用加密Loader对图片进行处理，所以loader的入参是图片资源（二进制数据资源），返回的资源能够让url-loader进行处理，所以返回的也是一个图片资源（二进制资源），整个过程处理和传递的都是二进制数据。为了方便使用，配置项里保留两个可选项，需要加密的路径(path)和不需要加密的图片白名单(whiteList)。

### 实现过程

创建项目
```
npm init img-hide -y
```

安装需要用到的Loader和插件: webpack webpack-cli url-loader file-loader和loader-utils, 其中url-loader依赖于file-loader。 loader-utils是开发loader中非常好用的插件。
```
npm install --save-dev webpack webpack-cli url-loader file-loader loader-utils
```
创建loader.js文件，引入一张正常图片logo.png, 创建一个入口文件mian.js, 创建webpack.config.js打包配置文件。入口文件main.js引入图片。

```
import img from './logo.png'
```
编写webpack打包配置

```
const path = require('path')
module.exports = {
  mode: 'development',
  entry: './main.js',
  output: {
    filename: 'out.js',
    path: path.resolve(__dirname, 'dist')
  },
  module: {
    rules: [
      {
        test: /\.(png|jpg|gif)$/i,
        use: [
          {
            loader: 'url-loader',
            options: {
              limit: 10
            }
          },
          {
            loader: path.resolve(__dirname, './loader.js'),
          }
        ],
      },
    ],
  }
}
```
写一个最简单的Loader不加任何处理
```
module.exports = function(source) {
  console.log('source', source)
  return source;
};
module.exports.raw = true
```
将module.exports.raw = true，source会以 raw Buffer 的形式传入到 loader 中了，将这个设置好。

package.json添加build命令

```
"scripts": {
  "build": "webpack"
}
```
npm run build 进行打包测试loader, 可以看到source被打印出来了

{% asset_img jpg2.jpeg This is an image %}

根据前面的需求，下面对loader的资源进行加密处理， loader.js进一步完善。

```
const { getOptions } = require('loader-utils')
const path = require('path')
module.exports = function (souce) {
  //  获取 loader 的配置参数 options
  const options = getOptions(this)
  //  不属于 options.path 目录下的图片不进行加密处理
  if (options.path && this.resourcePath.indexOf(options.path) === -1) {
    return souce
  }
  /**
   * 路径匹配，但是不在options.whiteList白名单里面的不进行加密处理
   * 不配置options.whiteList白名单项，则匹配路径下的图片全部进行加密处理
   */
  if (options.path && this.resourcePath.indexOf(options.path) === 0) {
    if (options.whiteList && Array.isArray(options.whiteList)) {
      var name = ''
      try {
        name = path.basename(this.resourcePath)
      } catch {
        console.log(this.resourcePath + '的格式不正确')
      }
      if (options.whiteList.indexOf(name) === -1) {
        return souce
      }
    }
  }
  var src = []
  for (let i = souce.length; i > 0; i--) {
    src.push(souce[i - 1])
  }
  var buf = Buffer.from(src)
  return buf
}
module.exports.raw = true
```
webpack可以进一步进行配置

```
use: [
  {
    loader: 'url-loader',
    options: {
      limit: 10
    }
  },
  {
    loader: path.resolve(__dirname, './loader.js'),
    options: {
      path: path.resolve(__dirname, './'),
      whiteList: ['logo.png']
    }
  }
],
```
再npm run build 打包一遍， 生成的图片已经是被加密过裂开的图片了。

{% asset_img jpg3.jpeg This is an image %}

{% asset_img gif3.gif This is an image %}