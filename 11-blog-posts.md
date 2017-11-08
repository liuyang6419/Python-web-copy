# 博客文章

本章要实现博客文章功能，允许用户阅读、撰写博客文章，我们关注的内容重点在 **模板重用** 、 **分页显示长列表** 以及 **富文本处理** 。


## 提交显示博客文章

先来定义文章模型`Post`：

```python
# app/models.py
import peewee as pw


class Post(db.Model):
    body = pw.TextField(null=True)
    timestamp = pw.DateTimeField(index=True, default=datetime.utcnow)
    author = pw.ForeignKeyField(User, related_name='posts', null=True)

    class Meta:
        db_table = 'posts'
```

博客文章包含正文、时间戳以及和`User`模型之间的一对多关系。`body`字段的定义类型是`TextField`，不限制长度。

在程序的首页要显示一个表单，包含一个多行文本输入框，用于输入博客文章的内容，另外还有一个提交按钮：

```python
# app/main/forms.py
class PostForm(FlaskForm):
    body = TextAreaField("What's on your mind?", validators=[Required()])
    submit = SubmitField('Submit')
```

`index()`视图函数处理这个表单并把以前发布的博客文章列表传给模板：

```python
# app/main/views.py
from .forms import PostForm


@main.route('/', methods=['GET', 'POST'])
def index():
    form = PostForm()
    if (current_user.is_authenticated and
        current_user.can(Permission.WRITE_ARTICLES) and
            form.validate_on_submit()):
        post = Post(body=form.body.data,
                    author=current_user._get_current_object())
        post.save()
        return redirect(url_for('.index'))
    posts = Post.select().order_by(Post.timestamp.desc())
    return render_template('index.html', form=form, posts=posts)
```

这个视图函数把表单和完整的博客文章列表传给模板。文章列表按照时间戳进行降序排列。如果博客文章表单提交的数据能通过验证就创建一个新`Post`实例。在发布新文章之前，要检查当前用户是否有写文章的权限。

新文章对象的`author`属性值为表达式`current_user._get_current_object()`。`current_user`由 Flask-Login 提供通过线程内的代理对象实现，需要真正的用户对象时 需调用`_get_current_object()`方法获取。

接着对显示博客文章的首页模板进行调整：

```django
{# app/templates/index.html #}
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

...
{% if current_user.is_authenticated %}
    <div>
        {% if current_user.can(Permission.WRITE_ARTICLES) %}
            {{ wtf.quick_form(form) }}
        {% endif %}
    </div>
{% endif %}
<ul class="posts">
    {% for post in posts %}
        <li class="post">
            <div class="post-thumbnail">
                <a href="{{ url_for('.user', username=post.author.username) }}">
                    <img class="img-rounded profile-thumbnail" src="{{ post.author.avatar(size=40) }}">
                </a>
            </div>
            <div class="post-content">
                <div class="post-date">{{ moment(post.timestamp).fromNow() }}</div>
                <div class="post-author"><a href="{{ url_for('.user', username=post.author.username) }}">{{ post.author.username }}</a></div>
                <div class="post-body">{{ post.body }}</div>
            </div>
        </li>
    {% endfor %}
</ul>
```

如果用户所属角色没有`WRITE_ARTICLES`权限，则经`User.can()`方法检查后，不会显示博客文章表单。博客文章列表会通过 CSS 调整显示成媒体列表的样式。

**🔖 执行`git checkout 11a`签出程序的这个版本。**


## 在资料页显示博客文章

将用户资料页改进一下，在上面显示该用户发布的博客文章列表。

获取博客文章的资料页路由如下：

```python
# app/main/views.py
@main.route('/user/<username>')
def user(username):
    user_query = User.select()
    user = futils.get_object_or_404(user_query, (User.username == username))
    posts = user.posts.order_by(Post.timestamp.desc())
    return render_template('user.html', user=user, posts=posts)
```

用户发布的博客文章列表通过`User.posts`关系获取，`User.posts`返回的是查询对象，因此可在其上调用过滤器，例如`order_by()`。

`user.html`模板也要使用一个HTML`<ul>`元素渲染博客文章。可以通过Jinja2 提供的`include()`指令来实现模板复用，将`index.html`和`user.html`中 相同的HTML片段移到新模板`_posts.html`中。

```django
{# app/templates/user.html #}
...
<h3>Posts by {{ user.username }}</h3>
{% include '_posts.html' %}
...
```

