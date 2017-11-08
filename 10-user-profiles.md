# 用户资料

为了显示用户在网站中的活动情况，或者便于分享，在本章我们将实现用户资料页面。


## 资料信息

下面是扩充了的`User`模型，增加了新的表示用户信息的字段：

```python
# app/models.py
from datetime import datetime
import peewee as pw


class User(UserMixin, db.Model):
    # ...
    name = pw.CharField(64, null=True)
    location = pw.CharField(64, null=True)
    about_me = pw.TextField(null=True)
    member_since = pw.DateTimeField(default=datetime.utcnow, null=True)
    last_seen = pw.DateTimeField(default=datetime.utcnow, null=True)
```

新添加的字段保存用户的真实姓名、所在地、自我介绍、注册日期和最后访问日期。

两个时间戳的默认值都是当前时间。`datetime.utcnow`后面没有`()`，因为`peewee.DateTimeField`的`default`参数可接受函数作为默认值，每次需要生成默认值时，其会调用指定的函数。

`last_seen`字段创建时的初始值也是当前时间，但用户每次访问网站后，这个值都会被刷新。可以在`User`模型中添加一个方法完成这个操作：

```python
# app/models.py
class User(UserMixin, db.Model):
    # ...

    def ping(self):
        self.last_seen = datetime.utcnow()
        self.save()
```

每次收到用户的请求时都需要调用`ping()`方法。使用蓝图中的`before_app_request`处理器 可以在每次请求前运行代码，更新已登录用户的访问时间。

```python
# app/auth/views.py
@auth.before_app_request
def before_request():
    if current_user.is_authenticated:
        current_user.ping()
```


## 用户资料页面

下面定义用户资料页面的路由：

```python
# app/main/views.py
import playhouse.flask_utils as futils

from ..models import User

@main.route('/user/<username>')
def user(username):
    user_query = User.select()
    user = futils.get_object_or_404(user_query, (User.username == username))
    return render_template('user.html', user=user)
```

这个视图函数会在数据库中搜索URL中指定的用户名，如果找到，则渲染模板`user.html`，并把用户名作为参数传入模板。如果传入路由的用户名不存在，则返回 **404** 错误。

用户资料页面的模板如下：

```django
{# app/templates/user.html #}
{% block page_content %}
    <div class="page-header">
        <h1>{{ user.username }}</h1>
        {% if user.name or user.location %}
            <p>
                {% if user.name %}{{ user.name }}{% endif %}
                {% if user.location %}
                    From {{ user.location }}
                {% endif %}
            </p>
        {% endif %}
        {% if current_user.is_administrator() %}
            <p><a href="mailto:{{ user.email }}">{{ user.email }}</a></p>
        {% endif %}
        {% if user.about_me %}<p>{{ user.about_me }}</p>{% endif %}
        <p>Member since {{ moment(user.member_since).format('L') }}.
            Last seen {{ moment(user.last_seen).fromNow() }}.</p>
    </div>
{% endblock %}
```

为了使用户能够访问自己的资料页面，可以在导航条中添加一个链接：

```django
{# app/tempaltes/base.html #}
{% if current_user.is_authenticated %}
    <li><a href="{{ url_for('main.user', username=current_user.username) }}">Profile</a></li>
{% endif %}
```

不应让未认证的用户看到资料页面的链接。

另外，还需要创建数据库迁移和进行单元测试，相应的代码可在代码仓库中找到。

**🔖 执行`git checkout 10a`签出程序的这个版本。**


## 编辑用户资料

用户资料的编辑分两种情况：

1.  用户编辑自己的资料；
2.  管理员应该能编辑任意用户的资料——包括编辑用户不能直接访问的`User`模型字段，例如用户角色。

为实现以上两种编辑需求要创建两个不同的表单。


### 用户级资料编辑

普通用户的资料编辑表单如下：

```python
class EditProfileForm(FlaskForm):
    name = StringField('Real name', validators=[Length(0, 64)])
    location = StringField('Location', validators=[Length(0, 64)])
    about_me = TextAreaField('About me')
    submit = SubmitField('Submit')
```

此表单所有字段都是可选的，因此长度验证函数允许长度为零。

显示这个表单的路由定义如下：

