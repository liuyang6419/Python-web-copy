# 用户认证

从本章开始，我们会学习如何实现一个社交博客程序实例。

程序需要进行用户跟踪，在用户连接程序时进行身份认证，以便知道用户是谁，从而提供有针对性的体验。

最常用的认证方法要求用户提供一个身份证明（用户的电子邮件或用户名）和一个密码。


## Flask认证扩展

本章使用的包列表如下：

-   **[Flask-Login](https://flask-login.readthedocs.io):** 管理已登录用户的用户会话
-   **[Werkzeug](https://github.com/pallets/werkzeug):** 计算密码散列值并进行核对
-   **[itsdangerous](https://github.com/pallets/itsdangerous):** 生成确认令牌

除了认证相关的包之外，还用到下列常规用途的扩展：

-   **Flask-Bootstrap:** HTML模板
-   **Flask-WTF:** Web表单


## 密码安全性

为了保证数据库中用户密码的安全，关键在于不能存储密码本身，而要存储密码的散列值。计算密码散列值的函数接收密码作为输入，使用一种或多种加密算法转换密码，最终 得到一个和原始密码没有关系的字符序列。核对密码时，密码散列值可代替原始密码，因为计算散列值的函数是可复现的：只要输入一样，结果就一样。

⚠️ 参考[“Salted Password Hashing - Doing it Right”](https://crackstation.net/hashing-security.htm)（计算加盐密码散列值的正确方法）这篇文章来了解如何生成安全的密码散列值。


### 使用Werkzeug实现密码散列

Werkzeug 中的`security`模块能够很方便地实现密码散列值的计算。

这一功能的实现只需要两个函数，分别用在注册用户和验证用户阶段。

-   `generate_password_hash(password, method='pbkdf2:sha1', salt_length=8)`

    这个函数将原始密码作为输入，以字符串形式输出密码的散列值，输出的值可保存在用户数据库中。`method`和`salt_length`的默认值就能满足大多数需求。生成的散列值格式为：`method$salt$hash`。

-   `check_password_hash(pwhash, password)`

    这个函数的参数是从数据库中取回的密码散列值和用户输入的密码。返回值为`True`表明密码正确。

下面的代码展示在`User`模型中加入密码散列：

```python
# app/models.py
from werkzeug.security import generate_password_hash, check_password_hash

class User(db.Model):
    # ...
    password_hash = pw.CharField(128)

    @property
    def password(self):
        raise AttributeError('password is not a readable attribute')

    @password.setter
    def password(self, password):
        self.password_hash = generate_password_hash(password)

    def verify_password(self, password):
        return check_password_hash(self.password_hash, password)
```

计算密码散列值的函数通过名为`password`的 **只写属性** 实现。设定这个属性的值时，赋值方法会调用`generate_password_hash()`函数，并把得到的结果赋给`password_hash`字段。如果试图读取`password`属性的值，则会触发异常，因为生成散列值后就无法还原成原来的密码了。

`verify_password`方法接受一个参数（即密码），将其传给`check_ password_hash()`函数，和存储在`User`模型中的密码散列值进行比对。如果这个方法返回`True`，就表明密码是正确的。

密码散列功能已经完成，为了方便在shell中测试时直接使用数据模型，我们可以将数据模型加入到 shell上下文中：

```python
@app.shell_context_processor
def make_shell_context():
    pw_models = {mod.__name__: mod for mod in db.models}
    return dict(db=db, **pw_models)
```

上面的代码使用了`shell_context_processor`装饰器，用来注册一个上下文处理器函数，注册的函数将在进入flask shell时通过调用`app.make_shell_context()`被执行，它和下面的代码是等价的：

```python
# Equivalent to the following
app.shell_context_processors.append(make_shell_context)
```

下面是在flask shell中测试密码散列功能：

```
(flaskr_env3) $ flask shell
...
>>> u = User()
>>> u.password = 'cat'
>>> u.password_hash
'pbkdf2:sha256:50000$XCaFpTV4$4a26974acd62d01d758f4e1c553abd1a2092441e97d8a2f914429622ff0a202c'
>>> u.verify_password('cat')
True
>>> u.verify_password('dog')
False
>>> u2 = User()
>>> u2.password = 'cat'
>>> u2.password_hash
'pbkdf2:sha256:50000$TPaM3QjQ$b0e9628bc1165ff48e235cec411ed49d31bbed272dd7f094cc7d8cef8cb696fe'
```

⚠️ 即使用户`u`和`u2`使用了相同的密码，它们的密码散列值也完全不一样。

**🔖 执行`git checkout 8a`签出程序的这个版本。**


### 密码散列化测试

为了确保这个功能今后可持续使用，我们可以把上述测试写成单元测试，以便于重复执行。

在`tests`包中新建一个模块，编写新的测试：

```python
# tests/test_user_model.py
import unittest

from app.models import User


class UserModelTestCase(unittest.TestCase):
    def test_password_setter(self):
        u = User(password='cat')
        self.assertTrue(u.password_hash is not None)

    def test_no_password_getter(self):
        u = User(password='cat')
        with self.assertRaises(AttributeError):
            u.password

    def test_password_verification(self):
        u = User(password='cat')
        self.assertTrue(u.verify_password('cat'))
        self.assertFalse(u.verify_password('dog'))

    def test_password_salts_are_random(self):
        u = User(password='cat')
        u2 = User(password='cat')
        self.assertTrue(u.password_hash != u2.password_hash)
```

下面是测试的结果（执行`flask test`运行）：

    ...
    test_no_password_getter (test_user_model.UserModelTestCase) ... ok
    test_password_salts_are_random (test_user_model.UserModelTestCase) ... ok
    test_password_setter (test_user_model.UserModelTestCase) ... ok
    test_password_verification (test_user_model.UserModelTestCase) ... ok
    ...


## 认证蓝图

我们将与用户认证系统相关的路由在`auth`蓝图中定义，其保存在同名Python包中，包括构造文件创建蓝图对象，再从`views.py`模块中引入路由。

创建蓝图：

```python
# app/auth/__init__.py
from flask import Blueprint

auth = Blueprint('auth', __name__)

from . import views
```

蓝图中的路由和视图函数：

```python
# app/auth/views.py
from flask import render_template

from . import auth


@auth.route('/login')
def login():
    return render_template('auth/login.html')
```

⚠️ 为`render_template()`指定的模板文件保存在`auth`文件夹中。这个文件夹必须在`app/templates`中创建，因为Flask认为模板的路径是相对于 程序模板文件夹而言的。为避免与`main`蓝图和后续添加的蓝图发生模板命名冲突，可以把蓝图使用的模板保存在单独的文件夹中。

💡 也可将蓝图配置成使用其独立的文件夹保存模板。`render_template()`函数 会首先搜索程序配置的模板文件夹，然后再搜索蓝图配置的模板文件夹。

`auth`蓝图要在`create_app()`工厂函数中附加到程序上：

```python
# app/__init__.py
def create_app(config_name):
    # ...
    from .auth import auth as auth_blueprint
    app.register_blueprint(auth_blueprint, url_prefix='/auth')

    return app
```

`url_prefix`是可选参数，上面视图函数绑定的`/login`路由会注册成`/auth/login`，在开发Web服务器中，完整的URL就变成了`http://localhost:5000/auth/login`。


## 使用Flask-Login认证用户

用户登录程序后，其认证状态要被记录下来。 **Flask-Login** 专门用来管理用户认证系统中的 认证状态，且不依赖特定的认证机制。

在虚拟环境中添加这个扩展的方式如下：<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>

1.  编辑`requirements.in`文件，在其中添加`flask-login`；
2.  执行`pip-compile requirements.in`得到编译更新后的`requirements.txt`；
3.  执行`pip-sync requirements.txt`同步依赖包。


### 用于登录的用户模型

使用 Flask-Login 扩展，`User`模型必须实现下面几个方法：

| 方法                 | 说明                                                     |
|----------------------|----------------------------------------------------------|
| `is_authenticated()` | 如果用户已经登录，必须返回 `True` ，否则返回 `False`     |
| `is_active()`        | 如果允许用户登录，必须返回 `True` ，否则返回 `False` 。如果要禁用账户，可以返回 `False` |
| `is_anonymous()`     | 对普通用户必须返回 `False`                               |
| `get_id()`           | 必须返回用户的唯一标识符，使用Unicode编码字符串          |

Flask-Login 在`UserMixin`类中提供了上述方法的默认实现，使`User`模型类直接继承即可。下面是修改后的`User`模型：

```python
# app/models.py
from flask_login import UserMixin

class User(UserMixin, db.Model):
    email = pw.CharField(64, unique=True, index=True)
```

另外，添加了`email`字段，用户使用电子邮件地址登录。

接着在程序的工厂函数中初始化 Flask-Login ：

```python
# app/__init__.py
from flask_login import LoginManager

login_manager = LoginManager()
login_manager.session_protection = 'strong'
login_manager.login_view = 'auth.login'

def create_app(config_name):
    # ...
    login_manager.init_app(app)
    # ...
```

`LoginManager`对象的`session_protection`属性可设为`None`、`'basic'`或`'strong'`，以提供不同的安全等级防止用户会话遭篡改。设为`'strong'`时，Flask-Login 会记录客户端 IP 地址和浏览器的用户代理信息，如果发现异动就登出用户。`login_view`属性设置登录页面的端点。

最后，Flask-Login 要求程序实现一个回调函数，使用指定的标识符加载用户：

```python
# app/models.py
from . import login_manager

@login_manager.user_loader
def load_user(user_id):
    return User.select().where(User.id == int(user_id)).first()
```

加载用户的回调函数接收以Unicode字符串形式表示的用户标识符。如果能找到用户，这个函数必须返回用户对象；否则应该返回`None`。


### 保护路由

Flask-Login 提供了一个`login_required`装饰器，用来保护路由只让认证用户访问，示例如下：

```python
from flask_login import login_required


@app.route('/secret')
@login_required
def secret():
    return 'Only authenticated users are allowed!'
```

如果未认证的用户访问这个路由，Flask-Login 会拦截请求，把用户发往登录页面。


### 添加登录表单

呈现给用户的登录表单中包含一个用于输入电子邮件地址的文本字段、 一个密码字段、一个“记住我”复选框和提交按钮。此表单使用 Flask-WTF 实现：

```python
# app/auth/forms.py
from flask_wtf import FlaskForm

from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import Required, Length, Email


class LoginForm(FlaskForm):
    email = StringField('Email', validators=[Required(), Length(1, 64),
                                             Email()])
    password = PasswordField('Password', validators=[Required()])
    remember_me = BooleanField('Keep me logged in')
    submit = SubmitField('Log In')
```

电子邮件字段用到了 WTForms 提供的`Length()`和`Email()`验证函数。`PasswordField`类表示属性为 `type="password"` 的`<input>`元素。`BooleanField`类表示复选框。

登录页面使用的模板保存在`auth/login.html`文件中。这个模板只需使用 Flask-Bootstrap 提供的`wtf.quick_form()`宏渲染表单即可。

`base.html`模板中的导航条使用 Jinja2 条件语句，并根据当前用户的登录状态分别显示 “Sign In”或“Sign Out”链接，代码如下：

```django
{# app/templates/base.html #}
<ul class="nav navbar-nav navbar-right">
    {% if current_user.is_authenticated %}
        <li><a href="{{ url_for('auth.logout') }}">Log Out</a></li>
    {% else %}
        <li><a href="{{ url_for('auth.login') }}">Log In</a></li>
    {% endif %}
</ul>
```

判断条件中的变量`current_user`由 Flask-Login 定义，且在视图函数和模板中自动可用。这个变量的值是当前登录的用户，如果用户尚未登录，则是一个匿名用户代理对象。如果是匿名用户，`is_authenticated()`方法返回`False`。所以此方法可用来判断当前用户是否已经登录。


### 登入登出用户

视图函数`login()`和`logout()`的实现：

```python
# app/auth/views.py
from flask import render_template, redirect, request, url_for, flash

from flask_login import login_user, logout_user, login_required

from . import auth
from ..models import User
from .forms import LoginForm


@auth.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        user = User.select().where(User.email == form.email.data).first()
        if user is not None and user.verify_password(form.password.data):
            login_user(user, form.remember_me.data)
            return redirect(request.args.get('next') or url_for('main.index'))
        flash('Invalid username or password.')
    return render_template('auth/login.html', form=form)


@auth.route('/logout')
@login_required
def logout():
    logout_user()
    flash('You have been logged out.')
    return redirect(url_for('main.index'))
```

`login()`视图函数创建了一个`LoginForm`对象。

当请求类型是 GET 时，视图函数直接渲染模板，即显示表单。当表单在 POST 请求中提交时，`validate_on_submit()`函数会验证表单数据，然后尝试登入用户。

`login_user()`函数的参数是要登录的用户，以及可选的“记住我”布尔值，如果值为`False`，那么关闭浏览器后用户会话就过期了，所以下次用户访问时要重新登录。如果值为`True`，那么会在用户浏览器中写入一个长期有效的cookie，使用这个cookie 可以 复现用户会话。

提交登录密令的POST请求最后也做了重定向，不过目标URL有两种可能。用户访问未授权的URL时会显示登录表单，Flask-Login会把原地址保存在查询字符串的`next`参数中，这个参数可从`request.args`字典中读取。如果查询字符串中没有`next`参数，则重定向到首页。如果用户输入的电子邮件或密码不正确，程序会设定一个Flash消息，再次渲染表单，让用户重试登录。

⚠️ 在生产服务器上，登录路由必须使用安全的 **HTTPS** ，从而加密传送给服务器的表单数据。

`logout`视图函数中的`logout_user()`函数将删除并重设用户会话。随后会显示一个Flash消息，确认这次操作，再重定向到首页，这样登出就完成了。

接着来更新登录模板以渲染表单：

```django
{# app/templates/auth/login.html #}
{% extends "base.html" %}
{% import "bootstrap/wtf.html" as wtf %}

{% block title %}Flaskr - Login{% endblock %}

{% block page_content %}
    <div class="page-header">
        <h1>Login</h1>
    </div>
    <div class="col-md-4">
        {{ wtf.quick_form(form) }}
    </div>
{% endblock %}
```


### 测试登录

为验证登录功能可用，可更新首页，使用已登录用户的名字显示一个欢迎消息。

```django
Hello,
{% if current_user.is_authenticated %}
    {{ current_user.username }}
{% else %}
    Stranger
{% endif %}!
```

因为还未创建用户注册功能，所以新用户可在 flask shell 中注 册<sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup>：

    (flaskr_env3) $ flask shell
    >>> u = User(email='john@example.com', username='john', password='cat')
    >>> u.save()

运行程序，尝试使用刚刚创建的用户进行登录。

**🔖 执行`git checkout 8b`签出程序的这个版本。**


## 注册新用户

现在来实现程序的用户注册功能。


### 添加用户注册表单

注册页面使用的表单要求用户输入电子邮件地址、用户名和密码。代码如下：

```python
# app/auth/forms.py
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, BooleanField, SubmitField
from wtforms.validators import Required, Length, Email, Regexp, EqualTo
from wtforms import ValidationError

from ..models import User


class RegistrationForm(FlaskForm):
    email = StringField('Email',
                        validators=[Required(), Length(1, 64),
                                    Email()])
    username = StringField('Username',
                           validators=[Required(),
                                       Length(1, 64),
                                       Regexp('^[A-Za-z][A-Za-z0-9_.]*$', 0,
                                              'Usernames must have only letters, '
                                              'numbers, dots and underscores')])
    password = PasswordField(
        'Password',
        validators=[Required(),
                    EqualTo('password2', message='Passwords must match.')])
    password2 = PasswordField('Confirm password', validators=[Required()])
    submit = SubmitField('Register')

    def validate_email(self, field):
        if User.select().where(User.email == field.data).first():
            raise ValidationError('Email already registered.')

    def validate_username(self, field):
        if User.select().where(User.username == field.data).first():
            raise ValidationError('Username already in use.')
```

这个表单使用 WTForms 提供的`Regexp`验证函数，确保`username`字段只包含 字母、数字、下划线和点号。其中正则表达式后面的两个参数分别是 **正则表达式的旗标（flag）** 和 **验证失败时显示的错误消息** 。

安全起见，密码要输入两次。使用`EqualTo`验证两个密码字段中的值是否一致。

这个表单还有两个自定义的验证方法，以`validate_`开头且后面跟着字段名的方法，通过这种方式分别为`email`和`username`字段定义了验证函数，确保填写的值在数据库中没出现过。自定义的验证函数要想表示验证失败，可以抛出`ValidationError`异常，其参数就是错误消息。

显示这个表单的模板是`/templates/auth/register.html`。和登录模板一样，这个模板也使用`wtf.quick_form()`渲染表单。

另外，登录页面要显示一个指向注册页面的链接，让没有账户的用户能轻易找到注册页面。改动如下：

```django
{# app/templates/auth/login.html #}
<br>
<p>
    New user?
    <a href="{{ url_for('auth.register') }}">
        Click here to register
    </a>.
</p>
```


### 注册新用户

提交注册表单，通过验证后，系统就使用用户填写的信息在数据库中添加一个新用户。其视图函数的实现如下：

```python
# app/auth/views.py
@auth.route('/register', methods=['GET', 'POST'])
def register():
    form = RegistrationForm()
    if form.validate_on_submit():
        user = User(email=form.email.data,
                    username=form.username.data,
                    password=form.password.data)
        user.save()
        flash('You can now login.')
        return redirect(url_for('auth.login'))
    return render_template('auth/register.html', form=form)
```

**🔖 执行`git checkout 8c`签出程序的这个版本。**


## 确认账户

为验证用户电子邮件地址，用户注册后，程序应发送一封确认邮件。新账户先被标记成 **待确认状态** ，用户按照邮件中的说明操作后 （往往会要求用户点击一个包含确认令牌的特殊URL链接），才能证明自己可以被联系上。


### 生成确认令牌

确认邮件中的确认链接是`http://www.example.com/auth/confirm/<token>`这种形式 的URL，其中`token`是使用 **itsdangerous** 包生成的包含用户id的安全令牌。用户点击链接后，处理这个路由的视图函数就将收到的安全令牌作为参数进行确认，然后将用户状态更新为已确认。

下面的 flask shell 会话展示如何生成安全令牌：

```python
(flaskr_env3) $ flask shell
>>> from itsdangerous import TimedJSONWebSignatureSerializer as Serializer
>>> s = Serializer(app.config['SECRET_KEY'], expires_in=3600)
>>> token = s.dumps({'confirm': 23})
>>> token
b'eyJhbGciOiJIUzI1NiIsImlhdCI6MTUwNDE2ODI3NiwiZXhwIjoxNTA0MTcxODc2fQ.eyJjb25maXJtIjoyM30.8bCdSHNQmHKrpeeYRkrrLmP5EJUsIj9Z8IepSBzvbuk'
>>> data = s.loads(token)
>>> data
{'confirm': 23}
```

`TimedJSONWebSignatureSerializer`类生成具有过期时间的JSON Web签名（JSON Web Signatures，JWS）。这个类的构造函数接收的参数是一个密钥，在 Flask 程序中可使用`SECRET_KEY`设置。`expires_in`参数设置令牌的过期时间，单位为 **秒** 。

`dumps()`方法为指定的数据生成一个加密签名，然后再对数据和签名进行序列化，生成令牌字符串。

序列化对象提供了`loads()`方法来解码令牌，其唯一的参数是令牌字符串。这个方法会检验 **签名** 和 **过期时间** ，如果通过，返回原始数据。如果提供给`loads()`方法的令牌不正确或过期了，则抛出异常。

下面将生成和检验令牌的功能添加到`User`模型中：

```python
# app/models.py
from flask import current_app

from itsdangerous import TimedJSONWebSignatureSerializer as Serializer


class User(UserMixin, db.Model):
    # ...
    confirmed = pw.BooleanField(default=False)

    def generate_confirmation_token(self, expiration=3600*2):
        s = Serializer(current_app.config['SECRET_KEY'], expiration)
        return s.dumps({'confirm': self.id})

    def confirm(self, token):
        s = Serializer(current_app.config['SECRET_KEY'])
        try:
            data = s.loads(token)
        except:
            return False
        if data.get('confirm') != self.id:
            return False
        self.confirmed = True
        self.save()
        return True
```

`generate_confirmation_token()`方法生成一个令牌，有效期默认为两小时。`confirm()`方法检验令牌，如果检验通过，则把新添加的`confirmed`属性设为`True`。

除了检验令牌，`confirm()`方法还需检查令牌中的`id`是否和存储在`current_user`中的已登录用户匹配。这样，即使恶意用户知道如何生成签名令牌，也无法确认别人的账户。


### 使用 Flask-PW 进行数据库迁移

`User`模型中新加入了一个列用来保存账户的确认状态，因此要生成并执行一个新数据库迁移。

步骤如下：

1. 使用`flask db create user_add_confirmd_field`创建数据库迁移；
2. 在创建的`migrations/001_user_add_confirmd_field.py`中手动添加增加和删除字段的代码：

   ```
   from app.models import User

   def migrate(migrator, database, fake=False, **kwargs):
       migrator.add_fields(User, confirmed=User.confirmed)

   def rollback(migrator, database, fake=False, **kwargs):
       migrator.remove_fields(User, 'confirmed')
   ```

3. 使用`flask db migrate 001_user_add_confirmd_field`完成数据库迁移。

**💡 使用`flask db rollback <NAME>`来回滚数据库迁移。**

**💡 从代码仓库中找到新添加的两个方法的单元测试。**


### 发送确认邮件

当前的`/register`路由把新用户添加到数据库中后，会重定向到`/index`。在重定向之前，这个路由需要发送确认邮件。

能发送确认邮件的注册路由：

```python
# app/auth/views.py
# from ..email import send_email

@auth.route('/register', methods=['GET', 'POST'])
def register():
    form = RegistrationForm()
    if form.validate_on_submit():
        user = User(email=form.email.data,
                    username=form.username.data,
                    password=form.password.data)
        user.save()
        # token = user.generate_confirmation_token()
        # send_email(user.email, 'Confirm Your Account',
        #            'auth/email/confirm', user=user, token=token)
        # flash('A confirmation email has been sent to you by email.')
        flash('You can now login.')
        return redirect(url_for('auth.login'))
    return render_template('auth/register.html', form=form)
```

**⚠️ 发送确认邮件的功能并未实现，只为展示用。**

确认账户的视图函数如下所示：

```python
# @auth.route('/confirm/<token>')
# @login_required
# def confirm(token):
#     if current_user.confirmed:
#         return redirect(url_for('main.index'))
#     if current_user.confirm(token):
#         flash('You have confirmed your account. Thanks!')
#     else:
#         flash('The confirmation link is invalid or has expired.')
#     return redirect(url_for('main.index'))
```

`login_required`装饰器会保护这个路由，用户点击确认邮件中的链接后，要先登录，然后才能执行这个视图函数。

先检查已登录的用户是否已经确认过，如果确认过，则重定向到首页，避免用户不小心多次点击确认令牌带来的额外工作。

程序可以限制用户确认账户之前可以做哪些操作，这个功能可使用 Flask 提供 的`before_request`钩子完成。

**⚠️`before_request`钩子只能应用到属于蓝图的请求上。若想 在蓝图中使用针对程序全局请求的钩子，必须使用`before_app_request`装饰器。**

在`before_app_request`处理程序中过滤未确认的账户：

```python
# app/auth/views.py
# @auth.before_app_request
# def before_request():
#     if (current_user.is_authenticated() and
#         not current_user.confirmed and
#         request.endpoint[:5] != 'auth.' and
#             request.endpoint != 'static'):
#         return redirect(url_for('auth.unconfirmed'))


# @auth.route('/unconfirmed')
# def unconfirmed():
#     if current_user.is_anonymous() or current_user.confirmed:
#         return redirect(url_for('main.index'))
#     return render_template('auth/unconfirmed.html')
```

同时满足以下3个条件时，`before_app_request`处理程序会拦截请求：

1.  用户已登录（`current_user.is_authenticated()`必须返回`True`）
2.  用户的账户还未确认
3.  请求的端点（使用`request.endpoint`获取）不在认证蓝图中

    访问认证路由要获取权限，因为这些路由的作用是让用户确认账户或执行其他账户管理操作。

如果请求满足以上3个条件，则会被重定向到`/auth/unconfirmed`路由，显示一个确认账户相关信息的页面。

未确认用户的页面有如何确认账户的说明，此外还提供了一个链接，用于请求发送新的确认邮件，以防之前的邮件丢失。

重新发送账户确认邮件的代码如下：

```python
# app/auth/views.py
# @auth.route('/confirm')
# @login_required
# def resend_confirmation():
#     token = current_user.generate_confirmation_token()
#     send_email(current_user.email, 'Confirm Your Account',
#                'auth/email/confirm', user=current_user, token=token)
#     flash('A new confirmation email has been sent to you by email.')
#     return redirect(url_for('main.index'))
```

这个路由为`current_user`（即已登录的用户，也是目标用户）重做了一遍注册路由中的操作。这个路由也用`login_required`保护，确保访问时程序知道请求再次发送邮件的是哪个用户。


## 管理账户

拥有程序账户的用户有时可能需要修改账户信息。下面这些操作可使用本章介绍的技术添加到验证蓝图中。

-   **修改密码**

    只要用户处于登录状态，就可显示一个表单，要求用户输入旧密码和替换的新密码。

    **🔖 执行`git checkout 8d`签出程序的这个版本。**

-   **重设密码**

    为避免用户忘记密码无法登入的情况，程序可以提供重设密码功能。安全起见，有必要使用类似于确认账户时用到的令牌。用户请求重设密码后，程序会向用户注册时提供的电子邮件地址发送一封包含重设令牌的邮件。用户点击邮件中的链接，令牌验证后，会显示一个用于输入新密码的表单。

    ⚠️ 上述流程并未实现，只要用户输入正确的电子邮件地址即可重设密码。

    **🔖 执行`git checkout 8e`签出程序的这个版本。**

-   **修改电子邮件地址**

    程序可以提供修改注册电子邮件地址的功能，接受新地址之前，应当使用确认邮件进行验证（此功能也未实现）。

    **🔖 执行`git checkout 8f`签出程序的这个版本。**

## 脚注

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> 后面安装依赖包的方式类似，以后不再赘述。

<sup><a id="fn.2" class="footnum" href="#fnr.2">2</a></sup> 确保已执行 `flask db createtables` 创建了数据库表。
