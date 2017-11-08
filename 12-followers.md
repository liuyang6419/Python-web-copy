# 关注用户

本章将实现关注功能，让用户“关注”其他用户，并在首页只显示所关注用户发布的博客文章列表。


## 高级自引用多对多关系

在数据库中表示用户之间的关注可使用自引用多对多关系，此外还要存储用户关注另一个用户的日期， 以便按照时间顺序列出所有关注者。下面的代码定义了用户关注使用的`Follow`模型：

```python
# app/models.py
import peewee as pw


class Follow(db.Model):
    follower = pw.ForeignKeyField(User, related_name='followed',
                                  on_delete='CASCADE')
    followed = pw.ForeignKeyField(User, related_name='followers',5
                                  on_delete='CASCADE')
    timestamp = pw.DateTimeField(default=datetime.utcnow)

    class Meta:
        db_table = 'follows'
        indexes = (
            (('follower', 'followed'), True),
        )
```

其中`follower`字段表示关注者，`followed`字段表示被关注者。这两个字段通过`User`模型 进行反向引用，将`Follow`模型中的多对多关系的左右两侧拆分成两个基本的一对多关系， 左侧表示的一对多关系把用户和`follows`表中的一组记录联系起来，用户是关注者。右侧表示的一对多关系把用户和`follows`表中的一组记录联系起来，用户是被关注者。例如， 如果某个用户关注了100个用户，调用`user.followed`后会返回一个可迭代查询对象 （`SelectQuery`） ，每一个迭代实例的`follower`和`followed`回引属性都指向相应的用户。

删除对象时，默认的层叠行为是把对象联接的所有相关对象的外键设为空值。但在关联表中，删除记录后正确的行为应该是把指向该记录的实体也删除， 设置参数 `on_delete='CASCADE'` 能有效销毁联接。

程序现在要处理两个一对多关系，以便实现多对多关系。在`User`模型中定义用于控制关系 的4个新方法：

```python
# app/models.py

class User(UserMixin, db.Model):
    # ...
    def follow(self, user):
        if not self.is_following(user):
            f = Follow(follower=self, followed=user)
            f.save()

    def unfollow(self, user):
        f = self.followed.where(Follow.followed == user.id).first()
        if f:
            f.delete_instance()

    def is_following(self, user):
        return self.followed.where(
            Follow.followed == user.id).first() is not None

    def is_followed_by(self, user):
        return self.followers.where(
            Follow.follower == user.id).first() is not None
```

`follow()`方法手动把`Follow`实例插入关联表，从而把关注者和被关注者联接起来， 并让程序设定自定义字段的值。`unfollow()`方法使用`followed`关系找到联接用户和被关注用户的`Follow`实例。若要销毁这两个用户之间的联接，只需删除这个`Follow`对象即可。`is_following()`方法和`is_followed_ by()`方法分别在左右两边的一对多关系中搜索指定用户，如果找到了就返回`True`。

为了开启Sqlite3的外键支持，需要在配置文件中添加Peewee的数据库连接参数：

```python
# config.py
class Config(object):
    # ...
    PEEWEE_CONNECTION_PARAMS = {
        'pragmas': [('foreign_keys', 'on')]
    }
```

另外，可以在代码仓库找到对于这个数据库关系的单元测试。

**🔖 执行`git checkout 12a`签出程序的这个版本。** ⚠️ 此版本包含数据库迁移。


## 显示关注者

用户查看一个尚未关注用户的资料页，页面中应显示一个“Follow”（关注）按钮， 如果查看已关注用户的资料页则显示“Unfollow”（取消关注）按钮。

页面中最好还能显示出关注者和被关注者的数量，再列出关注和被关注的用户列表， 并在相应的用户资料页中显示“Follows You”（关注了你）的提示。

下面是在用户资料页上部添加关注信息的模板：

