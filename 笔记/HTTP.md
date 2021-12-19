# 1. HTTP基础知识
- 每条由Web服务器返回的内容都是和它管理的某个文件相关联的，这些文件有唯一的名字：URL(Universal Resource Locator，通用资源定位符)
- HTTP使用TCP链接，但本身是**无状态**的。



# 2. DNS

# 3. HTTP报文结构
## 2.1 请求报文(Request)

![image.png](http://47.100.67.5:8090/upload/2021/05/image-d8ac3e21c24a439ebfc47e9c170afbda.png)

实际上很简单，主要分为 **请求行**、**请求头**、**请求体**

### 请求行
- 方法: GET/POST/DELETE/PUT
- URL: 统一资源定位符
- 版本号【如HTTP/1.1】

### 请求头
包括一堆key-value：

![image.png](http://47.100.67.5:8090/upload/2021/10/image-144c49143f4f4776a77dfb25692dbe07.png)
- Accept：给出客户能接受的媒体格式
- Accept-charset:给出客户端能处理的字符集
- Date：给出当前日期
- Cookie：存放token等
- UserAgent ：客户端使用的浏览器等信息
- Host：指明了请求将要发送到的服务器主机名和端口号（可能以域名的方式存在）
- Origin：----
- Referer：----
- Connection：（keep-alive）使用同一个TCP连接来发送和接收多个HTTP请求/应答，而不是为每一个新的请求/应答打开新的连接的方法。


### 请求体
![image.png](http://47.100.67.5:8090/upload/2021/10/image-a83e54807d4c4d04acd42a4dacf7bf16.png)

- form-data
就是 http 请求中的 multipart/form-data, 它会将表单的数据处理为一条消息，以标签为单元，用分隔符分开。既可以上传键值对，也可以上传文件。当上传的字段是文件时，会有 Content-Type 来说明文件类型；content-disposition，用来说明字段的一些信息；
由于有 boundary 隔离，所以 multipart/form-data **既可以上传文件，也可以上传键值对，它采用了键值对的方式，所以可以上传多个文件**。
![image.png](http://47.100.67.5:8090/upload/2021/10/image-2f370ab27c4a4be5a27b6c563e1a564e.png)

- x-www-form-urlencoded
就是 application/x-www-from-urlencoded, 会将表单内的数据转换为键值对，比如，name=java&age = 23

- raw(主要是json)
可以上传任意格式的**文本**，可以上传 text、json、xml、html 等


## 2.2 响应报文(Response)
![](http://47.100.67.5:8090/upload/2021/05/image-e5152685f80f4fb6ac48fc81fe92e30f.png)

也分为响应行，响应头，响应体

### 响应行
- 协议
- 状态码：
200 OK：成功
204 No Content：表示客户端发送给客户端的请求得到了成功处理，但在返回的响应报文中不含实体的主体部分（没有资源可以返回）；

301 Moved Permanently：永久性重定向，表示请求的资源被分配了新的URL，之后应使用更改的URL
302 Found：临时性重定向，表示请求的资源被分配了新的URL，希望本次访问使用新的URL；（301与302的区别：前者是永久移动，后者是临时移动（之后可能还会更改URL））


400 Bad Request：表示请求报文中存在语法错误；
401 Unauthorized：未经许可，需要通过HTTP认证；
403 Forbidden：服务器拒绝该次访问（访问权限出现问题）
404 请求地址错误

500 Inter Server Error：表示服务器在执行请求时发生了错误，也有可能是web应用存在的bug或某些临时的错误时；
503 Server Unavailable：表示服务器暂时处于超负载或正在进行停机维护，无法处理请求；

### 响应头
和请求头一致

#### 响应体
和请求体一致

# 4. 不同http请求类型（如GET/POST/PUT/DELETE）的区别 

> 对于tcp层来说，http协议的不同内容之间确实没什么区别，区别产生于客户端或服务端对于协议中不同内容的不同处理方式

## 浏览器层面发送HTTP请求

这里特指浏览器中非Ajax的HTTP请求，即从HTML和浏览器诞生就一直使用的HTTP协议中的GET/POST。

### 1. 请求定义不同
浏览器用GET请求来获取一个html页面/图片/css/js等资源；用POST来提交一个<form>表单，并得到一个结果的网页。浏览器将GET和POST定义为：

- GET
“读取“一个资源。比如Get到一个html文件。反复读取不应该对访问的数据有副作用。没有副作用被称为“幂等“（Idempotent)。**因为GET因为是读取，就可以对GET请求的数据做缓存。** 这个缓存可以做到浏览器本身上（彻底避免浏览器发请求），也可以做到代理上（如nginx），或者做到server端（用Etag，至少可以减少带宽消耗）

- POST
在页面里<form> 标签会定义一个表单。点击其中的submit元素会发出一个POST请求让服务器做一件事。这件事往往是有副作用的，不幂等的。不幂等也就意味着不能随意多次执行。因此也就不能缓存。比如通过POST下一个单，服务器创建了新的订单，然后返回订单成功的界面。这个页面不能被缓存。试想一下，如果POST请求被浏览器缓存了，那么下单请求就可以不向服务器发请求，而直接返回本地缓存的“下单成功界面”，却又没有真的在服务器下单。那是一件多么滑稽的事情。


### 2. 携带数据格式不同
GET和POST携带数据的格式也有区别。

当浏览器发出一个GET请求时，就意味着要么是用户自己在浏览器的地址栏输入，要不就是点击了html里a标签的href中的url。**所以其实并不是GET只能用url，而是浏览器直接发出的GET只能由一个url触发。** 所以没办法，GET上要在url之外带一些参数就只能依靠url上附带querystring。但是HTTP协议本身并没有这个限制。

浏览器的POST请求都来自表单提交。每次提交，表单的数据被浏览器用编码到HTTP请求的body里。浏览器发出的POST请求的body主要有有两种格式，一种是application/x-www-form-urlencoded用来传输简单的数据，大概就是"key1=value1&key2=value2"这样的格式。另外一种是传文件，会采用multipart/form-data格式。采用后者是因为application/x-www-form-urlencoded的编码方式对于文件这种二进制的数据非常低效。

因此我们一般会泛泛的说“GET请求没有body，只有url，请求数据放在url的querystring中；POST请求的数据在body中“。但这种情况仅限于浏览器发请求的场景。

## 接口层面发送HTTP请求

**在Momenta中涉及到的都是这个类型的HTTP**

这里是指通过浏览器的**Ajax api**，或者iOS/Android的App的**http client**，java的**commons-httpclient/okhttp**或者是**curl**，**postman**之类的工具发出来的GET和POST请求。
此时GET/POST不光能用在前端和后端的交互中，还能用在后端各个子服务的调用中（即当一种RPC协议使用）。尽管RPC有很多协议，比如thrift，grpc，但是http本身已经有大量的现成的支持工具可以使用，并且对人类很友好，容易debug。HTTP协议在微服务中的使用是相当普遍的。

此时，所有HTTP的请求大概都是遵从这样的格式：
```shell
<METHOD> <URL> HTTP/1.1\r\n
<Header1>: <HeaderValue1>\r\n
<Header2>: <HeaderValue2>\r\n
...
<HeaderN>: <HeaderValueN>\r\n
\r\n
<Body Data....>
```
例如：
![image.png](http://47.100.67.5:8090/upload/2021/10/image-0599b2721c17484fb038d362690cdb0b.png)

其中的“<METHOD>"可以是GET也可以是POST，或者其他的HTTP Method，如PUT、DELETE、OPTION。从协议本身看，并没有什么限制说GET一定不能没有body，POST就一定不能把参放到<URL>的querystring上。因此其实可以更加自由的去利用格式。
比如Elastic Search的_search api就用了带body的GET；也可以自己开发接口让POST一半的参数放在url的querystring里，另外一半放body里；你甚至还可以让所有的参数都放Header里——可以做各种各样的定制，只要请求的客户端和服务器端能够约定好。

当然，太自由也带来了另一种麻烦，开发人员不得不每次讨论确定参数是放url的path里，querystring里，body里，header里这种问题，太**低效**了。

于是就有了一些列接口规范/风格。其中名气最大的当属**REST**。REST充分运用GET、POST、PUT和DELETE，约定了这4个接口分别获取、创建、替换和删除“资源”，REST最佳实践还推荐在请求体使用json格式。这样仅仅通过看HTTP的method就可以明白接口是什么意思，并且解析格式也得到了统一。


### Restful

**幂等性：反复请求资源不应对业务逻辑产生副作用**

- GET--select
【GET】 + 【资源定位符】获取资源或者资源列表
```
GET http://foo.com/books          获取书籍列表
GET http://foo.com/books/:bookId  根据bookId获取一本具体的书
```

- POST--create
【POST】+ 【资源定位符】创建一个资源 
是一个**非幂等**方法，因为调用多次，都将产生新的资源。因为它会对资源本身产生影响，每次调用都会有新的资源产生，因此不满足幂等性。
```
POST http://foo.com/books
{
  "title": "大宽宽的碎碎念",
  "author": "大宽宽",
  ...
}
```

- PUT--update
【PUT】+ 【资源定位符】**创建或者替换**目标资源。
因为它直接把实体部分的数据替换到服务器的资源，我们多次调用它，只会产生一次影响，但是有相同结果的 HTTP 方法，所以满足幂等性。


- DELETE--delete
【DELETE】+ 【资源定位符】用于删除一个资源

![image.png](http://47.100.67.5:8090/upload/2021/10/image-b9d06604cae340e4a63d03e16524a74f.png)




# GET和POST的区别
![image.png](http://47.100.67.5:8090/upload/2021/10/image-ea317645fd2a461187225db64acebc0e.png)


# 接口幂等与操作幂等
- **接口幂等**：按照Restful的定义，GET/PUT/DELETE被定义为符合幂等，也就是说按照标准定义这些接口对应的操作应该是幂等的，接口幂等是一种规范。
- **操作幂等**：幂等的接口对应的操作一定是幂等的吗？这个也要看怎么实现。**接口幂等只是一种规范，就像事务隔离级别这种规范一样。离开接口的具体实现去谈接口幂等是没有意义的。** PUT也可以不是幂等，(比如相对值更新，update table set money=money+1)，POST也可以设计成幂等。

实现操作幂等