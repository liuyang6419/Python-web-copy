# Flask大型应用程序结构

Flask并不强制要求大型项目使用特定的组织方式，程序结构的组织方式完全由开发者决定。

下面我们将介绍一种使用包和模块组织大型程序的方式。


## 项目结构

Flask程序的基本结构如下：

    │Project
    ├── README.md
    ├── app
    │   ├── __init__.py
    │   ├── api
    │   ├── auth
    │   ├── decorators.py
    │   ├── main
    │   │   ├── __init__.py
    │   │   ├── errors.py
    │   │   ├── forms.py
    │   │   └── views.py
    │   ├── models.py
    │   ├── static
    │   └── templates
    ├── config.py
    ├── manage.py
    ├── migrations
    ├── tests
    ├── prod
    ├── requirements
    └── utils

顶级文件夹：

-   Flask程序一般都保存在名为`app`的包中；
-   `migrations`文件夹包含数据库迁移脚本；
-   单元测试编写在`tests`包中；
-   `prod`文件夹包含生产环境部署配置文件；
-   `utils`包中放置工具函数或者可独立使用的库；
-   `requirements`文件夹包含所有依赖包（不同环境），用来生成虚拟环境。

一些文件：

-   `config.py`存储配置；
-   `manage.py`用于指定`flask`命令的运行的程序和其他任务命令；
-   `README.md`项目介绍。

下面几节介绍如何把`hello.py`程序转换成上面这种结构。


## 配置选项

程序经常需要设定多个配置，比如开发、测试和生产环境要使用不同的数据库。

下面的代码展示了使用层次结构的配置类。

```python
# config.py

import os
basedir = os.path.abspath(os.path.dirname(__file__))


class Config(object):
    PROJECT_DIR = basedir
    SECRET_KEY = (os.environ.get('SECRET_KEY') or
                  '44617457d542163d10ada66726b31ef80a88ac1a41013ea5')

    PEEWEE_MODELS_MODULE = 'app.models'

    @classmethod
    def init_app(cls, app):
        pass


class DevelopmentConfig(Config):
    DEBUG = True
    PEEWEE_DATABASE_URI = (
        os.environ.get('DEV_DATABASE_URL') or
        'sqlite:///' + os.path.join(basedir, 'data-dev.sqlite')
    )


class TestingConfig(Config):
    TESTING = True
    PEEWEE_DATABASE_URI = (
        os.environ.get('DEV_DATABASE_URL') or
        'sqlite:///' + os.path.join(basedir, 'data-test.sqlite')
    )


class ProductionConfig(Config):
    DEBUG = False

    PEEWEE_DATABASE_URI = (
        os.environ.get('PROD_DATABASE_URL') or
        'sqlite:///' + os.path.join(basedir, 'data.sqlite')
    )


config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,

    'default': DevelopmentConfig
}
```

基类`Config`中包含通用配置，子类分别定义专用的配置。如果需要，你还可添加其他配置类。

为了让配置方式更灵活且更安全，某些配置可以从环境变量中导入。例如，`SECRET_KEY`的值，可以在环境中设定，但系统也提供了一个默认值，以防环境中没有定义。

基类`Config`中变量`PEEWEE_MODELS_MODULE`是 **Flask-PW** 扩展的设置选项，用来指定程序的数据模型定义模块的路径。

在3个子类中，`PEEWEE_DATABASE_URI`变量都被指定了不同的值。这样程序在不同的配置环境中运行就可使用不同的数据库。

配置类可以定义`init_app()`类方法，其参数是程序实例。在这个方法中，可以执行对当前环境的配置初始化。现在，基类`Config`中的`init_app()`方法为空。

配置脚本末尾，`config`字典中注册了不同的配置环境，而且还注册了一个默认配置（开发环境）。


## 程序包

程序包用来保存程序的所有代码、模板和静态文件。可以把这个包直接称为`app`（应用），如果有需求，也可使用一个程序专用名字。

`templates`和`static`文件夹是程序包的一部分，因此到后面这两个文件夹被移到了`app`中。

数据库模型被移到了这个包中，保存为`app/models.py`。


### 程序工厂函数

为了动态修改配置或在不同的配置环境中运行程序，可以延迟创建程序实例，将其创建过程移到可显式调用的工厂函数中。这样不仅可以给脚本留出配置程序的时间，还能够创建多个程序实例，这些实例有时在测试中非常有用。

程序的工厂函数在`app`包的构造文件中定义，如下所示：

```python
# app/__init__.py
from flask import Flask

from flask_bootstrap import Bootstrap
from flask_moment import Moment
from flask_pw import Peewee

from config import config


bootstrap = Bootstrap()
moment = Moment()
db = Peewee()


def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    bootstrap.init_app(app)
    moment.init_app(app)
    db.init_app(app)
    app.cli.add_command(db.cli, 'db')

    # 附加路由和自定义的错误页面

    return app
```

