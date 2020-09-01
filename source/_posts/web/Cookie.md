---
title: Cookie（缓存）
date: 2020-07-01
categories:
 - Web
tags:
 - Cookie
---


## Cookie（缓存）

代码将在Advanced-Study 项目进行练习

### 1.Cookie介绍

在了解这三个概念之前我们先要了解HTTP是无状态的Web服务器，什么是无状态呢？一次请求完成后，下一次请求完全不知道上一次请求发生了什么。如果在Web服务器中只是用来管理静态文件还好说，对方是谁并不重要，把文件从磁盘中读取出来发出去即可。但是随着网络的不断发展，比如电商中的购物车只有记住了用户的身份才能够执行接下来的一系列动作。所以此时就需要我们无状态的服务器记住一些事情。

那么Web服务器是如何记住一些事情呢？既然Web服务器记不住东西，那么我们就在外部想办法记住，相当于服务器给每个客户端都贴上了一个小纸条。上面记录了服务器给我们返回的一些信息。然后服务器看到这张小纸条就知道我们是谁了。那么Cookie是谁产生的呢？Cookies是由服务器产生的。接下来我们描述一下Cookie产生的过程。浏览器第一次访问服务端时，服务器此时肯定不知道他的身份，所以创建一个独特的身份标识数据，格式为key=value，放入到Set-Cookie字段里，随着响应报文发给浏览器。

- 浏览器看到有Set-Cookie字段以后就知道这是服务器给的身份标识，于是就保存起来，下次请求时会自动将此key=value值放入到Cookie字段中发给服务端。

- 服务端收到请求报文后，发现Cookie字段中有值，就能根据此值识别用户的身份然后提供个性化的服务。

### 2.Cookie的使用
   接下来用代码演示一下，接口代码如下，cookie的key值设置为testUser
````
@RequestMapping("/testCookies")
    public String cookiesRequest(HttpServletRequest request, HttpServletResponse response){

        Cookie cookie = new Cookie("testUser","cookiesUser");
        response.addCookie(cookie);

        return "cookie";
    }
````
   访问http://localhost:8082/testCookies 接口，来看一下接口请求和返回，可以看到请求中未带有我们需要的缓存中，返回中带有设定好的缓存key-value。

![请求不带缓存](/Cookie/cookie01.png)

   同样的请求地址和浏览器，当我们第二次发起接口调用请求时，会发现请求中带有上一次返回的cookie值

![请求带有缓存](/Cookie/cookie02.png)

   接下来我们换一个请求呢？是不是Cookie也会带过去呢？接下来我们输入路径http://localhost:8082 请求。我们可以看到Cookie字段还是被带过去了。

![请求带有缓存](/Cookie/cookie03.png)

   那么浏览器的Cookie是存放在哪呢？如果是使用的是Chrome浏览器的话，那么可以按照下面步骤。

- 在计算机打开Chrome

- 在右上角，一次点击更多图标->设置

- 在底部，点击高级

- 在隐私设置和安全性下方，点击网站设置

- 依次点击Cookie->查看所有Cookie和网站数据



  | 属性名 | 描述 |
  | :---: | :---- |
  | name | Cookie的名称，Cookie一旦创建，名称便不可更改 |
  | value | Cookie的值，如果值为Unicode字符，需要为字符编码。如果为二进制数据，则需要使用BASE64编码 |
  | maxAge | Cookie失效的时间，单位秒。如果为整数，则该Cookie在maxAge秒后失效。如果为负数，该Cookie为临时Cookie，关闭浏览器即失效，浏览器也不会以任何形式保存该Cookie。如果为0，表示删除该Cookie。默认为-1。 |
  | secure | 该Cookie是否仅被使用安全协议传输。安全协议。安全协议有HTTPS，SSL等，在网络上传输数据之前先将数据加密。默认为false。 |
  | path | Cookie的使用路径。如果设置为“/sessionWeb/”，则只有contextPath为“/sessionWeb”的程序可以访问该Cookie。如果设置为“/”，则本域名下contextPath都可以访问该Cookie。注意最后一个字符必须为“/”。 |
  | domain | 可以访问该Cookie的域名。如果设置为“.google.com”，则所有以“google.com”结尾的域名都可以访问该Cookie。注意第一个字符必须为“.”。 |
  | comment | 该Cookie的用处说明，浏览器显示Cookie信息的时候显示该说明。 |
  | version | Cookie使用的版本号。0表示遵循Netscape的Cookie规范，1表示遵循W3C的RFC 2109规范 |

**实例路径和地址限制**

设置为cookie.setPath("/testCookies")，接下来我们访问http://localhost:8082/testCookies,
我们可以看到请求中存在cookie，而访问非/testCookies地址，则没带有cookie。这就是Path控制的路径。

**Domain**

设置为cookie.setDomain("localhost")，接下来我们访问http://localhost:8082/testCookies
我们发现是有Cookie的字段的，但是我们访问http://172.16.42.81:8082/testCookies，
可以看到没有Cookie的字段了。这就是Domain控制的域名发送Cookie。

由于验证方式与前面一致，所以这里就不再贴图。
