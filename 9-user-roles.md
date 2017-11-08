# 用户角色

有多种方法可用于在程序中实现角色。具体采用何种实现方法取决于所需角色的数量和细分程度。

本章介绍的用户角色实现方式结合了分立的角色和权限，赋予用户分立的角色，但角色使用权限定义。


## 角色在数据库中的表示

下面是改进后的`Role`模型，增加了角色的权限：

```python
# app/models.py
class Role(db.Model):
    name = pw.CharField(64, unique=True)
    default = pw.BooleanField(default=False, index=True)
    permissions = pw.IntegerField(null=True)
```

只有一个角色的`default`字段要设为`True`，其他都设为`False`。用户注册时，其角色会被设为默认角色。

第二处改动是添加了`permissions`字段，其值是一个整数，表示位标志。各操作都对应一个位位置，能执行某项操作的角色，其位会被设为`1`。

各操作所需的程序权限是不一样的，如下表所示：

| 操作        | 位值                | 说明          |
|-------------|---------------------|---------------|
| 关注用户    | `0b00000001 (0x01)` | 关注其他用户  |
| 在他人的文章中发表评论 | `0b00000010 (0x02)` | 在他人撰写的文章中发布评论 |
| 写文章      | `0b00000100 (0x04)` | 写原创文章    |
| 管理他人发表的评论 | `0b00001000 (0x08)` | 查处他人发表的不当评论 |
| 管理员权限  | `0b10000000 (0x80)` | 管理网站      |

⚠️ 操作的权限使用 8 位表示，现在只用了其中 5 位，其他 3 位可用于将来的扩充。

使用如下的代码表示权限：

```python
# app/models.py
class Permission:
    FOLLOW = 0x01
    COMMENT = 0x02
    WRITE_ARTICLES = 0x04
    MODERATE_COMMENTS = 0x08
    ADMINISTER = 0x80
```

下表列出了要支持的用户角色以及定义角色使用的位权限：

| 用户角色 | 权限                | 说明                             |
|----------|---------------------|----------------------------------|
| 匿名 | `0b00000000 (0x00)` | 未登录的用户。在程序中只有阅读权限 |
| 用户 | `0b00000111 (0x07)` | 具有发布文章、发表评论和关注其他用户的权限。这是新用户的默认角色 |
| 协管员 | `0b00001111 (0x0f)` | 增加审查不当评论的权限           |
| 管理员 | `0b11111111 (0xff)` | 具有所有权限，包括修改其他用户所属角色的权限 |

使用权限组织角色，以后添加新角色时只需使用不同的权限组合即可。

在`Role`类中添加一个类方法用来将角色添加到数据库中：

```python
# app/models.py
class Role(db.Model):
    # ...
    @staticmethod
    def insert_roles():
        roles = {
            'User': (Permission.FOLLOW |
                     Permission.COMMENT |
                     Permission.WRITE_ARTICLES, True),
            'Moderator': (Permission.FOLLOW |
                          Permission.COMMENT |
                          Permission.WRITE_ARTICLES |
                          Permission.MODERATE_COMMENTS, False),
            'Administrator': (0xff, False)
        }
        for r in roles:
            role = Role.select().where(Role.name == r).first()
            if role is None:
                role = Role(name=r)
            role.permissions = roles[r][0]
            role.default = roles[r][1]
            role.save()
```

`insert_roles()`函数先通过角色名查找现有的角色，然后再进行更新。只有当数据库中没有某个角色名时才会创建新角色对象。如果以后更新了角色列表，就可以执行更新操作。要想添加新角色，或者修改角色的权限，修改`roles`数组，再运行函数即可。

使用 flask shell 会话将角色写入数据库：

```python
(flaskr_env3) $ flask shell
>>> Role.insert_roles()
>>> list(Role.select())
[<Role 'User'>, <Role 'Moderator'>, <Role 'Administrator'>]
```