```python
# app/main/views.py
@main.route('/edit-profile', methods=['GET', 'POST'])
@login_required
def edit_profile():
    form = EditProfileForm()
    if form.validate_on_submit():
        current_user.name = form.name.data
        current_user.location = form.location.data
        current_user.about_me = form.about_me.data
        current_user.save()
        flash('Your profile has been updated.')
        return redirect(url_for('.user', username=current_user.username))
    form.name.data = current_user.name
    form.location.data = current_user.location
    form.about_me.data = current_user.about_me
    return render_template('edit_profile.html', form=form)
```

视图函数通过赋值给`form.<field-name>.data`把所有字段设定了初始值，提交表单后，表单字段的`data`属性中保存有更新后的值，可以将其赋值给用户 对象中的各字段，然后再保存更新过的用户对象。

接着添加资料编辑的链接：

```django
{# app/templates/user.html #}
{% if user == current_user %}
    <a class="btn btn-default" href="{{ url_for('.edit_profile') }}">Edit Profile</a>
{% endif %}
```

条件语句能确保只有当用户查看自己的资料页面时才显示这个链接。


### 管理员级资料编辑

相比普通用户的资料编辑表单，管理员在表单中还要能编辑用户的电子邮件、用户名、确认状态和角色。其表单示例如下：

```python
# app/main/forms.py
class EditProfileAdminForm(FlaskForm):
    email = StringField('Email', validators=[Required(), Length(1, 64),
                                             Email()])
    username = StringField('Username', validators=[
        Required(), Length(1, 64), Regexp('^[A-Za-z][A-Za-z0-9_.]*$', 0,
                                          'Usernames must have only letters, '
                                          'numbers, dots or underscores')])
    confirmed = BooleanField('Confirmed')
    role = SelectField('Role', coerce=int)
    name = StringField('Real name', validators=[Length(0, 64)])
    location = StringField('Location', validators=[Length(0, 64)])
    about_me = TextAreaField('About me')
    submit = SubmitField('Submit')

    def __init__(self, user, *args, **kwargs):
        super(EditProfileAdminForm, self).__init__(*args, **kwargs)
        self.role.choices = [(role.id, role.name)
                             for role in Role.select().order_by(Role.name)]
        self.user = user

    def validate_email(self, field):
        if (field.data != self.user.email and
                User.select().where(User.email == field.data).first()):
            raise ValidationError('Email already registered.')

    def validate_username(self, field):
        if (field.data != self.user.username and
                User.select().where(User.username == field.data).first()):
            raise ValidationError('Username already in use.')
```

WTForms使用`SelectField`包装HTML表单控件`<select>`，从而实现下拉列表，用来在这个表单中选择用户角色。`SelectField`实例必须在其`choices`属性中设置各选项。选项必须是一个由元组组成的列表，各元组都包含两个元素： 选项的标识符和显示在控件中的文本字符串。`choices`列表在表单的构造函数中设定，其值从`Role`模型中获取，使用一个查询按照角色名 的字母顺序排列所有角色。元组中的标识符是角色的`id`，因为其是个整数，所以 在`SelectField`构造函数中添加`coerce=int`参数，从而把字段的值转换为整数，而不使用默认的字符串。

验证`email`和`username`字段时，首先要检查字段的值是否发生了变化，如果有变化，就要保证新值不和其他用户的相应字段值重复；如果字段值没有变化，则应该跳过验证。表单构造函数接收用户对象作为参数，并将其保存在成员变量中，随后自定义的验证方法要使用这个用户对象。

管理员的资料编辑器路由定义如下：

```python
# app/main/views.py
@main.route('/edit-profile/<int:id>', methods=['GET', 'POST'])
@login_required
@admin_required
def edit_profile_admin(id):
    user_query = User.select()
    user = futils.get_object_or_404(user_query, (User.id == id))
    form = EditProfileAdminForm(user=user)
    if form.validate_on_submit():
        user.email = form.email.data
        user.username = form.username.data
        user.confirmed = form.confirmed.data
        user.role = Role.select.where(Role.id == form.role.data).first()
        user.name = form.name.data
        user.location = form.location.data
        user.about_me = form.about_me.data
        user.save()
        flash('The profile has been updated.')
        return redirect(url_for('.user', username=user.username))
    form.email.data = user.email
    form.username.data = user.username
    form.confirmed.data = user.confirmed
    form.role.data = user.role_id
    form.name.data = user.name
    form.location.data = user.location
    form.about_me.data = user.about_me
    return render_template('edit_profile.html', form=form, user=user)
```

