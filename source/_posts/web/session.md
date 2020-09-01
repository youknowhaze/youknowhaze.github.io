---
title: session（会话）
date: 2020-07-02
categories:
 - Web
tags:
 - session
---


## session
   参考自：https://blog.csdn.net/J080624/article/details/78562787
   代码将在Advanced-Study 项目进行练习
### 1.什么是session
Cookie是存储在客户端方，Session是存储在服务端方，客户端只存储SessionId。

Session是另一种记录客户状态的机制，不同的是Cookie保存在客户端浏览器中，而Session保存在服务器上(内存或硬盘)。一般Session存储在服务器的内存中，tomcat的StandardManager类将session存储在内存中，也可以持久化到file，数据库，memcache，redis等。
客户端只保存sessionid到cookie中，而不会保存session，session销毁只能通过invalidate或超时(默认30分钟)，关掉浏览器并不会关闭session。

客户端浏览器访问服务器的时候，服务器把客户端信息以类似于散列表的形式记录在服务器上，这就是Session。客户端浏览器再次访问时只需要从该Session中查找该客户的状态就可以了。

服务端创建一个session前，会先判断客户端发起的请求中是否包含session标识（sessionId），若不包含，则为此用户创建一个session并生成一个与次session相关的sessionId返回给客户端。

### 2.session的使用及读取
   Session对应的类为javax.servlet.http.HttpSession类。

   每个来访者对应一个Session对象，所有该客户的状态信息都保存在这个Session对象里。

   Session对象是在客户端第一次请求服务器的时候创建的。

   Session也是一种key-value的属性对，通过getAttribute(Stringkey)和setAttribute(String key，Objectvalue)方法读写客户状态信息。

   Servlet里通过request.getSession()方法获取该客户的Session。
````
// 创建session
HttpSession session = request.getSession();
//设置属性
session.setAttribute("name","tom");
//读取属性
session.getAttribute("name");
//移除属性
session.removeAttribute("name");
//设置有效期，单位为秒，-1代表永不过期
session.setMaxInactiveInterval(1000);
//使其失效
session.invalidate();

````

   request还可以使用getSession(boolean create)来获取Session。

````
getSession(true)//先创建再返回
getSession(false)//返回null
getSession()//先创建再返回
````

   **这里写两个接口来测试session**
````
@RequestMapping("/testSession")
public String sessionRequest(HttpSession session){
    session.setAttribute("testSession","this is a session");
    return "session";
}

@RequestMapping("/getSession")
public String getSession(HttpSession session){
    Object sessionValue = session.getAttribute("testSession");
    return String.valueOf(sessionValue);
}
````
#### (1)、第一次请求，不携带session信息，服务端生成session，会在返回中带有sessionId。
![第一次请求](/session/session01.png)

#### (2)、第二次请求，携带sessionId，返回中不再带有session相关信息
![第二次请求](/session/session02.png)

#### (2)、第三次请求，带有sessionId，调用接口查询session中的信息
![第三次请求](/session/session03.png)

### 3.session的生命周期

**Session在用户第一次访问服务器的时候自动创建**。

   需要注意只有访问JSP、Servlet等程序时才会创建Session，只访问HTML、IMAGE等静态资源并不会创建Session。如果尚未生成Session，也可以使用request.getSession(true)强制生成Session。

**做如下解释说明**

1. session是服务器建立的，但服务器不会主动建立。当一个请求过来时，服务器并不会建立一个session,而是要这样：request.getSession();当我们用程序通知服务器时它才会建立一个会话。

2. 在jsp文件中有一个默认的属性 <%@ page session=“true” %>,翻译成java后就是： session = pageContext.getSession();

3. 所以当我们访问一个jsp文件的时候，如果没有去更改这个默认值，那也会建立一个会话，这也是大多数人会认为自动创建session的原因了，其实还是相当于我们主动通知服务器去创建。

4. 综上所述：Tomcat这类服务器不会自动的创建session,只有当我们主动通知它时，才会创建会话


   Session生成后，只要用户继续访问，服务器就会更新Session的最后访问时间，并维护该Session。用户每访问服务器一次，无论是否读写Session，服务器都认为该用户的Session“活跃（active）”了一次。

   由于会有越来越多的用户访问服务器，因此Session也会越来越多。为防止内存溢出，服务器会把长时间内没有活跃的Session从内存删除。这个时间就是Session的超时时间。如果超过了超时时间没访问过服务器，Session就自动失效了。

   Session的超时时间为maxInactiveInterval属性
   默认有效期为30分钟，30分钟内没有"活跃"则失效。如果"活跃"则重新计算生命周期。
````
//默认 time 1800
int time = session.getMaxInactiveInterval();
session.setMaxInactiveInterval(Integer.MAX_VALUE);
````
   Session的超时时间也可以在web.xml中修改。另外，通过调用Session的invalidate()方法可以使Session失效。

invalidate()是指清空session对象里的东西，并不指清除这个session对象本身。所以，要判断一个session里面是否存在自己想要的东西（这个session是否有效），是不能用isNew()的，应该用如下类似代码：

