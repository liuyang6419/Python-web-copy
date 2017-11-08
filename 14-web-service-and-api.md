# Web服务和API


服务器端向着Web服务或应用编程接口（Application Programming Interface, API）转变，服务端所有商业逻辑以RESTful API的方式提供给客户端，客户端通过HTTP与服务器端进行交互。这种统一的机制既能减小开发的复杂度，又具有良好的扩展性。


<a id="org4b634e5"></a>

## 什么是REST

Roy Thomas Fielding在他2000年的[博士论文](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)中介绍了他对互联网软件的架构方式，定名为 **REST** ，即 **Representational State Transfer** （常见的翻译是“表现层状态转移”）的缩写。

表现层实际上指的是 **资源** 的表现层。 **资源** 是指Web上一切可识别、可命名、可找到并被处理的实体。<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>用一个 **URI（统一资源标识符，Uniform Resource Identifier）** 指向资源，使用HTTP请求方法操作资源。 URI可进一步划分为统一资源名（URN，代表资源的名字）和统一资源定位符（URL，代表资源的地址），一般可以用URL替代URI。“资源”是一种信息实体，它可以有多种外在表现形式。我们把"资源"具体呈现出来的形式，叫做它的“表现层”（Representation）。

REST架构风格最重要的5个特征：

1.  客户端——服务器

    以 **Client/Server** 的架构形式提供了基本的分布式，客户端发起请求，服务器端响应或拒绝请求，如果出错则返回错误信息，异常由客户端处理。

2.  无状态

    客户端发出的请求中必须包含所有必要的信息。如果使用基于服务器端的会话，需要保证指定会话会使用同一个服务器响应所有请求，或者创建一个可供所有服务器访问的公用的会话存储区，对每个请求都额外访问这个集中式的数据存储区获得会话状态。

3.  缓存

    缓存可以抵消一部分无状态带来的影响，客户端（或客户端和服务器之间的中间服务）可以使用缓存。

4.  接口统一

    客户端访问服务器资源时使用的协议必须一致，定义良好，并已经标准化。REST Web服务最常使用的统一接口是HTTP协议。

5.  系统分层

    将系统划分为几个部分，每个部分负责相对单一的职责，通过上层对下层的依赖和调用组成一个完整的系统。通常可划分如下三层：

    -   应用层：负责返回JSON数据和其他业务逻辑。

    -   服务层：为应用层提供服务支持，比如账号系统、文件托管服务等。

    -   数据访问层：提供数据访问和存储的服务，如数据库、缓存系统、文件系统、搜索引擎等。

    系统分层可以提高性能、稳定性和伸缩性。

REST就是上述一系列设计约束的集合，如果一个架构符合REST原则，就称它为RESTful架构。


<a id="orgb682ccd"></a>

## RESTful API设计指南

本节将探讨如何设计一套合理、好用的API。API一旦发布其结构将很难修改，因此设计和实现一个符合规范、灵活、友好的API是一件非常重要的事情。


<a id="org0b4aed1"></a>

### 版本

应该将API的版本号放入URL。例如：

    https://example.com/api/v1/


<a id="org1912e97"></a>

### 路径（Endpoint）

又称“端点”或“终点”，表示API的具体网址。

在RESTful架构中，每个网址代表一种资源（resource），所以网址中不能有动词，只能有名词， 而且所用的名词往往与数据库的表名对应。一般来说，数据库中的表都是同种记录的"集合"（collection），所以API中的名词也应该使用复数。

举例来说，有一个API提供动物园（zoo）的信息，还包括各种动物和雇员的信息，则它的路径可以设计成下面这样：

    https://example.com/api/v1/zoos
    https://example.com/api/v1/animals
    https://example.com/api/v1/employees


<a id="org078ee5f"></a>

### HTTP动词

对于资源的具体操作类型，由HTTP动词表示。

常用的HTTP动词有下面五个（括号里是对应的SQL命令）：

-   **GET（SELECT）** ：从服务器取出资源（一项或多项）。
-   **POST（CREATE）** ：在服务器新建一个资源。
-   **PUT（UPDATE）** ：在服务器更新资源（客户端提供改变后的完整资源）。
-   **PATCH（UPDATE）** ：在服务器更新资源（客户端提供改变的属性）。
-   **DELETE（DELETE）** ：从服务器删除资源。

