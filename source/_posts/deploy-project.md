---
title: window系统自动发布项目到测试服务器脚本。
---
项目开发过程中想发布一个版本到测试服务器上，如果通过手动打包npm run build，然后手动连接服务器发布，必然会浪费很多时间。我们可以在项目中写一个自动发布的脚本，节省时间。第一步，使用公钥自动登录服务器。第二步，写好自动发布的命令。原理解释参照了该作者        [https://www.cnblogs.com/scofi/p/6617394.html](https://www.cnblogs.com/scofi/p/6617394.html)

## 一、SSH公钥登录

### 1、SSH公钥登录的原理


我们远程登录服务器要用到SSH协议, 如下， user填自己服务器用户名, host服务器地址.

``` bash
$ ssh user@host
```


一般登录有两种方式，通过密码口令登录，或者公钥登录。
密码口令登录，主要流程原理是：
            1、客户端连接上服务器之后，服务器把自己的公钥传给客户端
            2、客户端输入服务器密码通过公钥加密之后传给服务器
            3、服务器根据自己的私钥解密登陆密码，如果正确那么就让客户端登录

公钥登录，主要流程原理是：
            1、 客户端生成RSA公私对。
            2、 客户端将自己的公私钥存放在服务器。
            3、 客户端请求连接服务器，服务器将一个随机字符串发送给客户端。
            4、 客户端用自己的私钥加密这个随机字符串发给服务端。
            5、 服务端使用公钥解密这个随机字符串，如果解密成功，则允许客户端登录


### 2、聊一聊RSA

对称加密： 对称加密是最快速、最简单的一种加密方式，加密与解密用的是同样的密钥。对称加密的缺点是密钥的管理和分配，如何将密钥发送给需要解密你的消息的人是一个问题，在发送密钥的过程中，密钥有很大的风险会被黑客们拦截。现实中通常的做法是将对称加密的密钥进行非对称加密传送给需要它的人。
非对称加密：非对称加密数据为数据的加密与解密提供了一个非常安全的方法，它使用一对公私钥。私钥只能由一方安全保管，而公钥可以发给任何请求它的人。比如你可以向银行请求公钥， 银行把公钥发给你， 你用公钥对消息进行加密，那么只有私钥的持有人银行才能对你的消息解密。非对称加密很安全，但很慢。



公钥和私钥：
1、 一个公钥对应一个私钥
2、 公私钥对中，大家都知道的是公钥，只有自己知道的是私钥。
3、 如果用其中一个密钥加密数据， 则只有对应的那个密钥才可以解密。
4、 如果用其中一个密码可以进行解密数据，则该数据必然是对应的那个密码进行的加密

RSA算法的作用：
    1. 加密：公钥加密私钥解密
            主要用于将数据资料加密不被其他人非法获取，保证数据的安全性，使用公钥将数据进行加密，即使第三方获取到数据没有私钥也无法解密，从而保证了数据的安全性。
    2. 认证：私钥加密公钥解密
            主要用户身份验证，判断某个身份的真实性。使用私钥加密之后，用对应的公钥解密来验证身份。
SSH公钥登录则用的是第二种功能


### 2、SSH公钥登录的实例


1、在客户端终端运行命令生成公私钥对(window  可以使用git bash)，敲完命令后可以一直回车使用默认设置。


```bash
$ ssh-keygen -t rsa 
```


公私钥默认生成在~/.ssh目录下,也就是/c/Users/xxx/.ssh/目录下，xxx是你电脑的用户名。



2、将公钥上传到服务器


``` bash
$ ssh-copy-id -i ~/.ssh/id_rsa.pub user@server_ip
```


~/.ssh/id_rsa.pub为你本机公钥， user修改为你服务器用户名  server_ip修改为你服务器ip，回车后根据提示输入服务器密码，成功后会提示看到这些字符"ssh 'user@xxxxxx'"
xxxx为服务器ip，之后这样就可以连接服务器了


## 二、发布项目脚本


发布项目脚本，简单发一个脚本例子好了。建立一个deployProject.sh 文件，输入如下指令。


``` bash
$ mv dist deploytest
tar -czf deploytest.tar deploytest
rm -rf deploytest
scp deploytest.tar user@server_ip:/usr/share/nginx/html/deploytest.tar
rm -rf deploytest.tar
ssh user@server_ip 'cd /user/share/nginx/html && rm -rf deploytest 
&& tar -xzf deploytest.tar && rm -rf deploytest.tar'
echo 'done. 完成发布'
```


然后在添加运行该脚本的指令，在package.json里面添加


``` bash
"script": {
    "deploy": "npm run build && sh deployProject.sh"
}
```


使用 npm run deploy  即可发布项目  如出现不存在该指令等提示，需要配置好系统环境变量， 本人使用git bash执行以上全部操作的  都是OK的。

