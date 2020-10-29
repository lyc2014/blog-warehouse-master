---
layout: mock
title: 在vue中使用 mock + axios 实现http请求函数封装
date: 2019-06-16 13:07:12
tags:
---
axios使用的是promise，使用和维护起来都非常方便，所以我选择它。
mockjs使用起来也很简单，不需要搭建mock服务，虽然没有easymock那么简便，但是比easymock官方在线服务稳定，比本地部署简便，所以我选择mockjs。
开发项目的时候，后端还没写好接口是，前端可以根据文档预先定好的字段写出对应的mock数据，模拟接口数据，这样会节省很多的对接时间。接下来我会
结合axios和mock封装出一个比较好用的函数。demo的git地址是[https://github.com/lyc2014/template](https://github.com/lyc2014/template)。

使用vue.cli3搭建项目，引入axios、mockjs。
```
npm install -g @vue/cli
vue create template
cd template
npm install axios mockjs --save
```
### 使用axios
在src目录下创建一个axios目录，建立一个index.js, 简单封装一个请求函数。axios的使用文档[https://github.com/axios/axios](https://github.com/axios/axios)
```
// axios/index.js
function myAxios (url, postdata, method = 'get') {
  return new Promise((resolve, reject) => {
      axios({
          url: url,
          postdata,
          method
      }).then(res => {
          resolve(res)
      }).catch(error => {
          reject(error)
      })
  })
}
export default myAxios
```
在src/main.js中引入这个myAxios，并将其挂载在vue的prototype上方便全局调用。
```
import myAxios from '@/axios/index.js'
Vue.prototype.$axios = myAxios
```

接下来在vue文件中使用豆瓣电影的api测试一下。
```
mounted () {
  this.$axios('https://m.douban.com/rexxar/api/v2/muzzy/columns/10018/items', {
    start: 0,
    count: 3
  }).then(res => {
    console.log(res)
  })
}
```
打印内容
```
Access to XMLHttpRequest at 'https://m.douban.com/rexxar/api/v2/muzzy/columns/10018/items' from origin 'http://localhost:8080' has been blocked by CORS policy: No 'Access-Control-Allow-Origin' header is present on the requested resource.
```
这是一个跨域资源限制问题，我们可以利用devServer做一个代理映射去解决这个问题，在vue.config.js中添加一下配置(没有此文件可在项目根目录下建立一个),vue.cli3的配置文档[https://cli.vuejs.org/zh/config/](https://cli.vuejs.org/zh/config/)。
```
// vue.config.js
module.exports = {
    devServer: {
        proxy: {
            '^/api': {
                target: 'https://m.douban.com/rexxar',
                changeOrigin: true
            }
        }
    }
}
```
修改App.vue中请求的url地址
```
mounted () {
  this.$axios('/api/v2/muzzy/columns/10018/items', {
    start: 0,
    count: 3
  }).then(res => {
    console.log(res)
  })
}
```
重新npm run serve 运行一下，能拿到数据了。
```
config: {url: "/api/v2/muzzy/columns/10018/items", method: "get", headers: {…}, transformRequest: Array(1), transformResponse: Array(1), …}
data: {count: 20, start: 0, total: 16, items: Array(16)}
headers: {date: "Tue, 29 Oct 2019 06:25:31 GMT", strict-transport-security: "max-age=15552000;", x-content-type-options: "nosniff", x-powered-by: "Express", x-ratelimit-remaining2: "9892", …}
request: XMLHttpRequest {readyState: 4, timeout: 0, withCredentials: false, upload: XMLHttpRequestUpload, onreadystatechange: ƒ, …}
status: 200
statusText: "OK"
```
### 优化请求函数
总体思路，创建一个MyAxios类，所有接口公用部分的入参、公用配置new这个类的时候传入，然后生成的对象返回值是请求函数,
修改axios/index.js中的代码。

```
//  axios.js
import axios from 'axios'
// const SUCCESS_CODE = 200
const NOT_LOGIN_CODE = 401
class MyAxios {
  constructor (aipConfig, alertServerErrMsg) {
    if (!aipConfig) {
      throw new Error('aipConfig is required')
    }
    this.aipConfig = aipConfig
    this.alertServerErrMsg = alertServerErrMsg
    return this.doAxios.bind(this)
  }
  doAxios (apiName, postdata) {
    if (!this.aipConfig[apiName]) {
      console.error('please set apiName')
      return
    }
    if (typeof this.aipConfig[apiName] === 'string') {
      this.aipConfig[apiName] = { url: this.aipConfig[apiName] }
    }
    let {
      url,
      method = 'post',
      responseType = 'json',
      notAlertError = false,
      notNeedVerify = false
    } = this.aipConfig[apiName]

    if (!url) {
      console.error(`url is required in apiName`)
      return
    }
    return new Promise((resolve, reject) => {
      axios({
        url,
        method,
        params: postdata,
        data: postdata,
        responseType,
      }).then(this.checkStatus)
      .then(response => {
        //  go verification process or not
        if (notNeedVerify) {
          resolve(response)
          return
        }
        // if (response.data.code === SUCCESS_CODE) {
          resolve(response.data)
        // } else {
        //   //  triggering prompts or not
        //   if (!notAlertError) {
        //     this.errorNotify(this.alertServerErrMsg, response.data.message)
        //   }
        //   //  triggering error response event
        //   this.errorHandler(response.data.code)
        //   reject(response.data.message)
        // }
      }).catch((error) => {
        //  triggering prompts or not
        if (!notAlertError) {
          this.errorNotify(this.alertServerErrMsg, response.data.message)
        }
        reject(error.message)
      })
    })
  }
  errorNotify (notifyFunc, message) {
    if (notifyFunc) {
      notifyFunc(message)
    } else {
      window.alert(message)
    }
  }
  errorHandler (code) {
    if (code === NOT_LOGIN_CODE) {
      //  to do clear token
    }
  }
  checkStatus (response) {
    if (response.data && response.status === 200) {
      return response
    } else {
      let error = new Error(response.statusText || response.status || '网络通讯失败')
      throw error
    }
  }
}
export default MyAxios
```
创建一个api目录统一维护接口，创建index.js文件
```
// api/index.js
const apiList = {
  testApiOne: '/api/v2/muzzy/columns/10018/items',
  testApiTwo: {
    url: '/api/v2/muzzy/columns/10018/items',
    method: 'get'
  }
}
export default apiList
```
然后在main.js中初始化配置
```
//main.js
import Vue from 'vue'
import App from './App.vue'
import MyAxios from '@/axios/index.js'
import apiList from '@/api'
Vue.config.productionTip = false
Vue.prototype.$axios = new MyAxios(apiList)
new Vue({
  render: h => h(App),
}).$mount('#app')
```
在App.vue中使用
```
this.$axios('testApiTwo').then(res => {
    console.log(res)
  }).catch(() => {
    console.log('错误了')
  })
```
函数里面具体怎么处理返回值是根据和后端协定的格式决定。

### 使用mockjs
创建一个mock目录,建立index.js添加mock, mock的文档地址[https://github.com/nuysoft/Mock/wiki](https://github.com/nuysoft/Mock/wiki)
```
// mock/index.js
const Mock = require('mockjs')
//  建议将接口分批定义对应语义化的.js文件进行管理，然后统一引入到mock/index.js文件下。mock的语法规范请看官网，本例子只写封装过程。
const mockList = {
  testApiTwo: {
    "code": 200,
    "data": {
        test: 'mock数据'
    }
    "message": "成功了"
  }
}
export default function useMock(apiName, url) {
  if (!mockList[apiName]) {
    throw `请自行添加${apiName}接口的mock数据`
  }
  Mock.mock(url, mockList[apiName])
}
```
然后在api/index.js配置中引入mock。代码如下。
```
import useMock from '@/mock'
const version = 'v1'
//  建议apiList分批定义对应语义化的.js文件进行管理，然后统一引入到mock/index.js文件下.
const apiList = {
  testApiOne: '',
  testApiTwo: {
    url: '/testApiTwo',
    mock: true
  }
}
addApiParams(apiList)
function  addApiParams (apiList) {
  Object.keys(apiList).forEach(key => {
    //  验证参数
    valiateParams(apiList[key])
    //  补充url
    if (typeof apiList[key] === 'string') {
      apiList[key] = '/' + version + apiList[key]
    } else {
      apiList[key].url = '/' + version + apiList[key].url
    }
    //  是否使用mock
    if (apiList[key].mock) {
      let url = apiList[key].url || apiList[key]
      useMock(key,url)
    }
  })
}
function valiateParams (api) {
  if (typeof api !== 'string' && typeof api !== 'object' || api === null) {
    throw 'api错误格式， 必须的是url，或者包含url属性的对象'
  }
  if (typeof api === 'object' && typeof api.url !== 'string') {
    throw 'api错误格式， 必须的是url，或者包含url属性的对象'
  }
}
export default apiList
```
以上就是我自己封装的请求函数， 谢谢浏览。
