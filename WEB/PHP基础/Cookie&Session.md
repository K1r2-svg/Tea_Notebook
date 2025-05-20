

# Cookie & Session

## Cookie

### Cookie的定义

* 什么是cookie?

```
指某些网站为了辨别用户身份、进行session跟踪而存储在用户本地终端上的数据（通常经过加密）。也就是说如果知道一个用户的Cookie，并且在Cookie有效的时间内，就可以利用Cookie以这个用户的身份登录这个网站。
```

* 会话cookie和持久cookie的区别？

       如果不设置过期时间，则表示这个cookie生命周期为浏览器会话期间，只要关闭浏览器窗口，cookie就消失了。这种生命期为浏览会话期的cookie被称为会话cookie。会话cookie一般不保存在硬盘上而是保存在内存里。
       如果设置了过期时间，浏览器就会把cookie保存到硬盘上，关闭后再次打开浏览器，这些cookie依然有效直到超过设定的过期时间。
    　　存储在硬盘上的cookie可以在不同的浏览器进程间共享，比如两个IE窗口。而对于保存在内存的cookie，不同的浏览器有不同的处理方式。
## Session

### **session在含义**

```
session它的含义是指一类用来在客户端与服务器端之间保持状态的解决方案。有时候Session也用来指这种解决方案的存储结构。
```

### **session的机制**

```
	session机制是一种服务器端的机制，服务器使用一种类似于散列表的结构(也可能就是使用散列表)来保存信息。但程序需要为某个客户端的请求创建一个session的时候，服务器首先检查这个客户端的请求里是否包含了一个session标识 —— 称为session id。
	如果已经包含一个session id则说明以前已经为此客户创建过session，服务器就按照session id把这个session检索出来使用(如果检索不到，可能会新建一个，这种情况可能出现在服务端已经删除了该用户对应的session对象，但用户人为地在请求的URL后面附加上一个JSESSION的参数)。
	如果客户请求不包含session id，则为此客户创建一个session并且生成一个与此session相关联的session id，这个session id将在本次响应中返回给客户端保存。
```

### cookie机制和session机制的区别

```
cookie机制采用的是在客户端保持状态的方案，而session机制采用的是在服务器端保持状态的方案。 
```

### Cookie和Session、SessionID的关系

```
	sessionid 是一个会话的 key，浏览器第一次访问服务器会在服务器端生成一个 session，有一个 sessionID 和它对应，并返回给浏览器，这个 sessionID 会被保存在浏览器的会话 cookie 中。tomcat 生成的 sessionID 叫做 jsessionID。
```

```
   sessionID 在访问 tomcat 服务器 HttpServletRequest 的 getSession(true) 的时候创建，tomcat 的 ManagerBase 类提供创建sessionID 的方法：随机数+时间+jvmid。Tomcat 的 StandardManager 类将 session 存储在内存中，也可以持久化到 file，数据库，memcache，redis等。
```

```
 	客户端只保存 sessionID 到 cookie 中，而不会保存 session。session 不会因为浏览器的关闭而删除，只能通过程序调用 HttpSession.invalidate() 或超时才能销毁。
```
session 的 id 是从哪里来的，sessionID 是如何使用的？
```
	当客户端第一次请求 session 对象时候，服务器会为客户端创建一个 session，并将通过特殊算法算出一个 session 的 ID，用来标识该 session 对象。
```

session 存放在哪里？

       服务器端的内存中。不过 session 可以通过特殊的方式做持久化管理（memcache，redis）。

session在下列情况下被删除：
```
程序调用HttpSession.invalidate()
距离上一次收到客户端发送的session id时间间隔超过了session的最大有效时间
服务器进程被停止
```

注意：

1. 客户端只保存 sessionID 到 cookie 中，而不会保存 session。
2. 关闭浏览器只会使存储在客户端浏览器内存中的session cookie失效，不会使服务器端的session对象失效
3. 同样也不会使已经保存到硬盘上的持久化cookie消失。

### 客户端用cookie保存了sessionID时

```
　客户端用cookie保存了sessionID，当我们请求服务器的时候，会把这个sessionID一起发给服务器，服务器会到内存中搜索对应的sessionID，如果找到了对应的 sessionID，说明我们处于登录状态，有相应的权限；如果没有找到对应的sessionID，这说明：要么是我们把浏览器关掉了（后面会说明为什么），要么session超时了（没有请求服务器超过20分钟），session被服务器清除了，则服务器会给你分配一个新的sessionID。你得重新登录并把这个新的sessionID保存在cookie中。
```

        在没有把浏览器关掉的时候（这个时候假如已经把sessionID保存在cookie中了）这个sessionID会一直保存在浏览器中，每次请求的时候都会把这个sessionID提交到服务器，所以服务器认为我们是登录的；当然，如果太长时间没有请求服务器，服务器会认为我们已经所以把浏览器关掉了，这个时候服务器会把该sessionID从内存中清除掉，这个时候如果我们再去请求服务器，sessionID已经不存在了，所以服务器并没有在内存中找到对应的 sessionID，所以会再产生一个新的sessionID，这个时候一般我们又要再登录一次。 

参考 https://blog.csdn.net/weixin_43625577/article/details/92393581