还有两个不常用的HTTP动词：

-   **HEAD** ：获取资源的元数据。
-   **OPTIONS** ：获取信息，关于资源的哪些属性是客户端可以改变的。

下面是一些例子：

-   <code>GET /zoos</code>：列出所有动物园
-   <code>POST /zoos</code>：新建一个动物园
-   <code>GET /zoos/ID</code>：获取某个指定动物园的信息
-   <code>PUT /zoos/ID</code>：更新某个指定动物园的信息（提供该动物园的全部信息）
-   <code>PATCH /zoos/ID</code>：更新某个指定动物园的信息（提供该动物园的部分信息）
-   <code>DELETE /zoos/ID</code>：删除某个动物园
-   <code>GET /zoos/ID/animals</code>：列出某个指定动物园的所有动物
-   <code>DELETE /zoos/ID/animals/ID</code>：删除某个指定动物园的指定动物


<a id="org98e31e1"></a>

### 状态码

服务器向用户返回的状态码和提示信息，常见的有以下一些（方括号中是该状态码对应的HTTP动词）。

-   **200 OK** - **[GET]** ：服务器成功返回用户请求的数据，该操作是幂等的（Idempotent）。
-   **201 CREATED** - **[POST/PUT/PATCH]** ：用户新建或修改数据成功。
-   **202 Accepted** - **[\*]** ：表示一个请求已经进入后台排队（异步任务）。
-   **204 NO CONTENT** - **[DELETE]** ：用户删除数据成功。
-   **400 INVALID REQUEST** - **[POST/PUT/PATCH]** ：用户发出的请求有错误，服务器没有进行新建或修改数据的操作，该操作是幂等的。
-   **401 Unauthorized** - **[\*]** ：表示用户没有权限（令牌、用户名、密码错误）。
-   **403 Forbidden** - **[\*]** ：表示用户得到授权（与401错误相对），但是访问是被禁止的。
-   **404 NOT FOUND** - **[\*]** ：用户发出的请求针对的是不存在的记录，服务器没有进行操作，该操作是幂等的。
-   **406 Not Acceptable** - **[GET]** ：用户请求的格式不可得（比如用户请求JSON格式，但是只有XML格式）。
-   **410 Gone** - **[GET]** ：用户请求的资源被永久删除，且不会再得到的。
-   **422 Unprocesable entity** - **[POST/PUT/PATCH]** ：当创建一个对象时，发生一个验证错误。
-   **500 INTERNAL SERVER ERROR** - **[\*]** ：服务器发生错误，用户将无法判断发出的请求是否成功。

