---
layout: post
title:  "阿里云OSS 图片处理如何借助CDN将多个域名绑定到一个bucket（channel）上"
date:   2016-01-18 22:53:44
author: sunrainchy
categories: ShellCode
image: http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2016.01.18/aliyun.jpg@600w_600h
---
###阿里云OSS 图片处理如何借助CDN将多个域名绑定一个bucket（channel）上
**无论是从优化浏览器行为上还是处于其他原因（比如说oss对外限制bucket个数为10个），现在有很多用户想在一个bucket上绑定多个域名，目前OSS已经对此做了支持，但是阿里云图片处理控制台上只允许一个bucket（channel）绑定一个域名，还不支持将多个域名绑定到同一个bucket（channel）上，由于图片处理服务大多情况下是配合CDN一起使用的，当然不用CDN直接使用阿里云OSS 图片处理服务提供的三级域名也可以体验阿里云OSS 图片处理服务。**
**现在就介绍下如何借助阿里云CDN做到多个域名绑定到一个bucket上：**
**现在有一个bucket，名为bucket-example**
**这个bucket 的对应的oss endpoint 为oss-cn-qingdao.aliyuncs.com**     
**图片处理 img endpoint 为img-cn-qingdao.aliyuncs.com**
**登陆到阿里云CDN控制台：https://cdn.console.aliyun.com/console/index#/**
<br>
**点击左上角的CDN域名列表**
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118205117218)
<br>
**点击右上角添加新域名**
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118205310249)
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118205725951)
<br>
**填写相关信息后点击下一步， 这里要注意域名一定是要在阿里云备案的，没有备案的**
**备案一下，阿里云备案速度还是很快的，从提交到最终备案完成我只花了两周时间。**
<br>
**这步骤完成之后还要进行CNAME操作，将我们的CDN加速域名cdn-test.chenhongyu.cn**
<br>
**CNAME到刚刚申请好的cdn加速域名上，这个如何绑定？**
**简单截个图:**
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118210508619)
<br>
**完成之后可以dig下域名**
<br>
```
dig cdn-test.chenhongyu.cn
```
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118210830519)
<br>
cdn-test.chenhongyu.cn.   600   IN   CNAME   cdn-test.chenhongyu.cn.w.kunlunar.com.
**在浏览器上访问这个域名，出现如下结果，绑定成功**
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118211022334)
<br>
**还有最后一步，由于我们直接访问的是域名，cdn拿到请求后转发给oss，host头部为**
**cdn-test.chenhongyu.cn, oss 不知道这个host对应的bucket是什么，因此要将cdn**
**回源host头部改掉，点击回源Host，选择源站域名。**
<br>
![这里写图片描述](http://img.blog.csdn.net/20160118210227987)
![这里写图片描述](http://img.blog.csdn.net/20160118211329226)
<br>
**至此一个域名绑定到bucket（channel）成功，再加一个域名同样的步骤重复一次即可。**

**一句话总结以上步骤：申请多个CDN回源域名，将回源地址填写为同意个bucket对应的三级域名，
并更改回源host 头部为源头站域名。**