`_posts.html`模板名的下划线前缀是一种习惯用法，以区分独立模板和局部模板。

**🔖 执行`git checkout 11b`签出程序的这个版本。**


## 分页显示文章列表

随着网站的发展，博客文章的数量会不断增多，如果要在首页和资料页显示全部文章，浏览速度会变慢且不符合实际需求。这一问题的一种解决方法是分页显示数据，进行 片段式渲染。


### 创建虚拟博客文章数据

若想实现博客文章分页，我们需要一个包含大量数据的测试数据库。这里我们使用 **ForgeryPy** 包来自动生成虚拟信息数据。

严格来说，ForgeryPy 并不是这个程序的依赖，因为它只在开发过程中使用。为了区分生产环境的依赖和开发环境的依赖，可以把文件`requirements.in`换成`requirements`文件夹，在其中分别保存不同环境中的依赖。可以创建一个`dev.in`文件，列出开发过程中所需的依赖，再创建一个`prod.in`文件，列出生产环境所需的依赖。两个环境所需的大部分相同的依赖，可以创建一个`common.in`文件。这些依赖文件通过前面介绍的 pip-tools 工具进行管理，比如要生成`dev.txt`文件，可以通过 下面的命令获得：

```sh
pip-compile -o requirements/dev.txt requirements/common.in requirements/dev.in
```

下面的代码用来生成虚拟用户和博客文章：

```python
# app/models.py
class User(UserMixin, db.Model):
    # ...
    @staticmethod
    def generate_fake(count=100):
        from random import seed
        import forgery_py

        seed()
        fake_data = []
        for i in range(count):
            fake_data.append(
                dict(email=forgery_py.internet.email_address(),
                     username=forgery_py.internet.user_name(True),
                     password_hash=generate_password_hash(
                         forgery_py.lorem_ipsum.word()),
                     confirmed=True,
                     name=forgery_py.name.full_name(),
                     location=forgery_py.address.city(),
                     about_me=forgery_py.lorem_ipsum.sentence(),
                     member_since=forgery_py.date.date(True)))
        for idx in range(0, len(fake_data), 10):
            with db.database.atomic():
                User.insert_many(fake_data[idx:idx+10]).execute()


class Post(db.Model):
    # ...
    @staticmethod
    def generate_fake(count=100):
        from random import seed, randint
        import forgery_py

        seed()
        user_count = User.select().count()
        fake_data = []
        for i in range(count):
            u = User.select().offset(randint(0, user_count - 1)).first()
            fake_data.append(
                dict(body=forgery_py.lorem_ipsum.sentences(randint(1, 5)),
                     timestamp=forgery_py.date.date(True),
                     author=u))
        for idx in range(0, len(fake_data), 10):
            with db.database.atomic():
                Post.insert_many(fake_data[idx:idx+10]).execute()
```

这些虚拟对象的属性由 ForgeryPy 的随机信息生成器生成，其中的名字、电子邮件地址、 句子等属性看起来就像真的一样。

用户的电子邮件地址和用户名必须是唯一的，但 ForgeryPy 随机生成这些信息，因此有重复的风险。这个异常的处理方式是使用数据库事务，每次提交10个数据，循环操作，如果中间发生异常则只影响当次事务，不会把用户写入数据库，因此生成的虚拟用户总数可能会比预期少。

随机生成文章时要为每篇文章随机指定一个用户。使用`offset()`查询过滤器。这个过滤器会跳过参数中指定的记录数量。通过设定一个随机的偏移值，再调用`first()`方法，就能每次都获得一个不同的随机用户。

使用新添加的方法，可以在 flask shell 中轻易生成大量虚拟用户和文章：

```python
(flaskr_env3) $ flask shell
>>> User.generate_fake(50)
>>> Post.generate_fake(50)
```

现在运行程序，会看到首页中显示了一个很长的随机博客文章列表。

**🔖 执行`git checkout 11c`签出程序的这个版本。**


### 分页渲染数据

为了更好地处理分页数据以及后续添加分页导航，我们基于 Peewee 提供的`PaginatedQuery`并结合 Flask 实现了`Pagination`工具类，通过实例化这个类，我们就可以得到类似 Flask-SQLAlchemy 中的分页对象。下表是`Pagination`分页对象的属性：