工厂函数返回创建的程序示例，现在工厂函数创建的程序还不完整，因为没有路由和自定义的错误页面处理程序。


### Flask蓝图

在单脚本程序中，程序实例存在于全局作用域中，路由可以直接使用`app.route`装饰器定义。但现在程序在运行时创建，只有调用`create_app()`之后才能使用`app.route`装饰器，这时定义路由就太晚了。另外，自定义的错误页面处理程序也面临相同的困难，因为错误页面处理程序使用`app.errorhandler`装饰器定义。

Flask使用蓝图（Blueprint）提供了更好地解决方法。

蓝图实现了应用的模块化，其和程序类似，也可以定义路由，在蓝图中定义的路由处于休眠状态，直到蓝图注册到程序上后，路由才真正成为程序的一部分。蓝图通常作用于相同的URL前缀，比如`/user/:id`、`/user/profile`这样的地址，都以`/user`开头，那么它们就可以放在一个模块中。

使用蓝图让应用层次清晰，开发者可以更容易地开发和维护项目。

看一个简单的示例：

```python
# user.py
from flask import Blueprint

bp = Blueprint('user', __name__, url_prefix='/user')


@bp.route('/')
def index():
    return 'User"s index page'
```

通过实例化一个`Blueprint`类对象来创建蓝图。这个构造函数有两个必须指定的参数： 蓝图的名字和蓝图所在的包或模块。和程序一样，大多数情况下第二个参数使用 Python 的`__name__`变量即可。`url_prefix`是可选参数，如果设定，注册蓝图后其中 定义的所有路由都会加上指定的前缀，此例中为`/user`。

再看主程序：

```python
# app.py
from flask import Flask
import user


app = Flask(__name__)
app.register_blueprint(user.bp)
```

使用`register_blueprint`注册模块，如果想去掉模块只需要去掉对应的注册语句即可。

为了获得最大的灵活性，程序包中创建了一个子包，用于保存蓝图。下面的示例是这个子包的构造文件，蓝图就创建于此。

```python
# app/main/__init__.py
from flask import Blueprint

main = Blueprint('main', __name__)

from . import views, errors
```

程序的路由保存在包里的`app/main/views.py`模块中，而错误处理程序保存在`app/main/errors.py`模块中。导入这两个模块就能把路由和错误处理程序与蓝图关联起来。

**⚠️ 这些模块必须在`app/main/__init__.py`脚本的末尾导入，** 为了避免循环导入依赖，因为在`views.py`和`errors.py`中还要导入蓝图`main`。

蓝图在工厂函数`create_app()`中注册到程序上，如下所示：

```python
# app/__init__.py
def create_app(config_name):
    # ...

    from .main import main as main_blueprint
    app.register_blueprint(main_blueprint)

    return app
```

下面是错误处理程序：

```python
# app/main/errors.py
from flask import render_template

from . import main


@main.app_errorhandler(404)
def page_not_found(e):
    return render_template('404.html'), 404


@main.app_errorhandler(500)
def internal_server_error(e):
    return render_template('500.html'), 500
```

在蓝图中编写错误处理程序稍有不同，如果使用`errorhandler`装饰器，那么只有蓝图中的 错误才能触发处理程序。要想注册程序全局的错误处理程序，必须使用`app_errorhandler`。

在蓝图中定义程序路由：

```python
# app/main/views.py
from datetime import datetime

from flask import render_template, session, redirect, url_for

from . import main
from .forms import NameForm
from .. import db
from ..models import User


@main.route('/', methods=['GET', 'POST'])
def index():
    form = NameForm()
    if form.validate_on_submit():
        # ...
        return redirect(url_for('.index'))
    return render_template('index.html',
                           form=form, name=session.get('name'),
                           known=session.get('known', False),
                           current_time=datetime.utcnow())
```

在蓝图中编写视图函数主要有两点不同：

1.  和前面的错误处理程序一样，路由装饰器由蓝图提供；
2.  `url_for()`函数的用法不同。

    `url_for()`函数的第一个参数是路由的端点名，在程序的路由中，默认为视图函数的名字。例如，在单脚本程序中，`index()`视图函数的URL可使用`url_for('index')`获取。

    蓝图中就不一样了，Flask会为蓝图中的全部端点加上一个命名空间，这样就可以在不 同的蓝图中使用相同的端点名定义视图函数，而不会产生冲突。命名空间就是蓝图的名字（`Blueprint`构造函数的第一个参数），所以视图函数`index()`注册的端点名是`main.index`，其URL使用`url_for('main.index')`获取。

    `url_for()`函数还支持一种简写的端点形式，在蓝图中可以省略蓝图名，例如`url_for('.index')`。在这种写法中，命名空间是当前请求所在的蓝图。这意味着同一蓝图中的重定向可以使用简写形式，但跨蓝图的重定向必须使用带有命名空间 的端点名。

此外，表单对象也要移到蓝图中，保存于`app/main/forms.py`模块。


## 命令行脚本

