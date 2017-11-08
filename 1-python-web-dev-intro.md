<a id="orgd176733"></a>
# Python Web开发初识


<a id="orgb777448"></a>

## HTTP client-server

![img](../images/http_client-server.png "HTTP Client-Server")

-   通过请求和响应的交换达成通信
-   不保存通信状态（stateless）
-   使用 **URI** 定位互联网上的资源
-   请求资源时使用方法下达命令（GET、POST、HEAD等）
-   通过持久连接节省通信量
-   使用cookie来进行状态管理


<a id="org21ca446"></a>

### HTTP请求

    GET / HTTP/1.1
    Connection: close
    Host: httpbin.org
    User-agent: HTTPie/0.9.9
    Accept: */*
    Accept-Encoding: gzip, deflate
    Accept-Language: en
    Accept-Charset: *, utf-8

    Optional data
    ...

-   第一行定义 **请求类型** 、 **文档（选择符）** 和 **协议版本**
-   接着是报头行，包括各种有关客户端的信息
-   报头行后面是一个空白行，表示报头行结束
-   之后是发送表单的信息或者上传文件的事件中可能出现的数据
-   报头的每一行都应该使用回车符或者换行符（'\r\n'）终止

下表是常见HTTP请求方法：

| 方法 | 描述      |
|------|-----------|
| GET  | 获取文档  |
| POST | 将数据发布到表单 |
| HEAD | 仅返回报头信息 |
| PUT  | 将数据上传到服务器 |
| …    | …         |


<a id="org96b8141"></a>

### HTTP响应

    HTTP/1.1 200 OK
    Connection: keep-alive
    Content-Length: 580
    Content-Type: application/json
    Date: Tue, 25 Apr 2017 04:28:37 GMT
    Server: gunicorn/19.7.1
    ...
    Header: data

    Data
    ...

-   第一行表示 **HTTP协议版本** 、 **成功代码** 和 **返回消息**
-   响应行之后是一系列报头字段， 包含返回文档的类型、文档大小、Web服务器软件、cookie等方面的信息
-   通过空白行结束报头
-   之后是所请求文档的原始数据

下表是HTTP常见状态码：

| 代码       | 描述    | 符号常量                |
|------------|---------|-------------------------|
| 成功代码（2xx） |         |                         |
| 200        | 成功    | OK                      |
| 201        | 创建    | CREATED                 |
| 202        | 接受    | ACCEPTED                |
| 204        | 无内容  | NO\_CONTENT             |
| 重定向（3xx） |         |                         |
| 300        | 多种选择 | MULTIPLE\_CHOICES       |
| 301        | 永久移动 | MOVED\_PERMANENTLY      |
| 302        | 可被303替代 | FOUND                   |
| 303        | 临时移动 | SEE\_OTHER              |
| 304        | 不修改  | NOT\_MODIFIED           |
| 客户端错误（4xx） |         |                         |
| 400        | 请求错误 | BAD\_REQUEST            |
| 401        | 未授权  | UNAUTHORIZED            |
| 403        | 禁止访问 | FORBIDDEN               |
| 404        | 未找到  | NOT\_FOUND              |
| 405        | 方法不允许 | METHOD\_NOT\_ALLOWED    |
| 服务器错误（5xx） |         |                         |
| 500        | 内部服务器错误 | INTERNAL\_SERVER\_ERROR |
| 501        | 未实现  | NOT\_IMPLEMENTED        |
| 502        | 网关错误 | BAD\_GATEWAY            |
| 503        | 服务不可用 | SERVICE\_UNAVAILABLE    |


<a id="org918498b"></a>

## Python3 标准Web库

-   <code>http</code>处理所有客户端—服务器HTTP请求的具体细节
    -   <code>client</code>处理客户端部分
    -   <code>server</code>提供了实现HTTP服务器的各种类
    -   <code>cookies</code>支持在服务器端处理HTTP cookie
    -   <code>cookiejar</code>支持在客户端存储和管理HTTP cookie

-   <code>urllib</code>基于<code>http</code>的高层库，用于编写与HTTP服务器等交互的客户端
    -   <code>request</code>处理客户端请求
    -   <code>response</code>处理服务器端响应
    -   <code>parse</code>用于操作URL字符串

