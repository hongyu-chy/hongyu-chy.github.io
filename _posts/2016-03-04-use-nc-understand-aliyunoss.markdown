---
layout: post
title:  "阿里云OSS 图片处理如何借助CDN将多个域名绑定到一个bucket（channel）上"
date:   2016-01-18 22:53:44
author: sunrainchy
categories: ShellCode
image: http://site-img-data.img-cn-shanghai.aliyuncs.com/blog-data/2016.01.18/aliyun.jpg@600w_600h
---


##1、本文要说什么
利用nc 直接与阿里云OSS服务器建立TCP 连接，通过输入HTTP 请求头部及数据与OSS进行交互，以此理解阿里云OSS服务的本质及使用阿里云OSS过程中的一些trouble shooting。
##2、相关准备工作
 一台连上互联网的Linux主机（如果没有可以购买一台阿里云ECS），注册并开通阿里云OSS服务（oss.aliyun.com)并创建一个bucket，本文以青岛的bucket：bucket-example（bucket-example.oss-cn-qingdao.aliyuncs.com) 为例进行一系列操作，为了简化演示步骤将bucket设置为public-read-write 公开读写，这样可以忽略签名。OSS 相关文档见oss.aliyun.com。

##3、利用nc 从HTTP连接角度体验阿里云OSS
###3.1、在OSS上创建一个新文件
 用HTTP PUT 方法在OSS上创建一个新文件，先创建一个空文件吧
 先根据OSS Restful API 把HTTP 请求构造出来，注意下面content-length 设置为0，这样就可以创建一个空文件
 ```
 PUT /empty_file HTTP/1.1
 Content-Length: 0
 Host: bucket-example.oss-cn-qingdao.aliyuncs.com
 ```
 利用nc 与oss 服务器80端口建立一条tcp链接

 ```
 nc bucket-example.oss-cn-qingdao.aliyuncs.com 80
 ```
 看结果，OSS 返回200 Ok，请求成功，注意下面的Etag，正好是一个空文件的md5，这样，一个空文件就创建成功了，可以去控制台上看看是否真的有。
 ![这里写图片描述](http://img.blog.csdn.net/20160304175151138)

 接下来创建一个有内容的文件吧，把上面PUT 请求HTTP 头部的content-length 改成10试试。
 ![这里写图片描述](http://img.blog.csdn.net/20160304175950975)
 按理现在我们在OSS上已经创建一个内容为”Hello， World.“， 是否创建成功并且内容真的是”Hello， World. “接下来再利利用nc 体验下GET 请求吧。
###3.2、获取 OSS 上的文件
 首先还是构造GET 请求的HTTP 头部

 ```
 GET /empty_file HTTP/1.1
 Host: bucket-example.oss-cn-qingdao.aliyuncs.com
 ```
 我们先获取3.1 中创建的空文件，200 Ok，Content-Length为0， 说明文件存在而且内容确实为空
 ![这里写图片描述](http://img.blog.csdn.net/20160304180927877)

 再获取下刚刚的创建的内容为"Hello， World." 的文件（见下图），但是为什么GET 下来内容是"Hello，Wor"？
 再看下3.1 节PUT 时候的Content-Length长度为10， 而"Hello， World."长度超过10，对于多出来的数据OSS会做截断处理，如果是Keeplive连接超出Content-Length的数据会理解成下一个HTTP请求数据。
 ![这里写图片描述](http://img.blog.csdn.net/20160304181228393)

###3.3、复制OSS上的文件
 根据OSS Restful API 构造Copy 请求，其实就是PUT 请求加上x-oss-copy-source 头部指定Copy 源文件位置

 ```
 PUT /oss_file_copy HTTP/1.1
 Host: bucket-example.oss-cn-qingdao.aliyuncs.com
 x-oss-copy-source: /bucket-example/oss_file
 ```
 执行过程及结果结果如下图，如果源和目的为同一个文件可通过更改头部更改ObjectMeta，可以尝试用一些错误的头部测试OSS的行为。现在你的bucket 下通过copy 多出来一个oss_file_copy 文件。
 ![这里写图片描述](http://img.blog.csdn.net/20160304194023185)

###3.4、删除OSS上的文件
 Delete 请求HTTP 头部

 ```
 DELETE /oss_file HTTP/1.1
 Host: bucket-example.oss-cn-qingdao.aliyuncs.com
 ```
 执行过程及结果如下图，现在OSS 上bucket-example 下的oss_file文件已经被删除了
 ![这里写图片描述](http://img.blog.csdn.net/20160304195031649)


##4、利用nc 对进行OSS使用过程中相关trouble shooting
###4.1、调试cors 请求
 跨域这个东西做前端的同学遇到的应该比较多，使用OSS过程中出现浏览器向OSS发送跨域请求出问题时第一反应是”怪罪“OSS，其实不一定是OSS的问题，有可能是浏览器根本就没把请求发出去等等原因.......
 下面我们就来利用nc测试下当跨域请求出问题的时候是否是OSS问题。
####4.1.1、构造跨域请求（OPTIONS）

 ```
 OPTIONS /fs/static/axe/resaxe/css/loading.css HTTP/1.1
 Host:dbk-file-1.oss-cn-beijing.aliyuncs.com
 Access-Control-Request-Method: POST
 Access-Control-Request-Headers: accept, content-type
 Origin: http://fs.dongboke.com
 ```
 将此请求发送给OSS，OSS返回结果是403 Forbidden，如下图
 看到错误提示”CORSResponse: CORS is not enabled for this bucket.“
 ![这里写图片描述](http://img.blog.csdn.net/20160304200921550)
####4.1.2、设置OSS Bucket Cors
 去控制台上对bucket-example 设置cors
 登陆OSS管理控制台，点进bucket-example->bucket 属性->Cors设置->添加规则
 ![这里写图片描述](http://img.blog.csdn.net/20160304201653622)

####4.1.3、验证结果
 再将此OPTIONS 请求发送给OSS，请求成功
 ![这里写图片描述](http://img.blog.csdn.net/20160304201945522)

###4.2、利用nc replay OSS 请求
####4.1、背景
 在学习及使用OSS的过程中肯定会遇到很多问题，而这些问题必须通过抓包来查出问题，看看HTTP 请求到底把什么数据发出去了又收到了什么样的数据，而很多在线上使用的bucket权限是不能够设置为public-read-write 的，因此在构造请求头部的时候需要签名，这就大大加大了OSS HTTP请求头部构造难度，因此我们可以利用nc 及 wireshark 来对HTTP请求进行重放，OSS 签名在15分钟内有效，因此在15分钟内都可以对请求进行重放，甚至可以利用此点清空object内数据。
####4.2、wireshark 抓包
 为了便于演示这里用osscmd（可取官网下载）发送带OSS签名的请求，同时启动wireshark抓包

 ```
 osscmd put file oss://my-data-store
 ```
 过滤HTTP 请求包，也就是dst port 80 的包，点击并单击中右键->Follow TCP Stream
 ![这里写图片描述](http://img.blog.csdn.net/20160304203428473)
 可以清晰看到请求头部及数据
 ![这里写图片描述](http://img.blog.csdn.net/20160304203703435)
####4.3、将头部数据复制出来利用nc进行重放
 将头部复制出来

 ```
 PUT /file HTTP/1.1
 Host: my-data-store.oss.aliyuncs.com
 Accept-Encoding: identity
 Content-Length: 25
 User-Agent: aliyun-sdk-python/0.4.0 (Linux/3.2.0-87-generic-pae/i686[.7.3)
    Host: my-data-store.oss.aliyuncs.com
    Expect: 100-Continue
    Date: Fri, 04 Mar 2016 12:33:28 GMT
    Content-Type: application/octet-stream
    Authorization: OSS V7lMNqSkExeEsh0Z:QsfZOMIPjy2+eqmiPVtYsnK3JrU=
    ```
    进行重放，成功
    
    ![这里写图片描述](http://img.blog.csdn.net/20160304204442282)
####4.4、试试改改Content-Length
    content-length 可以改成任意合法值，我们就可以更改object内数据了
    我们来试试将content-length 改成0，清空object内数据试试
    
    注意content-length，和Etag，content-length我们改成0， OSS返回的Etag 是空文件的md5
    清空object内数据成功。
    ![这里写图片描述](http://img.blog.csdn.net/20160304204844380)
    
##5、总结
    以上几个例子通过nc 与OSS 服务器建立连接后输入HTTP请求数据直观感受OSS对外提供的功能，当然OSS提供的功能还远远不止这么多，还有其他很多很赞的功能都是以HTTP 形式对外提供，可以根据官方提供的API文档构造出HTTP请求头部用上述同样方法与OSS交互，nc 会打印出TCP连接上所有HTTP数据，直观理解OSS。通过nc 与 wireshark配合重放OSS 请求去重现问题并进行相关trouble shooting。
    
    