| 属性       | 说明             |
|------------|------------------|
| `items`    | 当前页面中的记录 |
| `query`    | 分页的源查询     |
| `page`     | 当前页数         |
| `prev_num` | 上一页的页数     |
| `next_num` | 下一页的页数     |
| `has_next` | 如果有下一页，返回 `True` |
| `has_prev` | 如果有上一页，返回 `True` |
| `pages`    | 查询得到的总页数 |
| `per_page` | 每页显示的记录数量 |
| `total`    | 查询返回的记录总数 |

分页对象上还可调用一些方法：

-   `item_pages(left_edge=2, left_current=2, right_current=5, right_edge=2)`

    一个迭代器，返回一个在分页导航中显示的页数列表。这个列表的最左边显示`left_edge`页，当前页的左边显示`left_current`页，当前页的右边显示`right_current`页，最右边显示`right_edge`页。例如，在一个100页的列表中，当前页为第50页，使用默认配置，这个方法会返回以下页数:1、2、None、48、49、50、51、52、53、54、 55、None、99、100。`None`表示页数之间的间隔`next()`下一页的分页对象。

-   `prev()`

    上一页的分页对象

-   `next()`

    下一页的分页对象

下面是使用`Pagination`类实现分页显示博客文章列表的视图函数：

```python
# app/main/views.py
from utils.paginate_peewee import Pagination

@main.route('/', methods=['GET', 'POST'])
def index():
    form = PostForm()
    if (current_user.is_authenticated and
        current_user.can(Permission.WRITE_ARTICLES) and
            form.validate_on_submit()):
        post = Post(body=form.body.data,
                    author=current_user._get_current_object())
        post.save()
        return redirect(url_for('.index'))

    pagination = Pagination(Post.select().order_by(Post.timestamp.desc()),
                            current_app.config['FLASKR_POSTS_PER_PAGE'],
                            check_bounds=False)
    posts = pagination.items
    return render_template('index.html', form=form, posts=posts,
                           pagination=pagination)
```

渲染的页数将从请求的查询字符串`request.args`中获取，如果没有明确指定，则默认渲染第一页。`Pagination`第一个参数是一个查询对象（`SelectQuery`）或者模型类，`per_page`用来指定每页显示的记录数量，为了能够便利地配置每页显示的记录数量，参数`per_page`的值从程序的环境变量`FLASKR_POSTS_PER_PAGE`中读取。

另一个可选参数为`check_bounds`，当其设为`True`时，如果请求的页数超出了范围，则会返回 **404** 错误；如果设为`False`（默认值），页数超出范围时会返回一个 查询对象（`SelectQuery`）。

修改之后，首页中的文章列表只会显示有限数量的文章。若想查看第 2 页中的文章，要在浏览器地址栏中的 URL 后加上查询字符串`?page=2`。


### 添加分页导航

接着在模板中添加分页导航，下面是以Jinja2宏的形式实现的分页导航：

```django
{# app/templates/_macros.html #}
{% macro pagination_widget(pagination, endpoint) %}
    <ul class="pagination">
        <li{% if not pagination.has_prev %} class="disabled"{% endif %}>
            <a href="{% if pagination.has_prev %}{{ url_for(endpoint, page=pagination.prev_num, **kwargs) }}{% else %}#{% endif %}">
                &laquo;
            </a>
        </li>
        {% for p in pagination.iter_pages() %}
            {% if p %}
                {% if p == pagination.page %}
                    <li class="active">
                        <a href="{{ url_for(endpoint, page = p, **kwargs) }}">{{ p }}</a>
                    </li>
                {% else %}
                    <li>
                        <a href="{{ url_for(endpoint, page = p, **kwargs) }}">{{ p }}</a>
                    </li>
                {% endif %}
            {% else %}
                <li class="disabled"><a href="#">&hellip;</a></li>
            {% endif %}
        {% endfor %}
        <li{% if not pagination.has_next %} class="disabled"{% endif %}>
            <a href="{% if pagination.has_next %}{{ url_for(endpoint, page=pagination.next_num, **kwargs) }}{% else %}#{% endif %}">
                &raquo;
            </a>
        </li>
    </ul>
{% endmacro %}
```

这个宏创建了一个 Bootstrap 分页元素，即一个有特殊样式的无序列表，其中定义了下述页 面链接。

-   “上一页”链接。如果当前页是第一页，则为这个链接加上`disabled`类。
-   分页对象的`iter_pages()`迭代器返回的所有页面链接。这些页面被渲染成具有明确页数的链接，页数在`url_for()`的参数中指定。当前显示的页面使用`active`CSS 类高亮显示。页数列表中的间隔使用省略号表示。
-   “下一页”链接。如果当前页是最后一页，则会禁用这个链接。

