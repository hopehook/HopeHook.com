---
date: 2016-09-14
layout: post
title: 简单的单点登陆实现方案
thread: 2016-09-14-single_sign_on.md
categories: 经验
tags: github
---


#### 针对场景

* 各个web系统一级域名相同,如域名都是 *.example.com
* 只考虑支持CORS的浏览器

</br>
#### AUTH工程是什么

* 是各个web系统的集中式登录验证后台
* 是各个web系统目前单点登录的解决方案主体

</br>
#### AUTH上线后的变化

* 各个web系统的登录和退出调用全部交由AUTH处理
* 各个web系统打破了边界限制，可以像一个web系统一样运作和跳转

</br>
#### AUTH的登录和退出流程
<ul>
  <li>
  各个web系统向AUTH发出跨域ajax调用请求登录
  </li>
  <li>
  AUTH登录验证成功
    <ul>
      <li>将session存储到session/redis/mysql中</li>
      <li>将cookie跨二级域写到ajax请求方的浏览器中</li>
    </ul>
  </li>
  <li>
  各个web系统向AUTH发出跨域ajax调用请求退出
  </li>
  <li>AUTH退出操作
    <ul>
      <li>判断cookie是否存在</li>
      <li>如果存在cookie清理对应的session和cookie</li>
      <li>如果不存在，无需清理session</li>
    </ul>
  </li>
</ul>

</br>
#### AUTH的技术要点

* 跨域请求
* 跨二级域写cookie

</br>
#### 如何实现跨域请求

采用了CORS方案:
</br>
当你使用XMLHttpRequest发送请求时，浏览器发现该请求不符合同源策略，会给该请求加一个请求头：Origin，
后台进行一系列处理，如果确定接受请求则在返回结果中加入一个响应头：Access-Control-Allow-Origin;
浏览器判断该相应头中是否包含Origin的值，如果有则浏览器会处理响应，我们就可以拿到响应数据，
如果不包含浏览器直接驳回，这时我们无法拿到响应数据。

* 前端不用调整
* 后端给返回http报文添加Access-Control-Allow-Origin头部和请求的Origin头部保持一致即可

  `self.set_header('Access-Control-Allow-Origin', self.request.headers['Origin'])`

</br>
#### 如何实现跨二级域写cookie

* 前端ajax请求
添加 xhrFields:{
        withCredentials:true
    },
<pre>
$.ajax({
    url: url ,
    type: 'POST',
    dataType: 'json',
    data: rqdata,
    success: function (returndata) {
        window.location.href = "/admin";
    },
    error: function (returndata) {
        console.log(returndata)
    },
    xhrFields:{
        withCredentials:true
    },
    crossDomain: true
 });
</pre>
* 后端设置的头部
</br>
   `self.set_header('Access-Control-Allow-Credentials', 'true')`