````
UserInfo userInfo=(UserInfo)session.getAttribute(”USERINFO”);
if (userInfo!=null){
	//...
}
````

Session需要使用Cookie作为识别标志。HTTP协议是无状态的，Session不能依据HTTP连接来判断是否为同一客户，因此服务器向客户端浏览器发送一个名为JSESSIONID的Cookie，它的值为该Session的id（也就是HttpSession.getId()的返回值）。Session依据该Cookie来识别是否为同一用户。

该Cookie为服务器自动生成的，它的maxAge属性一般为–1，表示仅当前浏览器内有效，并且不同的浏览器的窗口间不共享(同一浏览器共享)，关闭浏览器就会失效。

同一机器的两个不同类型的浏览器的窗口访问服务器时，会生成两个不同的Session。但是由浏览器窗口内的链接、脚本等打开的新窗口除外。这类子窗口会共享父窗口的Cookie，因此会共享一个Session。

### 4.URL重写

如果客户端浏览器将Cookie功能禁用，或者不支持Cookie怎么办？例如，绝大多数的手机浏览器都不支持Cookie。

Java Web提供了另一种解决方案：URL地址重写。
````
<a href="<%=response.encodeUrl(url)%>"></a>
````

URL地址重写的原理是将该用户Session的id信息重写到URL地址中。服务器能够解析重写后的URL获取Session的id。

这样即使客户端不支持Cookie，也可以使用Session来记录用户状态。

encodeurl()方法在使用时，会首先判断Session是否启用，如果未启用，直接返回url。 然后判断客户端是否启用Cookie，如果未启用，则将参数url中加入SessionID信息，然后返回修改的URL；如果启用，直接返回参数url。

**注意：Tomcat判断客户端浏览器是否支持Cookie的依据是请求中是否含有Cookie。**

尽管客户端可能会支持Cookie，但是由于第一次请求时不会携带任何Cookie（因为并无任何Cookie可以携带），URL地址重写后的地址中仍然会带有jsessionid。

当第二次访问时服务器已经在浏览器中写入Cookie了，因此URL地址重写后的地址中就不会带有jsessionid了。

### 5.session中禁止使用cookie

既然WAP上大部分的客户浏览器都不支持Cookie，索性禁止Session使用Cookie，统一使用URL地址重写会更好一些。Java Web规范支持通过配置的方式禁用Cookie。

打开项目sessionWeb的WebRoot目录下的META-INF文件夹（跟WEB-INF文件夹同级，如果没有则创建），打开context.xml（如果没有则创建），编辑内容如下：
````
/META-INF/context.xml
<?xml version='1.0' encoding='UTF-8'?>
<Context path="/sessionWeb"cookies="false">
</Context>
````
或者修改Tomcat全局的conf/context.xml，修改内容如下：
````
<!-- The contents of this file will be loaded for eachweb application -->
<Context cookies="false">
    <!-- ... 中间代码略 -->
</Context>
````
部署后TOMCAT便不会自动生成名JSESSIONID的Cookie，Session也不会以Cookie为识别标志，而仅仅以重写后的URL地址为识别标志了。

**注意：该配置只是禁止Session使用Cookie作为识别标志，并不能阻止其他的Cookie读写。也就是说服务器不会自动维护名为JSESSIONID的Cookie了，但是程序中仍然可以读写其他的Cookie。**


### 6.sessionId
sessionid是一个会话的key，浏览器第一次访问服务器会在服务器端生成一个session，有一个sessionid和它对应。tomcat生成的sessionid叫做jsessionid。

session在访问tomcat服务器HttpServletRequest的getSession(true)的时候创建，tomcat的ManagerBase类提供创建sessionid的方法：随机数+时间+jvmid。

其存储在服务器的内存中，tomcat的StandardManager类将session存储在内存中，也可以持久化到file，数据库，memcache，redis等。客户端只保存sessionid到cookie中，而不会保存session，session销毁只能通过invalidate或超时，关掉浏览器并不会关闭session。

session不会因为浏览器的关闭而删除。但是存有session ID的cookie的默认过期时间是会话级别。也就是用户关闭了浏览器，那么存储在客户端的session ID便会丢失，但是存储在服务器端的session数据并不会被立即删除。从客户端即浏览器看来，好像session被删除了一样（因为我们丢失了session ID，找不到原来的session数据了）。


### 7.Session Cookie

session通过SessionId来区分不同的客户，session是以cookie或url重写为基础的。默认使用cookie来实现(如果浏览器支持的话)，系统会创造一个名为JESSIONID的输出cookie，这称之为session cookie。以区别persistent cookie(也就是我们通常所说的cookie)。

session cookie是储存在浏览器内存的，并非写到硬盘上，通常看不到JESSIONID。但是当把浏览器的cookie禁用后，web服务器会采用URL重写的方式传递session id。这是地址栏将会看到。

session cookie只针对某一次会话而言，会话结束，session cookie也就跟着消失了。关闭浏览器只会使浏览器内存里的session cookie消失，但不会时服务器端的session对象消失。同样也不会使已经保存到硬盘的持久化cookie消失。

但是一旦session cookie消失，没有了session id，服务器端的原先对应的session对象也就无从分辨，相当于"失效"。