```django
{# app/templates/user.html #}
{% if current_user.can(Permission.FOLLOW) and user != current_user %}
    {% if not current_user.is_following(user) %}
        <a href="{{ url_for('.follow', username=user.username) }}"
           class="btn btn-primary btn-sm">Follow</a>
    {% else %}
        <a href="{{ url_for('.unfollow', username=user.username) }}"
           class="btn btn-danger btn-sm">Unfollow</a>
    {% endif %}
{% endif %}

<a class="btn btn-info btn-sm" href="{{ url_for('.followers', username=user.username) }}">
    Followers <span class="badge">{{ user.followers.count() }}</span>
</a>
<a class="btn btn-warning btn-sm" href="{{ url_for('.followed_by', username=user.username) }}">
    Following <span class="badge">{{ user.followed.count() }}</span>
</a>
{% if current_user.is_authenticated and user != current_user and user.is_following(current_user) %}
    | <span class="label label-success">Follows you</span>
{% endif %}
```

模板用到了4个新端点，分别对应相应的路由：

1.  `/follow/<username>`

    ```python
    # app/main/views.py
    @main.route('/follow/<username>')
    @login_required
    @permission_required(Permission.FOLLOW)
    def follow(username):
        user = User.select().where(User.username == username).first()
        if user is None:
            flash('Invalid user.')
            return redirect(url_for('.index'))
        if current_user.is_following(user):
            flash('You are already following this user.')
            return redirect(url_for('.user', username=username))
        current_user.follow(user)
        flash('You are now following %s.' % username)
        return redirect(url_for('.user', username=username))
    ```

    这个视图函数先加载请求的用户，确保用户存在且当前登录用户还没有关注这个用户，然 后调用`User`模型中定义的辅助方法`follow()`，用以联接两个用户。

2.  `/unfollow/<username>`

    与上面`/follow/<username>`路由类似。

3.  `/followers/<username>`

    ```python
    # app/main/views.py
    @main.route('/followers/<username>')
    def followers(username):
        user = User.select().where(User.username == username).first()
        if user is None:
            flash('Invalid user.')
            return redirect(url_for('.index'))
        page = request.args.get('page', 1, type=int)
        pagination = Pagination(Follow.followers_of(user),
                                current_app.config['FLASKR_FOLLOWERS_PER_PAGE'],
                                page,
                                check_bounds=False)
        follows = [{'user': item.follower, 'timestamp': item.timestamp}
                   for item in pagination.items]
        return render_template('followers.html', user=user,
                               title='Followers of',
                               endpoint='.followers', pagination=pagination,
                               follows=follows)
    ```

    这个函数加载并验证请求的用户，然后使用分页显示该用户的 followers 关系。

4.  `/followed-by/<username>`

    与`/followers/<username>`路由类似。

其中`Follow`模型类方法`followers_of`和`followed_by`的实现如下：

```python
# app/models.py
class Follow(db.Model):
    # ...
    @classmethod
    def followers_of(cls, user):
        """Followers of user."""
        return (cls.select(cls, User)
                .join(User, on=cls.follower)
                .where(cls.followed == user))

    @classmethod
    def followed_by(cls, user):
        """Followed by user."""
        return (cls.select(cls, User)
                .join(User, on=cls.followed)
                .where(Follow.follower == user))
```

渲染关注者列表的`followers.html`模板能用来渲染关注的用户列表和被关注的用户列表。模板接收的参数包括用户对象、分页链接使用的端点、分页对象和查询结果列表。

**🔖 执行`git checkout 12b`签出程序的这个版本。**


## 查询所关注用户的文章