## 赋予角色

用户在程序中注册账户时，即被赋予适当的角色。大多数用户在注册时赋予的角色都是 “用户”，即默认角色。管理员作为唯一的例外，应根据保存在设置变量`FLASKR_ADMIN`中 的电子邮件地址被赋予“管理员”角色。

下面的代码用来定义默认的用户角色：

```python
# app/models.py
class User(UserMixin, db.Model):
    # ...
    def __init__(self, **kwargs):
        super(User, self).__init__(**kwargs)
        if self.role is None:
            if self.email == current_app.config['FLASKR_ADMIN']:
                self.role = (Role.select()
                             .where(Role.permissions == 0xff)
                             .first())
            if self.role is None:
                self.role = Role.select().where(Role.default == True).first()
```

`User`类的构造函数首先调用基类的构造函数，如果创建基类对象后还没定义角色，则根据 电子邮件地址决定将其设为管理员还是默认角色。


## 角色验证

下面的代码用来检查用户是否有指定的权限：

```python
# app/models.py
from flask_login import UserMixin, AnonymousUserMixin

class User(UserMixin, db.Model):
    # ...

    def can(self, permissions):
        return (self.role is not None and
                (self.role.permissions & permissions) == permissions)

    def is_administrator(self):
        return self.can(Permission.ADMINISTER)


class AnonymousUser(AnonymousUserMixin):
    def can(self, permissions):
        return False

    def is_administrator(self):
        return False


login_manager.anonymouse_user = AnonymousUser
```

`can()`方法在请求和赋予角色这两种权限之间进行 **位与操作** 。如果角色中包含请求的所有权限位，则返回`True`，表示允许用户执行此项操作。检查管理员权限的功能使用单独的方法`is_administrator()`实现。

继承自 Flask-Login 中`AnonymousUserMixin`类的`AnonymousUser`类，也实现了`can()`方法和`is_administrator()`方法。通过将其设为用户未登录时`current_user`的值，程序不用先检查用户是否登录，也能自由调用`current_user.can()`和`current_user.is_administrator()`。

如果想让视图函数只对具有特定权限的用户开放，可以使用自定义的装饰器：

```python
# app/decorators.py
from functools import wraps

from flask import abort
from flask_login import current_user

from .models import Permission


def permission_required(permission):
    def decorator(f):
        @wraps(f)
        def decorated_function(*args, **kwargs):
            if not current_user.can(permission):
                abort(403)
            return f(*args, **kwargs)
        return decorated_function
    return decorator


def admin_required(f):
    return permission_required(Permission.ADMINISTER)(f)
```

上面代码实现了两个装饰器，一个用来检查常规权限，一个专门用来检查管理员权限。如果用户不具有指定权限，则返回 **403** 错误码，即HTTP“禁止”错误。

下面的代码演示如何使用上面的装饰器：

```python
from decorators import admin_required, permission_required
from .models import Permission

@main.route('/admin')
@login_required
@admin_required
def for_admins_only():
    return "For administrators!"

@main.route('/moderator')
@login_required
@permission_required(Permission.MODERATE_COMMENTS)
def for_moderators_only():
    return "For comment moderators!"
```

为了方便在模板中检查权限时使用`Permission`类，避免每次调用`render_template()`时都多添加一个模板参数，可以使用 **上下文处理器** 。 **上下文处理器** 能让变量在所有模板中全局可访问。

下面的代码用来把`Permission`类加入模板上下文：

```python
# app/main/__init__.py
from ..models import Permission


@main.app_context_processor
def inject_permissions():
    return dict(Permission=Permission)
```

另外，新添加的角色和权限可在单元测试中进行测试。

**🔖 执行`git checkout 9a`签出程序的这个版本。** ⚠️ 此版本包含一个数据库迁移。

⚠️ 最好重新创建或更新开发数据库，为了赋予角色给在实现角色和权限之前创建的用户。