顶级文件夹中的`manage.py`用于指定`flask`命令的运行的程序和其他任务命令。

```python
# manage.py
import os
import sys

import click

from app import create_app, db


app = create_app(os.getenv('FLASK_CONFIG') or 'default')


@app.cli.command()
def create_tables():
    db.database.create_tables(db.models)
```

这个脚本先创建程序。如果已经定义了环境变量`FLASK_CONFIG`，则从中读取配置名；否则使用默认配置。

另外，添加了一个命令`create_tables`，用来根据定义的模型创建数据库表。


## 需求文件

程序中必须包含一个`requirements.txt`文件，用于记录所有依赖包及其精确的版本号。

这里我们使用一个工具 [pip-tools](https://github.com/jazzband/pip-tools) 来进行依赖包管理。

**pip-tools = pip-compile + pip-sync**

有两个文件来进行依赖包管理：

-   `requirements.in`（手动创建）包含了项目中直接使用到的包
-   `requirements.txt`（通过`pip-compile requirements.in`创建）包含所有包包括依赖包

项目中如果用到新的依赖包，将包的名字添加到`requirements.in`中，然后使用`pip-compile requirements.in`生成`requirements.txt`文件，再使用`pip-sync requirements.txt`安装所有包。

更多关于`pip-tools`的使用介绍，请参考其文档。

下面所示为`requirements.in`文件的内容：

    flask
    flask-bootstrap
    flask-wtf
    flask-moment
    flask-pw
    pip-tools


## 单元测试

为了演示，编写两个简单的测试：

```python
# tests/test_basics.py
import unittest

from flask import current_app
from app import create_app, db


class BasicsTestCase(unittest.TestCase):
    def setUp(self):
        self.app = create_app('testing')
        self.app_context = self.app.app_context()
        self.app_context.push()
        # create all db tables

    def tearDown(self):
        # drop all db tables
        self.app_context.pop()

    def test_app_exists(self):
        self.assertFalse(current_app is None)

    def test_app_is_testing(self):
        self.assertTrue(current_app.config['TESTING'])
```

这个测试使用 Python 标准库中的`unittest`包编写。`setUp()`和`tearDown()`方法分别在各测试前后运行，并且名字以`test_`开头的函数 都作为测试执行。

如果想进一步了解如何使用Python的`unittest`包编写测试，请阅读[官方文档](https://docs.python.org/3/library/unittest.html)。

`setUp()`方法尝试创建一个测试环境，类似于运行中的程序。首先，使用测试配置创建程序，然后激活上下文。这一步的作用是确保能在测试中使用`current_app`，像普通请求一样。然后创建一个全新的数据库。数据库和程序上下文在`tearDown()`方法中删除。

第一个测试确保程序实例存在。第二个测试确保程序在测试配置中运行。若想把`tests`文件夹作为包使用，需要添加`tests/__init__.py`文件，这个文件可以为空，因为`unittest`包会扫描所有模块并查找测试。

为了运行单元测试，可以在`manage.py`脚本中添加一个自定义命令`test`。

```python
# manage.py
@app.cli.command()
def test():
    """Run the unit tests."""
    import unittest
    tests = unittest.TestLoader().discover('tests')
    unittest.TextTestRunner(verbosity=2).run(tests)
```

修饰函数名就是命令名，函数的文档字符串会显示在帮助消息中。`test()`函数的定义体中调用了`unittest`包提供的测试运行函数。

单元测试可使用下面的命令运行：

```sh
(flaskr_env3) $ export FLASK_APP=manage.py
(flaskr_env3) $ flask test
test_app_exists (test_basics.BasicsTestCase) ... ok
test_app_is_testing (test_basics.BasicsTestCase) ... ok

----------------------------------------------------------------------
Ran 2 tests in 0.003s

OK
```

**🔖 执行`git checkout 7a`签出程序的这个版本。**


## 新的创建数据库命令

Flask-PW 提供了一个`db`命令，下面的代码展示了如何将新的创建数据库命令`createtables`添加`db`命令下面，作为其子命令使用。

```python
# manage.py
from flask.cli import with_appcontext

from flask_pw import BaseSignalModel


@db.cli.command('createtables', short_help='Create database tables.')
@click.option('--safe', default=False, is_flag=True,
              help=('Check first whether the table exists '
                    'before attempting to create it.'))
@click.argument('models', nargs=-1, type=click.UNPROCESSED)
@with_appcontext
def create_tables(models, safe):
    from importlib import import_module
    from flask.globals import _app_ctx_stack
    app = _app_ctx_stack.top.app
    if models:
        pw_models = []

        module = import_module(app.config['PEEWEE_MODELS_MODULE'])
        for mod in models:
            model = getattr(module, mod)
            if not isinstance(model, BaseSignalModel):
                continue
            pw_models.append(model)
        if pw_models:
            db.database.create_tables(pw_models, safe)
        return
    db.database.create_tables(db.models, safe)
```

**🔖 执行`git checkout 7b`签出程序的这个版本。**
