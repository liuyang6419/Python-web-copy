# Flask应用测试

编写单元测试主要有两个目的：

1.  实现新功能时，确保新添加的代码按预期方式运行
2.  每次修改程序后，保证现有代码的功能没有退化，改动没有影响原有代码的正常运行


<a id="org49a6e41"></a>

## 获取代码覆盖报告

代码覆盖工具用来统计单元测试检查了多少程序功能，并提供一个详细的报告，说明程序的哪些代码没有测试到。它能指引你为最需要测试的部分编写新测试。

Python提供了一个优秀的代码覆盖工具，称为 [**coverage**](https://coverage.readthedocs.io)，使用 pip 进行安装：

```sh
(flaskr_env3) $ pip install coverage
```

这个工具本身是一个命令行脚本，可在任何一个Python程序中检查代码覆盖。另外，其还提供了更方便的脚本访问功能，使用编程方式启动覆盖检查引擎。为了能更好地把覆盖检测集成到<code>manage.py</code>中，我们可以增强之前自定义的<code>test</code>命令，添加可选选项<code>--coverage</code>。

```python
# manage.py
import os

cov = None
if os.environ.get('FLASK_COVERAGE'):
    import coverage
    cov = coverage.Coverage(branch=True, include='app/*')
    cov.start()

# ...

@app.cli.command()
@click.option('--coverage', default=False, is_flag=True,
              help=('Run the coverage test.'))
def test(coverage):
    """Run the unit tests."""
    if coverage and not os.environ.get('FLASK_COVERAGE'):
        import sys
        os.environ['FLASK_COVERAGE'] = '1'
        os.execvp(sys.executable, [sys.executable] + sys.argv)
    import unittest
    tests = unittest.TestLoader().discover('tests')
    unittest.TextTestRunner(verbosity=2).run(tests)
    if cov:
        cov.stop()
        cov.save()
        print('Coverage Summary:')
        cov.report()
        basedir = os.path.abspath(os.path.dirname(__file__))
        covdir = os.path.join(basedir, 'tmp/coverage')
        cov.html_report(directory=covdir)
        print('HTML version: file://%s/index.html' % covdir)
        cov.erase()


# ...
```

<code>test()</code>函数收到<code>--coverage</code>选项的值后再启动覆盖检测已经晚了，那时全局作用域中的所有代码都已经执行了。为了检测的准确性，设定完环境变量<CODE>FLASK_COVERAGE</CODE>后，脚本会重启。再次运行时，脚本顶端的代码发现已经设定了环境变量，于是立即启动覆盖检测。

函数<code>coverage.coverage()</code>用于启动覆盖检测引擎。<code>branch=True</code>选项开启分支覆盖分析，除了跟踪哪行代码已经执行外，还要检查每个条件语句的<code>True</code>分支 和<code>False</code>分支是否都执行了。<code>include</code>选项用来限制程序包中文件的分析范围，只对这些文件中的代码进行覆盖检测。如果不指定<code>include</code>选项，虚拟环境中安装的全部扩展和测试代码都会包含进覆盖报告中，给报告添加很多杂项。

执行完所有测试后，<code>text()</code>函数会在终端输出报告，同时还会生成一个使用HTML编写的精美报告并写入硬盘。HTML格式的报告非常适合直观形象地展示覆盖信息，因为它按照源码的使用情况给代码行加上了不同的颜色。

文本格式的报告示例如下：

    (flaskr_env3) $ flask test --help
    Usage: flask test [OPTIONS]

      Run the unit tests.

    Options:
      --coverage  Run the coverage test.
      --help      Show this message and exit.

    (flaskr_env3) $ flask test --coverage
    ...
    ----------------------------------------------------------------------
    Ran 21 tests in 8.683s

    OK
    Coverage Summary:
    Name                            Stmts   Miss Branch BrPart  Cover
    -----------------------------------------------------------------
    app/__init__.py                    31     15      0      0    52%
    app/api_1_0/__init__.py             3      0      0      0   100%
    app/api_1_0/authentication.py      30     19     10      0    28%
    app/api_1_0/comments.py            40     29     12      0    21%
    app/api_1_0/decorators.py          11      3      2      0    62%
    app/api_1_0/errors.py              17     10      0      0    41%
    app/api_1_0/posts.py               38     25      8      0    28%
    app/api_1_0/users.py               30     22     12      0    19%
    app/auth/__init__.py                3      0      0      0   100%
    app/auth/forms.py                  45      8      8      0    70%
    app/auth/views.py                  82     64     32      0    16%
    app/decorators.py                  20      4      4      1    71%
    app/exceptions.py                   2      0      0      0   100%
    app/main/__init__.py                6      1      0      0    83%
    app/main/errors.py                 20     15      6      0    19%
    app/main/forms.py                  39      7      6      0    71%
    app/main/views.py                 174    135     34      0    19%
    app/models.py                     288     88     56      8    65%
    -----------------------------------------------------------------
    TOTAL                             879    445    190      9    43%
    HTML version: file:///.../flaskr/tmp/coverage/index.html

上述报告显示，整体覆盖率为 **44%** 。情况并不遭，但也不太好。现阶段，模型类是单元测试的关注焦点，它共包含 **288** 个语句，测试覆盖了其中 **65%** 的语句。

很明显，<code>main</code>和<code>auth</code>蓝本中的<code>views.py</code>文件以及<code>api_1_0</code>蓝本中的路由的覆盖率都很低，因为我们没有为这些代码编写单元测试。

通过这个报告，我们就能很容易确定向测试组件中添加哪些测试以提高覆盖率。但是并非程序的所有组成部分都像数据库模型那样易于测试。接下来我们将介绍更高级的测试策略，可用于测试视图函数、表单和模板。


<a id="orgbd3b913"></a>

## Flask测试客户端

程序的某些代码严重依赖运行中的程序所创建的环境。例如，不能直接调用视图函数中的代码进行测试，因为这个函数可能需要访问Flask上下文全局变量，如<code>request</code>或<code>session</code>；视图函数可能还等待接收 **POST** 请求中的表单数据，而且某些视图函数要求用户先登录。简而言之，视图函数只能在请求上下文和运行中的程序里运行。

Flask内建了一个测试客户端用于解决（至少部分解决）这一问题。测试客户端能复现程序运行在Web服务器中的环境，让测试扮演成客户端从而发送请求。

在测试客户端中运行的视图函数和正常情况下的没有太大区别，服务器收到请求，将其分配给适当的视图函数，视图函数生成响应，将其返回给测试客户端。执行视图函数后，生成的响应会传入测试，检查是否正确。


<a id="org91e1149"></a>

### 测试Web程序

下面是使用测试客户端编写的单元测试框架：

```python
# tests/test_client.py
import unittest

from flask import url_for
from app import create_app, db
from app.models import User, Role


class FlaskClientTestCase(unittest.TestCase):
    def setUp(self):
        self.app = create_app('testing')
        self.app_context = self.app.app_context()
        self.app_context.push()
        db.database.create_tables(db.models, safe=True)
        Role.insert_roles()
        self.client = self.app.test_client(use_cookies=True)

    def tearDown(self):
        db.database.drop_tables(db.models, safe=True)
        self.app_context.pop()

    def test_home_page(self):
        response = self.client.get(url_for('main.index'))
        self.assertTrue(b'Stranger' in response.data)
```

实例变量<code>self.client</code>是Flask测试客户端对象。在这个对象上可调用方法向程序发起请求。如果创建测试客户端时启用了<code>use_cookies</code>选项，这个测试客户端就能像浏览器一样接收和发送cookie，因此能使用依赖cookie的功能记住请求之间的上下文。这个选项还可用来启用用户会话，让用户登录和退出。

<code>test_home_page()</code>测试作为一个简单的例子演示了测试客户端的作用。

客户端向首页发起了一个请求。在测试客户端上调用<code>get()</code>方法得到的结果是一个<code>Response</code>对象，内容是调用视图函数得到的响应。为了检查测试是否成功，要在响应主体中搜索是否包含 **“Stranger”** 这个词。响应主体可使用<code>response.get_data()</code>获取，而 “Stranger” 这个词包含在向匿名用户显示的欢迎消息“Hello, Stranger!”中。

⚠ 默认情况下<code>get_data()</code>得到的响应主体是一个字节数组，传入参数<code>as_text=True</code>后得到的是一个更易于处理的Unicode字符串。

测试客户端还能使用<code>post()</code>方法发送包含表单数据的POST请求。Flask-WTF生成的表单中包含一个隐藏字段，其内容是CSRF令牌，需要和表单中的数据一起提交。为了复现这个功能，测试必须请求包含表单的页面，然后解析响应返回的 HTML 代码并提取令牌，这样才能把令牌和表单中的数据一起发送。为了避免在测试中处理CSRF令牌这一烦琐操作，最好在测试配置中禁用 CSRF 保护功能。

```python
# config.py

class TestingConfig(Config):
    # ...
    WTF_CSRF_ENABLED = False
```

下面是一个更为高级的单元测试，模拟了新用户注册账户、登录、使用确认令牌确认账户以及退出的过程。

```python
# tests/test_client.py

class FlaskClientTestCase(unittest.TestCase):
    # ...
    def test_register_and_login(self):
        # register a new account
        response = self.client.post(
            url_for('auth.register'),
            data={'email': 'john@example.com',
                  'username': 'john',
                  'password': 'cat',
                  'password2': 'cat'})
        self.assertTrue(response.status_code == 302)

        # login with the new account
        response = self.client.post(
            url_for('auth.login'),
            data={'email': 'john@example.com',
                  'password': 'cat'},
            follow_redirects=True)
        self.assertTrue(re.search(b'Hello,\s+john\s+!', response.data))

        # send a confirmation token
        user = User.select().where(User.email == 'john@example.com').first()
        token = user.generate_confirmation_token()
        response = self.client.get(url_for('auth.confirm', token=token),
                                   follow_redirects=True)
        self.assertTrue(b'You have confirmed your account' in response.data)

        # log out
        response = self.client.get(url_for('auth.logout'),
                                   follow_redirects=True)
        self.assertTrue(b'You have been logged out' in response.data)
```

**🔖 执行<code>git checkout 15b</code>签出程序的这个版本。**


<a id="orgf3be2f6"></a>

### 测试Web服务

下面的实例展示了如何使用Flask测试客户端测试REST Web服务：

```python
# tests/test_api.py
class APITestCase(unittest.TestCase):
    # ...
    def get_api_headers(self, username, password):
        return {
            'Authorization': 'Basic ' + b64encode(
                (username + ':' + password).encode('utf-8')).decode('utf-8'),
            'Accept': 'application/json',
            'Content-Type': 'application/json'
        }

    def test_posts(self):
        # add a user
        r = Role.select().where(Role.name == 'User').first()
        self.assertIsNotNone(r)
        u = User(email='rose@example.com',
                 username='rose',
                 password='cat', confirmed=True,
                 role=r)
        u.save()

        # write an empty post
        response = self.client.post(
            url_for('api.new_post'),
            headers=self.get_api_headers('rose@example.com', 'cat'),
            data=json.dumps({'body': ''}))
        self.assertTrue(response.status_code == 400)

        # write a post
        response = self.client.post(
            url_for('api.new_post'),
            headers=self.get_api_headers('rose@example.com', 'cat'),
            data=json.dumps({'body': 'body of the *blog* post'}))
        self.assertTrue(response.status_code == 201)
        url = response.headers.get('Location')
        self.assertIsNotNone(url)

        # get the new post
        response = self.client.get(
            url,
            headers=self.get_api_headers('rose@example.com', 'cat'))
        self.assertTrue(response.status_code == 200)
        json_response = json.loads(response.data.decode('utf-8'))
        self.assertTrue(json_response['url'] == url)
        self.assertTrue(json_response['body'] == 'body of the *blog* post')
        self.assertTrue(json_response['body_html'] ==
                        '<p>body of the <em>blog</em> post</p>')
        json_post = json_response

        # get the post from the user
        response = self.client.get(
            url_for('api.get_user_posts', id=u.id),
            headers=self.get_api_headers('rose@example.com', 'cat'))
        self.assertTrue(response.status_code == 200)
        json_response = json.loads(response.data.decode('utf-8'))
        self.assertIsNotNone(json_response.get('posts'))
        self.assertTrue(json_response.get('count', 0) == 1)
        self.assertTrue(json_response['posts'][0] == json_post)

        # get the post from the user as a follower
        response = self.client.get(
            url_for('api.get_user_followed_posts', id=u.id),
            headers=self.get_api_headers('rose@example.com', 'cat'))
        self.assertTrue(response.status_code == 200)
        json_response = json.loads(response.data.decode('utf-8'))
        self.assertIsNotNone(json_response.get('posts'))
        self.assertTrue(json_response.get('count', 0) == 1)
        self.assertTrue(json_response['posts'][0] == json_post)

        # edit post
        response = self.client.put(
            url,
            headers=self.get_api_headers('rose@example.com', 'cat'),
            data=json.dumps({'body': 'updated body'}))
        self.assertTrue(response.status_code == 200)
        json_response = json.loads(response.data.decode('utf-8'))
        self.assertTrue(json_response['url'] == url)
        self.assertTrue(json_response['body'] == 'updated body')
        self.assertTrue(json_response['body_html'] == '<p>updated body</p>')
```

测试API时使用的<code>setUp()</code>和<code>tearDown()</code>方法和测试普通程序所用的一样， 不过API不使用cookie，所以无需配置相应支持。<code>get_api_headers()</code>是一个辅助方法，返回所有请求都要发送的通用首部，其中包含认证密令和MIME类型相关的首部。大多数测试都要发送这些首部。

<code>test_posts()</code>测试把一个用户插入数据库，然后使用基于REST的API创建一篇博客文章， 然后再读取这篇文章。所有请求主体中发送的数据都要使用<code>json.dumps()</code>方法进行编码，因为Flask测试客户端不会自动编码JSON格式数据。类似地，返回的响应主体也是JSON格式，处理之前必须使用<code>json.loads()</code>方法解码。

**🔖 执行<code>git checkout 15c</code>签出程序的这个版本。**


<a id="org45a3358"></a>

## 使用Selenium进行端到端测试

Flask测试客户端不能完全模拟运行中的程序所在的环境，例如无法像在真正的Web浏览器客户端中那样运行JavaScript代码。如果测试需要完整的环境，只能使用真正的Web浏览器连接Web服务器来运行程序。大多数浏览器都支持自动化操作。

[Selenium](http://www.seleniumhq.org/)是一个Web浏览器自动化工具，支持3种主要操作系统中的大多数主流Web浏览器。

使用pip来安装：

```
(flaskr_env3) $ pip install selenium
```

**安装WebDriver**

下载支持Chrome浏览器的[ChromeDriver](https://sites.google.com/a/chromium.org/chromedriver/home)<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>，注意查看其版本号，确保所下载的WebDriver支持你当前使用的浏览器版本。

将下载后的ChromeDriver安装到系统环境变量<CODE>PATH</CODE>设置的路径下面。<sup><a id="fnr.2" class="footref" href="#fn.2">2</a></sup>

使用Selenium进行的测试要求程序在Web服务器中运行，监听真实的HTTP请求。程序运行在后台线程里的开发服务器中，而测试运行在主线程中。在测试的控制下，Selenium启动Web浏览器并连接程序以执行所需操作。

使用这种方法要解决一个问题，即当所有测试都完成后，要停止Flask服务器，Werkzeug Web服务器本身就有停止选项，但由于服务器运行在单独的线程中，关闭服务器的唯一方法是发送一个普通的HTTP请求。

下面代码实现了关闭服务器的路由：

```python
# app/main/views.py
@main.route('/shutdown')
def server_shutdown():
    if not current_app.testing:
        abort(404)
    shutdown = request.environ.get('werkzeug.server.shutdown')
    if not shutdown:
        abort(500)
    shutdown()
    return 'Shutting down...'
```

只有当程序运行在测试环境中时，这个关闭服务器的路由才可用，在其他配置中调用时将不起作用。在实际过程中，关闭服务器时要调用Werkzeug在环境中提供的关闭函数。调用这个函数且请求处理完成后，开发服务器就知道自己需要优雅地退出了。

下面是使用Selenium运行测试时测试用例所用的代码结构：

```python
# tests/test_selenium.py
from selenium import webdriver


class SeleniumTestCase(unittest.TestCase):
    client = None

    @classmethod
    def setUpClass(cls):
        # start Chrome
        try:
            cls.client = webdriver.Chrome()
        except:
            pass

        # skip these tests if the browser could not be started
        if cls.client:
            # create the application
            cls.app = create_app('testing')
            cls.app_context = cls.app.app_context()
            cls.app_context.push()

            # suppress logging to keep unittest output clean
            import logging
            logger = logging.getLogger('werkzeug')
            logger.setLevel("ERROR")

            # create the database and populate with some fake data
            db.database.create_tables(db.models, safe=True)
            Role.insert_roles()
            User.generate_fake(10)
            Post.generate_fake(10)

            # add an administrator user
            admin_role = Role.select().where(Role.permissions == 0xff).first()
            admin = User(email='john@example.com',
                         username='john', password='cat',
                         role=admin_role, confirmed=True)
            admin.save()

            # start the Flask server in a thread
            threading.Thread(target=cls.app.run).start()

            # give the server a second to ensure it is up
            time.sleep(1)

    @classmethod
    def tearDownClass(cls):
        if cls.client:
            # stop the flask server and the browser
            cls.client.get('http://localhost:5000/shutdown')
            cls.client.close()

            # destroy database
            db.database.drop_tables(db.models, safe=True)

            # remove application context
            cls.app_context.pop()

    def setUp(self):
        if not self.client:
            self.skipTest('Web browser not available')

    def tearDown(self):
        pass
```

<code>setUpClass()</code>和<code>tearDownClass()</code>类方法分别在这个类中的全部测试运行前、后执行。

<code>setUpClass()</code>方法使用Selenium提供的webdriver API启动一个 Chrome 实例，并创建一个程序和数据库，其中写入了一些供测试使用的初始数据。然后调用标准的<code>app.run()</code>方法在一个线程中启动程序。完成所有测试后，程序会收到一个发往<code>/shutdown</code>的请求，进而停止后台线程。随后，关闭浏览器，删除测试数据库。

<code>setUp()</code>方法在每个测试运行之前执行，如果Selenium无法利用<code>startUpClass()</code>方法启动Web浏览器就跳过测试。

下面是一个使用Selenium进行测试的例子：

```python
# tests/test_selenium.py
class SeleniumTestCase(unittest.TestCase):
    # ...

    def test_admin_home_page(self):
        admin = User.select().where(User.email == 'john@example.com').first()
        self.assertTrue(isinstance(admin, User))

        # navigate to home page
        self.client.get('http://localhost:5000/')
        self.assertTrue(re.search('Hello,\s+Stranger\s+!',
                                  self.client.page_source))

        # navigate to login page
        self.client.find_element_by_link_text('Log In').click()
        self.assertTrue('<h1>Login</h1>' in self.client.page_source)

        # login
        self.client.find_element_by_name('email').\
            send_keys('john@example.com')
        self.client.find_element_by_name('password').send_keys('cat')
        self.client.find_element_by_name('submit').click()
        self.assertTrue(re.search('Hello,\s+john\s+!', self.client.page_source))

        # navigate to the user's profile page
        self.client.find_element_by_link_text('Profile').click()
        self.assertTrue('<h1>john</h1>' in self.client.page_source)
```

这个测试使用<code>setUpClass()</code>方法中创建的管理员账户登录程序，然后打开资料页。使用Selenium进行测试时，测试向Web浏览器发出指令且从不直接和程序交互。发给浏览器的指令和真实用户使用鼠标或键盘执行的操作几乎一样。

这个测试首先调用<code>get()</code>方法访问程序的首页。在浏览器中，这个操作就是在地址栏 中输入URL。为了验证这一步操作的结果，测试代码检查页面源码中是否包含“Hello, Stranger!” 这个欢迎消息。

为了访问登录页面，测试使用<code>find_element_by_link_text()</code>方法查找“Log In”链接，然后在这个链接上调用<code>click()</code>方法，从而在浏览器中触发一次真正的点击。Selenium提供了很多<code>find_element_by...()</code>简便方法，可使用不同的方式搜索元素。

为了登录程序，测试使用<code>find_element_by_name()</code>方法通过名字找到表单中的电子邮件 和密码字段，然后再使用<code>send_keys()</code>方法在各字段中填入值。表单的提交通过在提交按钮上调用<code>click()</code>方法完成。此外，还要检查针对用户定制的欢迎消息，以确保登录成功且浏览器显示的是首页。

测试的最后一部分是找到导航条中的“Profile”链接，然后点击。为证实资料页已经加载，测试要在页面源码中搜索内容为用户名的标题。

**🔖 执行<code>git checkout 15d</code>签出程序的这个版本。**

最后再来进行一次代码覆盖测试：

    (flaskr_env3) $ flask test --coverage
    ...
    ----------------------------------------------------------------------
    Ran 33 tests in 35.238s

    OK
    Coverage Summary:
    Name                            Stmts   Miss Branch BrPart  Cover
    -----------------------------------------------------------------
    app/__init__.py                    31     15      0      0    52%
    app/api_1_0/__init__.py             3      0      0      0   100%
    app/api_1_0/authentication.py      30      2     10      2    90%
    app/api_1_0/comments.py            40      4     12      4    85%
    app/api_1_0/decorators.py          11      1      2      1    85%
    app/api_1_0/errors.py              17      0      0      0   100%
    app/api_1_0/posts.py               38      3      8      3    87%
    app/api_1_0/users.py               30      4     12      4    81%
    app/auth/__init__.py                3      0      0      0   100%
    app/auth/forms.py                  45      6      8      2    77%
    app/auth/views.py                  90     48     36      4    40%
    app/decorators.py                  20      4      4      1    71%
    app/exceptions.py                   2      0      0      0   100%
    app/main/__init__.py                6      0      0      0   100%
    app/main/errors.py                 20     10      6      0    46%
    app/main/forms.py                  39      7      6      0    71%
    app/main/views.py                 182    122     38      4    30%
    app/models.py                     288     18     56     11    90%
    -----------------------------------------------------------------
    TOTAL                             895    244    198     36    68%
    HTML version: file:///.../flaskr/tmp/coverage/index.html

## Footnotes

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> 使用Firefox浏览器的话，可下载[GeckoDriver](https://github.com/mozilla/geckodriver)。

<sup><a id="fn.2" class="footnum" href="#fnr.2">2</a></sup> 更详细的信息[这里](https://github.com/SeleniumHQ/selenium/wiki/ChromeDriver)。