Jinja2 宏的参数列表中不用加入`**kwargs`即可接收关键字参数。分页宏把接收到的所有关键字参数都传给了生成分页链接的`url_for()`方法。这种方式也可在路由中使用，例如包含一个动态部分的资料页。

`pagination_widget`宏可放在`index.html`和`user.html`中的`_posts.html`模板后面。

下面展示其在首页中的应用：

```django
{# app/templates/index.html #}
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}
{% import "_macros.html" as macros %}

...
{% include '_posts.html' %}
{% if pagination and pagination.pages > 1 %}
    <div class="pagination">
        {{ macros.pagination_widget(pagination, '.index') }}
    </div>
{% endif %}
...
```

**🔖 执行`git checkout 11d`签出程序的这个版本。**


## 富文本文章

本节将对输入文章的多行文本输入框进行升级，让其支持[Markdown](http://daringfireball.net/projects/markdown/)语法，还要添加富文本文章的预览功能。

要用到的包：

-   **[PageDown](https://github.com/StackExchange/pagedown):** 使用 JavaScript 实现的客户端 Markdown 到 HTML 的转换程序。
-   **[Flask-PageDown](https://github.com/miguelgrinberg/Flask-PageDown):** 为 Flask 包装的 PageDown，把 PageDown 集成到 Flask-WTF 表单中。
-   **[Markdown](https://pythonhosted.org/Markdown/):** 使用 Python 实现的服务器端 Markdown 到 HTML 的转换程序。
-   **[Bleach](https://bleach.readthedocs.io):** 使用 Python 实现的 HTML 清理器。

将这些包添加到依赖包文件中并进行安装。


### 预览Markdown

Flask-PageDown 扩展定义了一个`PageDownField`类，其和 WTForms 中的`TextAreaField`接口 一致。先来初始化扩展：

```python
# app/__init__.py
from flask_pagedown import PageDown
# ...
pagedown = Pagedown()
# ...
def create_app(config_name):
    # ...
    pagedown.init_app(app)
    # ...
```

修改`PostForm`表单中的`body`字段，启动Markdown的文章表单：

```python
# app/main/forms.py
from flask_pagedown.fields import PageDownField


class PostForm(FlaskForm):
    body = PageDownField("What's on your mind?", validators=[Required()])
    submit = SubmitField('Submit')
```

Markdown 预览使用 PageDown 库生成，Flask-PageDown 提供了一个模板宏，支持从CDN加载所需文件。下面是模板声明：

```django
{# app/templates/index.html #}
{% block scripts %}
	{{ super() }}
    {{ pagedown.include_pagedown() }}
{% endblock %}
```

此外，还要为预览部分添加CSS样式，可参考代码仓库中的代码。

**🔖 执行`git checkout 11e`签出程序的这个版本。** ⚠️ 安装程序所需的依赖。


### 服务器端处理富文本

提交表单后，POST请求只会发送纯Markdown文本，服务器上使用Markdown（使用Python编 写的Markdown到HTML转换程序）将其转换成HTML。得到HTML后，再使用Bleach进行清理，确保其中只包含几个允许使用的HTML标签。

转换后的博客文章HTML代码缓存在`Post`模型的一个新字段中，在模板中可以直接调用。文章的Markdown源文本还要保存在数据库中，以防需要编辑。

下面是在`Post`模型中处理Markdown文本：

```python
# app/decorators.py
def require_instance(func):
    @wraps(func)
    def inner(self, *args, **kwargs):
        if not self._get_pk_value():
            raise TypeError(
                "Can't call %s with a non-instance %s" % (
                    func.__name__, self.__class__.__name__))
        return func(self, *args, **kwargs)
    return inner
```

```python
# app/models.py
from markdown import markdown
import bleach

from .decorators import require_instance

class Post(db.Model):
    # ...
    body_html = pw.TextField(null=True)

    # ...
    @require_instance
    def update_body_html(self):
        allowed_tags = ['a', 'abbr', 'acronym', 'b', 'blockquote', 'code',
                        'em', 'i', 'li', 'ol', 'pre', 'strong', 'ul',
                        'h1', 'h2', 'h3', 'p']
        self.__class__.update(body_html=bleach.linkify(bleach.clean(
            markdown(self.body, output_format='html'),
            tags=allowed_tags, strip=True))).where(self._pk_expr()).execute()
```

上面的代码中使用了一个装饰器`require_instance`，用来确保更新`body_html`字段之前 相应的`Post`模型实例已经创建并存储在数据库中。

将Markdown转换成HTML的过程分三步完成。

1.  `markdown()`函数初步把Markdown文本转换成 HTML。

2.  把得到的结果和允许使用的 HTML 标签列表传给`clean()`函数，删除所有不在白名单中的标签。

3.  最后一步通过`bleach.linkify`把纯文本中的URL转换成适当的`<a>`链接<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>。

当插入或更新`body`字段时，先要保存实例，然后通过调用实例的`update_body_html`方法更新`body_html`字段。

接着修改在模板中使用文章HTML内容的模板：

```django
{# app/templates/_posts.html #}
...
<div class="post-body">
    {% if post.body_html %}
        {{ post.body_html | safe }}
    {% else %}
        {{ post.body }}
    {% endif %}
</div>
...
```

渲染HTML格式内容时使用`| safe`后缀，其目的是告诉Jinja2不要转义HTML元素。

**🔖 执行`git checkout 11f`签出程序的这个版本。** ⚠️ 此版本包含数据库迁移和安装依赖包。


## 文章固定链接

为每篇文章添加一个专页，使用唯一的 URL 引用。下面是支持固定链接功能的路由和视图函数：

```python
# app/main/views.py
@main.route('/post/<int:id>')
def post(id):
    post_query = Post.select()
    post = futils.get_object_or_404(post_query, (Post.id == id))
    return render_template('post.html', posts=[post])
```

博客文章的URL使用插入数据库时分配的唯一`id`字段构建。

⚠️ `post.html`模板接收一个列表作为参数，这个列表就是要渲染的文章。这里必须要传入列表，因为要在这个页面中复用`_posts.html`模板。

固定链接添加到通用模板`_posts.html`中，显示在文章下方：

```django
{# app/templates/_posts.html #}

...
<div class="post-body">
    {% if post.body_html %}
        {{ post.body_html | safe }}
    {% else %}
        {{ post.body }}
    {% endif %}
</div>
<div class="post-footer">
    <a class="label label-info" href="{{ url_for('.post', id=post.id) }}">Permalink</a>
</div>
...
```

渲染固定链接页面的`post.html`模板：

```django
{# app/templates/post.html #}
{% extends "base.html" %}

{% block title %}Flaskr - Post{% endblock %}

{% block page_content %}
    {% include '_posts.html' %}
{% endblock %}
```

**🔖 执行`git checkout 11g`签出程序的这个版本。**


## 编辑文章

文章编辑显示在单独的页面中，其上部会显示文章的当前版本，下面跟着一个Markdown编辑器，用于修改Markdown源，页面下部还会显示一个编辑后的文章预览。

下面是编辑文章的模板：

```django
{# app/templates/edit_post.html #}
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flaskr - Edit Post{% endblock %}

{% block page_content %}
    <div class="page-header">
        <h1>Edit Post</h1>
    </div>
    <div>
        {{ wtf.quick_form(form) }}
    </div>
{% endblock %}

{% block scripts %}
    {{ super() }}
    {{ pagedown.include_pagedown() }}
{% endblock %}
```

编辑文章的路由如下：

```python
# app/main/views.py
@main.route('/edit/<int:id>', methods=['GET', 'POST'])
@login_required
def edit(id):
    post_query = Post.select()
    post = futils.get_object_or_404(post_query, (Post.id == id))
    if current_user != post.author and \
       not current_user.can(Permission.ADMINISTER):
        abort(403)
    form = PostForm()
    if form.validate_on_submit():
        post.body = form.body.data
        post.save()
        post.update_body_html()
        flash('The post has been updated.')
        return redirect(url_for('.post', id=post.id))
    form.body.data = post.body
    return render_template('edit_post.html', form=form)
```

这个视图函数只允许博客文章的作者编辑文章，但管理员能编辑所有用户的文章。如果用户试图编辑其他用户的文章，视图函数会返回 **403** 错误。这里使用的`PostForm`表单类和首页中使用的是同一个。

另外，为了功能完整，可在每篇博客文章的下面、固定链接的旁边添加一个指向编辑页面的链接。

**🔖 执行`git checkout 11h`签出程序的这个版本。**

## 脚注

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> Markdown规范没有为自动生成链接提供官方支持，PageDown以扩展的形式实现了这个功能，在服务器上要调用 `linkify()` 函数。