⚠️代码中对于选择用户角色的`SelectField`的处理，用户的角色根据表单字段的`data`属性 提取`id`，并通过其值加在角色对象。

接着添加管理员使用的资料编辑链接：

```django
{# app/templates/user.html #}
{% if current_user.is_administrator() %}
    <a class="btn btn-danger" href="{{ url_for('.edit_profile_admin', id=user.id) }}">
        Edit Profile [Admin]
    </a>
{% endif %}
```

条件语句确保只当登录用户为管理员时才显示按钮。

**🔖 执行`git checkout 10b`签出程序的这个版本。**


## 用户头像

这一节我们着手考虑为用户添加头像，以改善页面的外观。可以考虑以下几种解决方案：

1.  使用第三方头像服务，比如[Gravatar](https://gravatar.com)
2.  允许用户上传头像图片
3.  自动为程序中注册的用户生成唯一性头像

我们在当前程序中加入结合第1种和第3种解决方案的实现。

首先来了解一下 **Gravatar** 头像服务。

Gravatar 是一个行业领先的头像服务，能把头像和电子邮件地址关联起来。用户先要到<https://gravatar.com>中注册账户，然后上传图片。生成头像的URL时，要计算电子邮件地址的 **MD5** 散列值：

```python
(flaskr_env3) $ python
>>> import hashlib
>>> hashlib.md5('john@example.com'.encode('utf-8')).hexdigest()
'd4c74594d841139328695756648b6bd6'
```

生成的头像URL是在`http://www.gravatar.com/avatar/`或`https://secure.gravatar.com/avatar/`之后加上这个 MD5 散列值。例如，你在浏览器的地址栏中输入`http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6`，就会看到电子邮件地址`john@example.com`对应的头像图片。

如果这个电子邮件地址没有对应的头像，则会显示一个默认图片（或根据设置返回HTTP **404** 错误）。头像URL的查询字符串中可以包含多个参数以配置头像图片的特征。

可设参数如下表所示：

| 参数名           | 说明                                                                         |
|:-----------------|:-----------------------------------------------------------------------------|
| s (size)         | 图片大小，单位为像素                                                         |
| r (rating)       | 图片级别。可选值有“g”、“pg”、“r”和“x”                                        |
| d (default)      | 没有注册Gravatar服务的用户使用的默认图片生成方式。可选值有：“404”，返回404错误；默认图片的URL；图片生成器 “mm”、“identicon”、“monsterid”、“wavatar”、“retro” 或 “blank” 之一  |
| f (forcedefault) | 强制使用默认头像                                                             |

更详细的介绍请参考[Gravatar官方文档](https://gravatar.com/site/implement/images/)。

接着来了解如何自动为用户生成唯一性头像。

如果用户的电子邮件地址没有对应的Gravatar头像（用户没使用Gravatar头像服务），或者在 用户网络条件不佳无法访问Gravatar的情况，此时一种方案是自动为用户生成图像。

这里直接使用了一个开源的库 **Identicon** ，其会根据散列值生成具有Github头像 风格的 **SVG** 格式的图片。

我们将这部分的代码添加到`User`模型中，如下所示：

```python
# app/models.py
import hashlib
from flask import request

import requests

from utils.identicon import IdenticonSVG


class User(UserMixin, db.Model):
    # ...
    def gravatar(self, size=100, default='404', rating='g'):
        if request.is_secure:
            url = 'https://secure.gravatar.com/avatar'
        else:
            url = 'http://www.gravatar.com/avatar'

        hash = hashlib.md5(self.email.lower().encode('utf-8')).hexdigest()
        gravatar_url = '{url}/{hash}?s={size}&d={default}&r={rating}'.format(
            url=url, hash=hash, size=size, default=default, rating=rating)
        return gravatar_url

    def avatar(self, size=100, **kwargs):
        gravatar_url = self.gravatar(size)
        r = requests.get(gravatar_url)
        if r.status_code == 404:
            hash = gravatar_url.split('/')[-1].split('?')[0]
            i = IdenticonSVG(hash, size=size, **kwargs)
            gravatar_url = 'data:image/svg+xml;text,{0}'.format(i.to_string(True))

        return gravatar_url
```

上面代码实现了两个实例方法`gravatar`和`avatar`，其中`gravatar`方法会根据URL基、 用户电子邮件地址的MD5散列值及相关参数来构建Gravatar请求URL，其会选择标准的或加密 的Gravatar URL基以匹配用户的安全需求。`avatar`方法会首先通过调用`gravatar`方法 来获得Gravatar请求URL，然后尝试使用`requests`库去获取用户头像，当返回 **404** 错误时 便使用 Identicon 库提供的`IdenticonSVG`类来生成 **SVG** 格式的头像图像并将其转换成 数据类型的URL以便直接嵌入到网页中<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>。

现在就可以在flask shell中生成头像的URL了：

```python
(flaskr_env3) $ flask shell
>>> ctx = app.test_request_context('/')
>>> ctx.push()
>>> u = User(email='john@example.com')
>>> u.gravatar()
'http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6?s=100&d=404&r=g'
>>> u.gravatar(size=256)
'http://www.gravatar.com/avatar/d4c74594d841139328695756648b6bd6?s=256&d=404&r=g'
>>> u.avatar()
"data:image/svg+xml;text,<svg xmlns='http://www.w3.org/2000/svg' width='100' ..."
>>> ctx.pop()
```

因为`gravatar`方法要访问`request`请求对象，上面代码执行过程需要先激活请求上下文，否则会导致运行时错误，具体请参考[Diving into Context Locals](http://flask.pocoo.org/docs/0.12/reqcontext/#diving-into-context-locals)来了解细节。

接着在用户资料页面中添加头像：

```django
{# app/templates/user.html #}
...
<img class="img-rounded profile-thumbnail" src="{{ user.avatar(size=256) }}">
...
```

使用类似方式，可在基模板的导航条上添加一个已登录用户头像的小型缩略图。为了更好地调整页面中头像图片的显示格式，可使用一些自定义的 CSS 类。可以在代码仓库的`styles.css`文件中查看自定义的 CSS，`styles.css`文件保存在程序静态文件的文件夹中，而且要在`base.html`模板中引用。

**🔖 执行`git checkout 10c`签出程序的这个版本。** ⚠️ 签出的代码中`app/models.py`文件`avatar`方法存在BUG，请使用上面展示的代码进行临时修复。


### 缓存电子邮件地址散列值

为避免生成头像时每次都要生成MD5散列，减小计算量，可以将用户电子邮件地址的MD5散列缓存 在`User`模型中。

下面是使用缓存的MD5散列值生成头像的代码：

```python
# app/models.py
from requests.exceptions import ConnectionError, HTTPError


class User(UserMixin, db.Model):
    # ...
    avatar_hash = pw.CharField(32, null=True)

    def __init__(self, **kwargs):
        if self.email is not None and self.avatar_hash is None:
            self.avatar_hash = hashlib.md5(
                self.email.lower().encode('utf-8')).hexdigest()

    def change_email(self, token):
        # ...
        self.email = new_email
        self.avatar_hash = hashlib.md5(
            self.email.lower().encode('utf-8')).hexdigest()

    def gravatar(self, size=100, default='404', rating='g'):
        if request.is_secure:
            url = 'https://secure.gravatar.com/avatar'
        else:
            url = 'http://www.gravatar.com/avatar'
        if not self.avatar_hash:
            self.avatar_hash = hashlib.md5(
                self.email.lower().encode('utf-8')).hexdigest()
            self.save()
        hash = self.avatar_hash
        gravatar_url = '{url}/{hash}?s={size}&d={default}&r={rating}'.format(
            url=url, hash=hash, size=size, default=default, rating=rating)
        return gravatar_url

    def avatar(self, size=100, **kwargs):
        gravatar_url = self.gravatar(size)
        try:
            r = requests.get(gravatar_url)
            r.raise_for_status()
        except (ConnectionError, HTTPError):
            i = IdenticonSVG(self.avatar_hash, size=size, **kwargs)
            gravatar_url = 'data:image/svg+xml;text,{0}'.format(i.to_string(True))

        return gravatar_url
```

模型初始化过程中会计算电子邮件的散列值，然后存入数据库，若用户更新了电子邮件地址，则会重新计算散列值。`gravatar()`方法会使用模型中保存的散列值； 如果模型中没有，就和之前一样计算电子邮件地址的散列值。另外，代码中还增加了网络错误处理。

**🔖 执行`git checkout 10d`签出程序的这个版本。**

## 脚注

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> 可参考[Data URLs](https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs)来了解细节。