在程序首页显示用户所关注用户发布的所有文章。这里使用了数据库联结查询，为了提升效率，避免 [N+1查询问题](http://docs.peewee-orm.com/en/latest/peewee/querying.html#avoiding-n-1-queries)，同时查询`Post`和`User`模型，然后进行联结操作，最后进行过滤操作。下面是获取所关注用户的文章的代码示例：

```python
# app/models.py
class User(UserMixin, db.Model):
    # ...
    @property
    def followed_posts(self):
        return (Post.select(Post, self.__class__)
                .join(Follow, on=(Post.author == Follow.followed))
                .join(self.__class__, on=(Post.author == self.__class__.id))
                .where(Follow.follower == self))
```

`followed_posts()`方法定义为属性，调用时无需加`()`。

**🔖 执行`git checkout 12c`签出程序的这个版本。**


## 显示所关注用户文章

现在，用户可以选择在首页显示所有用户的博客文章还是只显示所关注用户的文章。

显示所有博客文章或只显示所关注用户的文章的代码如下：

```python
# app/main/views.py
@main.route('/', methods=['GET', 'POST'])
def index():
    # ...
    show_followed = False
    if current_user.is_authenticated:
        show_followed = bool(request.cookies.get('show_followed', ''))
    if show_followed:
        query = current_user.followed_posts
    else:
        query = Post.timeline()
    pagination = Pagination(query,
                            current_app.config['FLASKR_POSTS_PER_PAGE'],
                            check_bounds=False)
    posts = pagination.items
    return render_template('index.html', form=form, posts=posts,
                           show_followed=show_followed, pagination=pagination)
```

显示所关注用户文章的选项存储在cookie的`show_followed`字段中， 如果其值为非空字符串，则表示只显示所关注用户的文章。cookie以`request.cookies`字典的形式存储在请求对象中。这个cookie的值会转换成布尔值，根据得到的值设定本地变量`query`的值。`query`的值决定最终获取所有博客文章的查询，或是获取过滤后的博客文章查询。

`show_followed`cookie在两个新路由中设定：

```python
# app/main/views.py
@main.route('/all')
@login_required
def show_all():
    resp = make_response(redirect(url_for('.index')))
    resp.set_cookie('show_followed', '', max_age=30*24*60*60)
    return resp


@main.route('/followed')
@login_required
def show_followed():
    resp = make_response(redirect(url_for('.index')))
    resp.set_cookie('show_followed', '1', max_age=30*24*60*60)
    return resp
```

指向这两个路由的链接添加在首页模板中。点击这两个链接后会为`show_followed`cookie设定适当的值，然后重定向到首页。cookie只能在响应对象中设置，要使用`make_response()`方法创建响应对象。

`set_cookie()`函数的前两个参数分别是cookie名和值。可选的`max_age`参数设置 cookie的过期时间，单位为秒。如果不指定参数`max_age`，浏览器关闭后cookie就会过期。在本例中，过期时间为30天。

接下来需要对模板进行改动，在页面上部添加两个导航选项卡，分别调用`/all`和`/followed`路由，并在会话中设定正确的值。

```django
{# app/templates/index.html #}
...
<div class="post-tabs">
    <ul class="nav nav-tabs">
        <li{% if not show_followed %} class="active"{% endif %}>
            <a href="{{ url_for('.show_all') }}">All</a>
        </li>
        {% if current_user.is_authenticated %}
            <li{% if show_followed %} class="active"{% endif %}>
                <a href="{{ url_for('.show_followed') }}">Followers</a>
            </li>
        {% endif %}
    </ul>
    {% include '_posts.html' %}
</div>
...
```

**🔖 执行`git checkout 12d`签出程序的这个版本。**

最后，为了切换所关注用户文章列表时能显示自己的文章，可在构建用户时把用户设为自己的关注者:

```python
# app/models.py
class User(UserMixin, db.Model):
    # ...
    @staticmethod
    def add_self_follows():
        for user in User.select():
            if not user.is_following(user):
                user.follow(user)
    # ...
    def save(self, *args, **kwargs):
        super(self.__class__, self).save(*args, **kwargs)
        if not self.is_following(self):
            self.follow(self)
```

通过覆盖基类的`save()`方法，用户实例在保存时会检查自己是否是自己的关注者， 如果不是，将关注自己。

通过`add_self_follows`方法，可以更新数据库中现有的用户关注自身。

可通过在 flask shell中运行这个函数来更新数据库：

```python
(flaskr_env3) $ flask shell
>>> User.add_self_follows()
```

用户关注自己这一功能的实现有一些副作用。因为用户的自关注链接，用户资料页显示的关注者 和被关注者的数量都增加了1个。为了显示准确，这些数字要减去1，直接渲染`{{ user.followers.count() - 1 }}`和`{{ user.followed.count() - 1 }}`即可。还要通过在模板中使用条件语句调整关注用户和被关注用户的列表，不显示自己。

**🔖 执行`git checkout 12e`签出程序的这个版本。**
