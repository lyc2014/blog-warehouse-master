---
title: 了解arraybuffer和blob区别，并实现图片加密
date: 2019-11-21 14:24:51
tags:
---
### ArrayBuffer
ArrayBuffer对象用于表示通用的、固定长度的原始二进制数据缓冲区（即代表着储存二进制数据的一段内存）。它是不能直接读写的，只能通过视图（TypedArray视图和DataView视图）来读写，视图的作用是以指定格式解读二进制数据。

ArrayBuffer也是一个构造函数，可以分配一段可以存放数据的连续内存区域

```
const buf = new ArrayBuffer(8)
```
{% asset_img photo1.jpeg This is an image %}
然后可以通过视图修改这段内存
```
const x1 = new Uint8Array(buf)
x1[0] = 7
```
{% asset_img photo2.jpeg This is an image %}
可以看出二进制值已经被修改了。

### Blob
Blob对象表示一个不可变、原始数据的类文件对象。Blob表示的不一定是JavaScript的原生格式的数据。

如果要从其他非blob对象和数据构造一个Blob, 请使用Blob()构造函数, Blob()构造函数返回一个新的Blob对象，blob类型只有slice()方法。

```
var aBlob = new Blob(array, options)
```

对比发现，blob和arraybuffer都是二进制的容器，但是ArrayBuffer的数据，是可以按照字节去操作的，而Blob的只能作为一个整的对象去处理， 所以说，Arraybuffer相比Blob更接近真实的二进制，更底层。

### ArrayBuffer与Blob相互转换

#### ArrayBuffer转Blob

arraybuffer转blob很方便，直接作为参数传入即可
```
var buffer = new ArrayBuffer(16)
var blob = new Blob([buffer])
```
#### Blob转ArrayBuffer

Blob转ArrayBuffer需要借助Web的API： FileReader。 FileReader对象允许Web应用程序异步读取存储在用户计算机上的文件（或原始数据缓冲区）的内容，使用File或Blob对象指定要读取的文件或数据。

FileReader.readAsArrayBuffer() --- result为被读取文件的 ArrayBuffer数据对象
FileReader.readAsBinaryString() --- result为所读取文件的原始二进制数据  （非标准，尽量不要使用）
FileReader.readAsArrayBuffer() --- result属性中将包含一个data: URL格式的Base64字符串以表示所读取文件的内容
FileReader.readAsText()  --- result属性中将包含一个字符串以表示所读取的文件内容

```
  var blob = new Blob([1,2,3,4,5]);
  var reader = new FileReader();

  reader.onload = function() {
      console.log(this.result);
  }
  reader.readAsArrayBuffer(blob);
```
### 应用
思路：每种格式的图片的头一个二进制字节都是一样的，可以将这个字节-1打乱使得图片无法打开。然后获取图片的时候再-1解密，并使用window.URL.createObjectURL创建一个URL对象，图片渲染完后销毁。
#### 加密图片
input上传图片后，对图片进行加密处理, 并下载下来。(仅展示关键代码部分， 详情可点击文章尾部git仓库查看demo)
```
var img = document.getElementById('img')
var blob = Files[0]
var name = blob.name
var reader = new FileReader()
reader.onload = function () {
  var arrayBuff = this.result
  var v1 = new Uint8Array(arrayBuff)
  v1[0]--
  var blobFile = new Blob([arrayBuff])
  var link = document.createElement('a')
  link.href = window.URL.createObjectURL(blobFile)
  link.download = name
  link.click()
}
reader.readAsArrayBuffer(blob);
```

#### 解密图片
使用 ajax 获取图片, 对头一个字节+1解密。然后渲染图片。(仅展示关键代码部分， 详情可点击文章尾部git仓库查看demo)
```
xhr.responseType = 'arraybuffer'
xhr.onload = function () {
  if (this.status == 200) {
    var arrayBuf = this.response
    var v1 = new Uint8Array(arrayBuf)
    v1[0]++
    var blobFile = new Blob([arrayBuf])

    img.onload = function(e) {
      window.URL.revokeObjectURL(img.src); 
    };
    img.isParse = true
    img.src = window.URL.createObjectURL(blobFile);
  }
}
```
这样图片即可展示出来，这样以来无论是img的src上的链接还是sources下载下来的图片资源，都是无效图片。实现了简单加密的功能。