状态码的完整列表参见[这里](https://www.w3.org/Protocols/rfc2616/rfc2616-sec10.html)。


<a id="orgc164304"></a>

### 错误处理

如果状态码是 **4xx** ，就应该向用户返回出错信息。 一般来说，返回的信息中将<code>error</code>作为键名，出错信息作为键值即可。

```js
{
    error: "Invalid API key"
}
```


<a id="org9b52750"></a>

### Hypermedia API

RESTful API最好做到Hypermedia，即返回结果中提供链接，链向其他API方法， 使得用户不查文档，也知道下一步应该做什么。

Hypermedia API的设计被称为[HATEOAS](https://en.wikipedia.org/wiki/HATEOAS)。 Github的API就是这种设计，访问[api.github.com](https://api.github.com)会得到一个所有可用API的网址列表。

```js
{
  "current_user_url": "https://api.github.com/user",
  "current_user_authorizations_html_url": "https://github.com/settings/connections/applications{/client_id}",
  "authorizations_url": "https://api.github.com/authorizations",
  // ...
}
```

从上面可以看到，如果想获取当前用户的信息，应该去访问[api.github.com/user](https://api.github.com/user)， 然后就得到了下面结果：

```js
{
  "message": "Requires authentication",
  "documentation_url": "https://developer.github.com/v3"
}
```

服务器给出了提示信息以及文档的网址。

还有两处容易出错的地方需要留意：

-   使用“201 Created”响应时，应该在响应头中带<code>Location</code>，指向新建地址的资源。
-   使用“405 Method Not Allowed”响应时，应带有<code>Allow</code>头，告诉客户端对该资源有效的HTTP方法。


<a id="org361a9cc"></a>

### 过滤信息

API应该提供参数来过滤返回结果、排序和搜索。

下面是一些常见的参数：

-   `?limit=10` ：指定返回记录的数量
-   `?offset=10` ：指定返回记录的开始位置。
-   `?page=2&per_page=100` ：指定第几页，以及每页的记录数。
-   `?sortby=name&order=asc` ：指定返回结果按照哪个属性排序，以及排序顺序。
-   `?sort=name,desc` ：多个排序条件组合
-   `?animal_type_id=1` ：指定筛选条件

参数的设计允许存在冗余，即允许API路径和URL参数偶尔有重复。比如，<code>GET /zoo/ID/animals</code>与<code>GET /animals?zoo_id=ID</code>的含义是相同的。


<a id="orgdbc07e0"></a>

### URI失效和迁移

随着业务发展，会出现一些API失效或者迁移。对失效的API，应返回“404 Not Found”或“410 Gone”；对迁移的API，返回301重定向。


<a id="org6f7cc54"></a>

## 提供REST Web服务

使用Flask创建REST Web服务很简单。请求中包含的JSON数据可通过<code>request.json</code>这个Python字典获取，并且需要包含JSON的响应可以使用<code>jsonify()</code>从Python字典中生成。

接下来将介绍如何使用Flask创建一个REST Web服务，以便让客户端访问博客文章及相关资源。


<a id="org579b613"></a>

### 创建API蓝图

API蓝图的结构如下：

    ├─ app
       ├─ api_1_0
          ├── __init__.py
          ├── authentication.py
          ├── comments.py
          ├── decorators.py
          ├── errors.py
          ├── posts.py
          └── users.py

⚠️ API包的名字中有一个版本号。

在这个API蓝图中，各资源分别在不同的模块中实现。蓝图中还包含处理认证、错误以及提供自定义装饰器的模块。

蓝图的构造文件如下：

```python
# app/api_1_0/__init__.py
from flask import Blueprint

api = Blueprint('api', __name__)

from . import authentication, posts, users, comments, errors
```

注册API蓝图的代码如下：

```python
# app/__init__.py
def create_app(config_name):
    # ...
    from .api_1_0 import api as api_1_0_blueprint
    app.register_blueprint(api_1_0_blueprint, url_prefix='/api/v1.0')
    # ...
```


<a id="orgc19b322"></a>

### 错误处理

现在需要改进一下404和500错误处理，使得程序能根据客户端请求的格式改写响应返回相应格式的错误消息，这种技术称为内容协商。

下面是改进后的404错误处理程序，500错误处理程序与之类似：

```python
# app/main/errors.py
@main.app_errorhandler(404)
def page_not_found(e):
    if request.accept_mimetypes.accept_json and \
       not request.accept_mimetypes.accept_html:
        response = jsonify({'error': 'not found'})
        response.status_code = 404
        return response
    return render_template('404.html'), 404
```

上述代码会检查<code>Accept</code>请求首部（Werkzeug将其解码为<code>request.accept_mimetypes</code>），根据首部的值决定客户端期望接收的响应格式。浏览器一般不限制响应的格式，所以只为接受JSON格式而不接受HTML格式的客户端生成JSON格式响应。

其他状态码都由Web服务程序生成，可在蓝图的<code>errors.py</code>模块作为辅助函数实现。下面是403状态码的错误处理程序：

```python
# app/api_1_0/errors.py
def forbidden(message):
    response = jsonify({'error': 'forbidden', 'message': message})
    response.status_code = 403
    return response
```

Web服务的视图函数通过调用这些辅助函数生成错误响应。


<a id="org91fd756"></a>

### 认证用户

和普通的Web程序一样，Web服务也需要保护信息，确保未经授权的用户无法访问。

因为REST架构基于HTTP协议，所以发送密令的最佳方式是使用HTTP认证，基本认证和摘要认证都可以。在HTTP认证中，用户密令包含在请求的<code>Authorization</code>首部中。

Flask-HTTPAuth扩展提供了一个装饰器，将协议的细节隐藏其中，类似Flask-Login提供的<code>login_required</code>装饰器。

下面的代码展示如何初始化Flask-HTTPAuth扩展，以及如何在回调函数中验证密令。

```python
# app/api_1_0/authentication.py
from flask_httpauth import HTTPBasicAuth

auth = HTTPBasicAuth()

@auth.verify_password
def verify_password(email, password):
    if email == '':
        g.current_user = AnonymousUser()
        return True
    user = User.select().where(User.email == email_or_token).first()
    if not user:
        return False
    g.current_user = user
    return user.verify_password(password)
```

这种用户认证方法只在API蓝图中使用，所以Flask-HTTPAuth扩展只在蓝图包中初始化，而不像其他扩展那样要在程序包中初始化。

电子邮件和密码使用<code>User</code>模型中现有的方法验证。如果登录密令正确，这个验证回调函数就返回<code>True</code>，否则返回<code>False</code>。API蓝图也支持匿名用户访问，此时客户端发送的电子邮件字段必须为空。

验证回调函数把通过认证的用户保存在全局对象<code>g</code>中，这样视图函数便能进行访问。注意，匿名登录时，这个函数返回<code>True</code>并把Flask-Login提供的<code>AnonymousUser</code>类实例赋值给<code>g.current_user</code>。

⚠️ 由于每次请求时都要传送用户密令，所以API路由最好通过安全的HTTPS提供，加密所有的请求和响应。

如果认证密令不正确，服务器向客户端返回401错误。这个错误响应可以自定义：

```python
# app/api_1_0/errors.py
from flask import jsonify


def unauthorized(message):
    response = jsonify({'error': 'unauthorized', 'message': message})
    response.status_code = 401
    return response
```

```python
# app/api_1_0/authentication.py
from .errors import unauthorized


@auth.error_handler
def auth_error():
    return unauthorized('Invalid credentials')
```

为保护路由，可使用装饰器<code>auth.login_required</code>：

```python
@api.route('/posts/')
@auth.login_required
def get_posts():
    pass
```

API蓝图中所有路由都要使用相同的方式进行保护，可以在<code>before_request</code>处理程序中使用一次<code>login_required</code>装饰器，应用到整个蓝图：

```python
# app/api_1_0/authentication.py
from .errors import forbidden

@api.before_request
@auth.login_required
def before_request():
    if not g.current_user.is_anonymous and \
       not g.current_user.confirmed:
        return forbidden('Unconfirmed account')
```

现在，API蓝图中的所有路由都能进行自动认证。作为附加认证，<code>before_request</code>处理程序还会拒绝已通过认证但没有确认账户的用户。


<a id="org7e7c1b8"></a>

### 基于令牌的认证

为了避免总是发送敏感信息，可以提供一种基于令牌的认证方案。

使用基于令牌的认证方案时，客户端要先把登录密令发送给一个特殊的URL，从而生成认证令牌。一旦客户端获得令牌，就可用令牌代替登录密令认证请求。出于安全考虑，令牌有过期时间。令牌过期后，客户端必须重新发送登录密令以生成新令牌。令牌落入他人之手所带来的安全隐患受限于令牌的短暂使用期限。为了生成和验证认证令牌，要在<code>User</code>模型中定义两个新方法。这两个新方法用到了<code>itsdangerous</code>包，代码如下：

```python
# app/models.py
class User(UserMixin, db.Model):
    # ...
    def generate_auth_token(self, expiration):
        s = Serializer(current_app.config['SECRET_KEY'],
                       expires_in=expiration)
        return s.dumps({'id': self.id}).decode('ascii')

    @staticmethod
    def verify_auth_token(token):
        s = Serializer(current_app.config['SECRET_KEY'])
        try:
            data = s.loads(token)
        except:
            return None
        return User.select().where(User.id == data['id']).first()
```

<code>generate_auth_token()</code>方法使用编码后的用户<code>id</code>字段值生成一个签名令牌，还指定了以秒为单位的过期时间。<code>verify_auth_token()</code>方法接受的参数是一个令牌，如果令牌可用就返回对应的用户。<code>verify_auth_token()</code>是静态方法，因为只有解码令牌后才能知道用户是谁。

为了能够认证包含令牌的请求，必须修改<code>verify_password</code>回调，除了普通的密令之外，还要接受令牌。修改后的回调如下所示：

```python
# app/api_1_0/authentication.py
@auth.verify_password
def verify_password(email_or_token, password):
    if email_or_token == '':
        g.current_user = AnonymousUser()
        return True
    if password == '':
        g.current_user = User.verify_auth_token(email_or_token)
        g.token_used = True
        return g.current_user is not None
    user = User.select().where(User.email == email_or_token).first()
    if not user:
        return False
    g.current_user = user
    g.token_used = False
    return user.verify_password(password)
```

新版本中，第一个认证参数可以是电子邮件地址或认证令牌。如果这个参数为空，那就和之前一样，假定是匿名用户。如果密码为空，那就假定<code>email_or_token</code>参数提供的是令牌，按照令牌的方式进行认证。如果两个参数都不为空，假定使用常规的邮件地址和密码进行认证。在这种实现方式中，基于令牌的认证是可选的，由客户端决定是否使用。为了让视图函数能区分这两种认证方法，添加了<code>g.token_used</code>变量。

把认证令牌发送给客户端的路由也要添加到API蓝图中：

```python
# app/api_1_0/authentication.py
@api.route('/token')
def get_token():
    if g.current_user.is_anonymous or g.token_used:
        return unauthorized('Invalid credentials')
    return jsonify({'token': g.current_user.generate_auth_token(
        expiration=3600), 'expiration': 3600})
```

由于这个路由也在蓝图中，所以添加到<code>before_request</code>处理程序上的认证机制也会用在这个路由上。为了避免客户端使用旧令牌申请新令牌，要在视图函数中检查<code>g.token_used</code>变量的值，如果使用令牌进行认证就拒绝请求。这个视图函数返回JSON格式的响应，其中包含了过期时间为1小时的令牌。JSON格式的响应也包含过期时间。


<a id="orgbe3b156"></a>

### 资源和JSON的序列化转换

开发Web程序时，经常需要在资源的内部表示和JSON之间进行转换。JSON是 HTTP 请求和响应使用的传输格式。

下面是是新添加到Post类中的<code>to_json()</code>方法，用来把文章转换成JSON格式的序列化字典：

```python
# app/models.py
class Post(db.Model):
    # ...
    def to_json(self):
        json_post = {
            'url': url_for('api.get_post', id=self.id, _external=True),
            'body': self.body,
            'body_html': self.body_html,
            'timestamp': self.timestamp,
            'author': url_for('api.get_user', id=self.author_id,
                              _external=True),
            'comments': url_for('api.get_post_comments', id=self.id,
                                _external=True),
            'comment_count': self.comments.count()
        }
        return json_post
```

<code>url</code>、<code>author</code>和<code>comments</code>字段要分别返回各自资源的URL，使用<code>url_for()</code>生成，所调用的路由即将在 API 蓝图中定义。

⚠️ 上述所有<code>url_for()</code>方法都指定了参数 `_external=True` ，这么做是为了生成完整的URL。

<code>User</code>模型的<code>to_json()</code>方法可以按照<code>Post</code>模型的方式定义。

把JSON转换成模型时面临的问题是，客户端提供的数据可能无效、错误或者多余。

下面是从JSON格式数据创建<code>Post</code>模型实例的方法，其可以从JSON格式数据创建一篇博客文章：

```python
# app/models.py
from app.exceptions import ValidationError

class Post(db.Model):
    # ...
    @staticmethod
    def from_json(json_post):
        body = json_post.get('body')
        if body is None or body == '':
            raise ValidationError('post does not have a body')
        return Post(body=body)
```

如果没有<code>body</code>字段或者其值为空，<code>from_json()</code>方法会抛出<code>ValidationError</code>异常。在这种情况下，抛出异常才是处理错误的正确方式，因为<code>from_json()</code>方法并没有掌握处理问题的足够信息，唯有把错误交给调用者，由上层代码处理这个错误。其中<code>ValidationError</code>类是Python中<code>ValueError</code>类的简单子类。

程序需要向客户端提供适当的响应以处理这个异常。为了避免在视图函数中编写捕获异常的代码，可创建一个全局异常处理程序。对于<code>ValidationError</code>异常，其处理程序如下：

```python
# app/api_1_0/errors.py

@api.errorhandler(ValidationError)
def validation_error(e):
    return bad_request(e.args[0])
```

这里使用的<code>errorhandler</code>装饰器和注册HTTP状态码处理程序时使用的是同一个， 只不过此时接收的参数是<code>Exception</code>类，只要抛出了指定类的异常，就会调用被修饰的函数。

使用这个技术时，视图函数中得代码可以写得十分简洁明，而且无需检查错误。


<a id="orga2b9174"></a>

### 实现资源端点

现在来实现用于处理不同资源的路由。先来处理 **GET** 请求，下面是博客文章的两个GET请求处理的 视图函数：

```python
# app/api_1_0/posts.py
import playhouse.flask_utils as futils


@api.route('/posts')
def get_posts():
    posts = Post.timeline()
    return jsonify({'posts': [post.to_json() for post in posts]})


@api.route('/posts/<int:id>')
def get_post(id):
    post = futils.get_object_or_404(Post.select(), (Post.id == id))
    return jsonify(post.to_json())
```

第一个路由处理获取文章集合的请求。第二个路由返回单篇博客文章。

博客文章资源的POST请求处理程序把一篇新博客文章插入数据库。路由的定义如下：

```python
# app/api_1_0/posts.py
@api.route('/posts/', methods=['POST'])
@permission_required(Permission.WRITE_ARTICLES)
def new_post():
    post = Post.from_json(request.json)
    post.author = g.current_user
    post.save()
    post.update_body_html()
    post = post.refresh()
    return jsonify(post.to_json()), 201, \
        {'Location': url_for('api.get_post', id=post.id, _external=True)}
```

使用<code>permission_required</code>装饰器，确保通过认证的用户有写博客文章的权限。博客文章从JSON数据中创建，其作者就是通过认证的用户。这个模型写入数据库之后，会返回 **201** 状态码，并把 **Location** 首部的值设为刚创建的这个资源的URL。

为便于客户端操作，响应的主体中包含了新建的资源，而无需在创建资源后再立即发起一个GET请求以获取资源。

用来防止未授权用户创建新博客文章的<code>permission_required</code>修饰器和程序中使用的类似，但会针对 API 蓝本进行自定义。

```python
# app/api_1_0/decorators.py
from functools import wraps
from flask import g
from .errors import forbidden


def permission_required(permission):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not g.current_user.can(permission):
                return forbidden('Insufficient permissions')
            return f(*args, **kwargs)
        return decorated_function
    return decorator
```

博客文章 **PUT** 请求的处理程序用来更新现有资源：

```python
# app/api_1_0/posts.py
@api.route('/posts/<int:id>', methods=['PUT'])
@permission_required(Permission.WRITE_ARTICLES)
def edit_post(id):
    post = futils.get_object_or_404(Post.select(), (Post.id == id))
    if g.current_user != post.author and \
       not g.current_user.can(Permission.ADMINISTER):
        return forbidden('Insufficient permissions')
    post.body = request.json.get('body', post.body)
    post.save()
    post.update_body_html()
    post = post.refresh()
    return jsonify(post.to_json())
```

修饰器用来检查用户是否有写博客文章的权限，但为了确保用户能编辑博客文章，这个函数还要保证用户是文章的作者或者是管理员。这个检查直接添加到视图函数中。

因为程序不允许删除文章，所以没必要实现 **DELETE** 请求方法的处理程序。

用户资源和评论资源的处理程序实现方式类似。

下表列出了这个程序要实现的资源，可以到代码仓库中看完整实现。

| 资源URL                     | 方法      | 说明                         |
|-----------------------------|-----------|------------------------------|
| `/users/<int:id>`           | GET       | 一个用户                     |
| `/users/<int:id>/posts/`    | GET       | 一个用户发布的博客文章       |
| `/users/<int:id>/timeline/` | GET       | 一个用户所关注用户发布的文章 |
| `/posts/`                   | GET、POST | 所有博客文章                 |
| `/posts/<int:id>`           | GET、PUT  | 一篇博客文章                 |
| `/posts/<int:id/>comments/` | GET、POST | 一篇博客文章中的评论         |
| `/comments/`                | GET       | 所有评论                     |
| `/comments/<int:id>`        | GET       | 一篇评论                     |

⚠️ 这些资源只允许客户端实现Web程序提供的部分功能。支持的资源可以按需扩展，比如说提供关注者资源、支持评论管理，以及实现客户端需要的其他功能。


<a id="org147f580"></a>

### 分页大型资源集合

对大型资源集合来说，获取集合的GET请求消耗很大，而且难以管理。和Web程序一样，Web服务也可以对集合进行分页。

下面是分页博客列表的实现：

```python
# app/api_1_0/posts.py

@api.route('/posts/')
def get_posts():
    pagination = Pagination(Post.timeline(),
                            current_app.config['FLASKR_POSTS_PER_PAGE'],
                            check_bounds=False)
    posts = pagination.items
    prev = None
    if pagination.has_prev:
        prev = url_for('api.get_posts', page=pagination.page-1, _external=True)
    next = None
    if pagination.has_next:
        next = url_for('api.get_posts', page=pagination.page+1, _external=True)
    return jsonify({
        'posts': [post.to_json() for post in posts],
        'prev': prev,
        'next': next,
        'count': pagination.total
    })
```

JSON格式响应中的<code>posts</code>字段依旧包含各篇文章，但现在这只是完整集合的一部分。 如果资源有上一页和下一页，<code>prev</code>和<code>next</code>字段分别表示上一页和下一页资源的URL。<code>count</code>是集合中博客文章的总数。

这种分页技术可应用于所有返回集合的路由。

**🔖 执行<code>git checkout 14a</code>签出程序的这个版本。** ⚠ 此版本需要安装依赖。


<a id="org1077fa1"></a>

### 使用HTTPie测试Web服务

测试Web服务时必须使用HTTP客户端。最常使用的两个在命令行中测试Web服务的客户端是 [curl](https://curl.haxx.se) 和 [HTTPie](https://httpie.org) 。后者的命令行更简洁，可读性也更高。

使用pip安装HTTPie：

```sh
(flaskr_env3) $ pip install httpie
```

GET请求按照下面的方式发起：

```sh
(flaskr_env3) $ http --json --auth <email>:<password> GET \
> http://127.0.0.1:5000/api/v1.0/posts/
```

    HTTP/1.0 200 OK
    Content-Length: 6958
    Content-Type: application/json
    Date: Mon, 18 Sep 2017 06:16:12 GMT
    Server: Werkzeug/0.12.2 Python/3.6.2

    {
        "count": 37,
        "next": "http://127.0.0.1:5000/api/v1.0/posts/?page=2",
        "posts": [
            ...
        ],
        "prev": null
    }

注意响应中的分页链接。

匿名用户可发送空邮件地址和密码以发起相同的请求：

```sh
(flaskr_env3) $ http --json --auth : GET http://127.0.0.1:5000/api/v1.0/posts/
```

下面这个命令发送POST请求以添加一篇新博客文章：

```sh
(flaskr_env3) $ http --auth <email>:<password> --json POST \
> http://127.0.0.1:5000/api/v1.0/posts/ \
> "body=I'm adding a post from the *command line*."
```

    HTTP/1.0 201 CREATED
    Content-Length: 383
    Content-Type: application/json
    Date: Mon, 18 Sep 2017 06:22:20 GMT
    Location: http://127.0.0.1:5000/api/v1.0/posts/38
    Server: Werkzeug/0.12.2 Python/3.6.2

    {
        "author": "http://127.0.0.1:5000/api/v1.0/users/1",
        "body": "I'm adding a post from the *command line*.",
        "body_html": "<p>I'm adding a post from the <em>command line</em>.</p>",
        "comment_count": 0,
        "comments": "http://127.0.0.1:5000/api/v1.0/posts/38/comments/",
        "timestamp": "Mon, 18 Sep 2017 06:22:19 GMT",
        "url": "http://127.0.0.1:5000/api/v1.0/posts/38"
    }

要想使用认证令牌，可向<code>/api/v1.0/token/</code>发送请求：

```sh
(flaskr_env3) $ http --auth <email>:<password> --json GET \
> http://127.0.0.1:5000/api/v1.0/token
```

    HTTP/1.0 200 OK
    Content-Length: 163
    Content-Type: application/json
    Date: Mon, 18 Sep 2017 06:24:11 GMT
    Server: Werkzeug/0.12.2 Python/3.6.2

    {
        "expiration": 3600,
        "token": "eyJhbGciOiJIUzI1NiIsImlhdCI6MTUwNTcxNTg1MSwiZXhwIj..."
    }

在接下来的1小时中，这个令牌可用于访问API，请求时要和空密码一起发送：

```sh
(flaskr_env3) $ http --json --auth eyJhb...: GET http://127.0.0.1:5000/api/v1.0/posts/
```

令牌过期后，请求会返回 **401** 错误，表示需要获取新令牌。

## 脚注

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> 原文是“Any information that can be named can be a resource.”
