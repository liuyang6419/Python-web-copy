# Flask命令行接口

在之前的章节 **Flask框架简介** 中我们已经使用过Flask的命令行接口。


## Click

[Click](https://github.com/pallets/click) 是 Flask 的开发团队 [Pallets](https://github.com/pallets) 的另一款开源工具，用于快速创建命令行命令。Python 内置了一个 **Argparse** 的标准库用于创建命令行，但使用起来有些繁琐。<sup><a id="fnr.1" class="footref" href="#fn.1">1</a></sup>


### 快速使用

Click 的使用大致有两个步骤：

-   使用`@click.command()`装饰一个函数，使之成为命令行接口；

-   使用`@click.option()`等装饰函数，为其添加命令行选项等。

它的一种典型使用形式如下：

```python
import click

@click.command()
@click.option('--param', default=default_value, help='description')
def func(param):
    pass
```

看一下官方文档的入门例子：

```python
import click

@click.command()
@click.option('--count', default=1, help='Number of greetings.')
@click.option('--name', prompt='Your name', help='The person to greet.')
def hello(count, name):
    """Simple program that greets NAME for a total of COUNT times."""
    for x in range(count):
        click.echo('Hello %s!' % name)

if __name__ == '__main__':
    hello()
```

在上面的例子中，函数`hello`有两个参数：`count`和`name`，它们的值从命令行中获取。

-   `@click.command()`使函数`hello`成为命令行接口；
-   `@click.option`的第一个参数指定了命令行选项的名称，`count`的默认值是`1`，`name`的值从输入获取；

-   使用`click.echo`进行输出是为了获得更好的兼容性，因为`print`在 Python2 和 Python3 的用法有些差别。


### `click.option`

`option`最基本的用法就是通过指定命令行选项的名称，从命令行读取参数值，再将其传递给函数。在上面的例子，除了设置命令行选项的名称，还会指定默认值，help 说明等，`option`常用的设置参数如下：

-   `default`：设置命令行参数的默认值
-   `help`：参数说明
-   `type`：参数类型，可以是`string`、`int`、`float`等
-   `prompt`：当在命令行中没有输入相应的参数时，会根据`prompt`提示用户输入
-   `nargs`：指定命令行参数接收的值的个数


#### 指定 `type`

可以使用`type`来指定参数类型：

```python
import click

@click.command()
@click.option('--rate', type=float, help='rate')
def show(rate):
    click.echo('rate: %s' % rate)

if __name__ == '__main__':
    show()
```

```
$ python click_type.py --rate 1
rate: 1.0
$ python click_type.py --rate 0.66
rate: 0.66
```

#### 可选值

在某些情况下，一个参数的值只能是某些可选的值，如果用户输入了其他值， 应该提示用户输入正确的值。在这种情况下，可以通过`click.Choice()`来限定：

```python
import click

@click.command()
@click.option('--gender', type=click.Choice(['man', 'woman']))
def choose(gender):
    click.echo('gender: %s' % gender)

if __name__ == '__main__':
    choose()
```

    $ python click_choice.py --gender boy
    Usage: click_choice.py [OPTIONS]

    Error: Invalid value for "--gender": invalid choice: boy. (choose from man, woman)

    $ python click_choice.py --gender man
    gender: man


#### 多值参数

有时，一个参数需要接收多个值。`option`支持设置固定长度的参数值，通过`nargs`指定。

```python
import click

@click.command()
@click.option('--center', nargs=2, type=float, help='center of the circle')
@click.option('--radius', type=float, help='radius of the circle')
def circle(center, radius):
    click.echo('center: %s, radius: %s' % (center, radius))

if __name__ == '__main__':
    circle()
```

`option`指定了两个参数：`center`和`radius`，`center`表示二维平面上一个圆的圆心坐标，接收两个值，以元组的形式将值传递给函数，`radius`表示圆的半径。

    $ python click_multi_values.py --center 3 4 --radius 10
    center: (3.0, 4.0), radius: 10.0

    $ python click_multi_values.py --center 3 4 5 --radius 10
    Usage: click_multi_values.py [OPTIONS]

    Error: Got unexpected extra argument (5)


#### 输入密码

`option`提供了两个参数来设置密码的输入：

-   `hide_input`：用于隐藏输入
-   `confirmation_promt`：用于确认输入

```python
import click

@click.command()
@click.option('--password', prompt=True, hide_input=True, confirmation_prompt=True)
def input_password(password):
    click.echo('password: %s' % password)

if __name__ == '__main__':
    input_password()
```

    $ python click_password.py
    Password:
    Repeat for confirmation:
    password: 666666

Click 也提供了一种快捷的方式，通过使用`@click.password_option()`，上面的代码可以简写成：

```python
import click

@click.command()
@click.password_option()
def input_password(password):
    click.echo('password: %s' % password)

if __name__ == '__main__':
    input_password()
```


#### 改变命令行程序的执行

有些参数会改变命令行程序的执行，比如在终端输入`python`是进入 Python 控制台， 而输入`python --version`是打印 Python 版本。Click 提供`eager`标识对参数名进行标识，如果输入该参数，则会拦截既定的命令行执行流程， 跳转去执行一个回调函数。

```python
import click

def print_version(ctx, param, value):
    if not value or ctx.resilient_parsing:
        return
    click.echo('Version 1.0')
    ctx.exit()

@click.command()
@click.option('--version', is_flag=True, callback=print_version,
              expose_value=False, is_eager=True)
@click.option('--name', default='Ethan', help='name')
def hello(name):
    click.echo('Hello %s!' % name)

if __name__ == '__main__':
    hello()
```

其中：

-   `is_eager=True`表明该命令行选项优先级高于其他选项；
-   `expose_value=False`表示如果没有输入该命令行选项，会执行既定的命令行流程；
-   `callback`指定了输入该命令行选项时，要跳转执行的函数；

    ``` 
    $ python click_eager.py
    Hello Ethan!

    $ python click_eager.py --version
    Version 1.0

    $ python click_eager.py --name Michael
    Hello Michael!

    $ python click_eager.py --version --name Ethan
    Version 1.0
    ```

### `click.argument`

除了使用`@click.option`来添加可选参数，还经常使用`@click.argument`来添加固定参数。

一个简单的例子：

```python
import click

@click.command()
@click.argument('coordinates')
def show(coordinates):
    click.echo('coordinates: %s' % coordinates)

if __name__ == '__main__':
    show()
```

    $ python click_argument.py
    Usage: click_argument.py [OPTIONS] COORDINATES

    Error: Missing argument "coordinates".

    $ python click_argument.py --help
    Usage: click_argument.py [OPTIONS] COORDINATES

    Options:
      --help  Show this message and exit.

    $ python click_argument.py --coordinates 10
    Error: no such option: --coordinates

    $ python click_argument.py 10
    coordinates: 10


#### 多个 `argument`

```python
import click

@click.command()
@click.argument('x')
@click.argument('y')
@click.argument('z')
def show(x, y, z):
    click.echo('x: %s, y: %s, z:%s' % (x, y, z))

if __name__ == '__main__':
    show()
```

    $ python click_argument.py 10 20 30
    x: 10, y: 20, z:30

    $ python click_argument.py 10
    Usage: click_argument.py [OPTIONS] X Y Z

    Error: Missing argument "y".

    $ python click_argument.py 10 20
    Usage: click_argument.py [OPTIONS] X Y Z

    Error: Missing argument "z".

    $ python click_argument.py 10 20 30 40
    Usage: click_argument.py [OPTIONS] X Y Z

    Error: Got unexpected extra argument (40)


#### 不定参数

`argument`还有另外一种常见的用法，就是接收不定量的参数：

```python
import click

@click.command()
@click.argument('src', nargs=-1)
@click.argument('dst', nargs=1)
def move(src, dst):
    click.echo('move %s to %s' % (src, dst))

if __name__ == '__main__':
    move()
```

其中， `nargs=-1` 表明参数`src`接收不定量的参数值，参数值会以元组的形式传入函数。如果`nargs`大于等于 1，表示接收`nargs`个参数值，上面的例子中，`dst`接收一个参数值。

    $ python click_argument.py file1 trash
    move (u'file1',) to trash

    $ python click_argument.py file1 file2 file3 trash
    move (u'file1', u'file2', u'file3') to trash


### 彩色输出

使用`click.echo`进行输出，如果配合 [colorama](https://github.com/tartley/colorama) 这个模块，可以使用`click.secho`进行彩色输出。

使用 pip 安装 colorama：

```sh
(flaskr_env3) $ pip install colorama
```

```python
import click

@click.command()
@click.option('--name', help='The person to greet.')
def hello(name):
    click.secho('Hello %s!' % name, fg='red', underline=True)
    click.secho('Hello %s!' % name, fg='yellow', bg='black')

if __name__ == '__main__':
    hello()
```

-   `fg`表示前景色（即字体颜色），可选值：
    -   `BLACK`
    -   `RED`
    -   `GREEN`
    -   `YELLOW`
    -   `BLUE`
    -   `MAGENTA`
    -   `CYAN`
    -   `WHITE`
    -   &#x2026;

-   `bg`表示背景色，可选值：
    -   `BLACK`
    -   `RED`
    -   `GREEN`
    -   `YELLOW`
    -   `BLUE`
    -   `MAGENTA`
    -   `CYAN`
    -   `WHITE`
    -   &#x2026;

-   `underline`表示下划线，可选的样式：
    -   `dim=True`
    -   `bold=True`


### 小结

-   使用`click.command()`装饰一个函数，使其成为命令行接口。
-   使用`click.option()`添加可选参数，支持设置固定长度的参数值。
-   使用`click.argument()`添加固定参数，支持设置不定长度的参数值。


## 运行Shell

使用下面的shell命令来开启一个交互式的Python shell：

    (flaskr_env3) $ flask shell

这将开启一个交互式的Python shell，并且在其中设置好了正确的应用上下文和 本地变量（local variables）。这是通过调用应用的`Flask.make_shell_context()`方法做到的。默认地你将可访问到`app`和`g`。


## 自定义命令

Flask 使用 [click](http://click.pocoo.org/) 来实现命令行接口，这使得添加自定义命令非常容易。例如，如果你想 要一个shell 命令来初始化数据库，你可以这样做：

```python
import click
from flask import Flask

app = Flask(__name__)

@app.cli.command()
def initdb():
    """Initialize the database."""
    click.echo('Init the db')
```

在命令行中执行这个命令：

```sh
(flaskr_env3) $ flask initdb
Init the db
```


## 应用上下文

大多数命令需要对应用对象执行操作，所以为它们设置好应用上下文非常有必要。正因为如此，如果你在`app.cli`上通过`command()`注册了一个回调，这个回调将自动被`cli.with_appcontext()`包装（wrapped）来通知命令行系统确保设置好一个应用上下文。如果一个命令是通过`add_command`或其他方法稍后添加的，那这种行为将不可用。

这种行为可以通过向装饰器传递`with_appcontext=False`来禁用：

```python
@app.cli.command(with_appcontext=False)
def example():
    pass
```

## 脚注

<sup><a id="fn.1" class="footnum" href="#fnr.1">1</a></sup> Click 相比于 Argparse，就好比 requests 相比于 urllib 。
