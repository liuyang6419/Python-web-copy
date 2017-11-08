# 文章评论

本章将实现简单的用户评论功能。


## 评论模型

评论属于某篇博客文章，从`posts`表到`comments`表是一对多关系。使用这 个关系可以获取某篇特定博客文章的评论列表。

`users`和`comments`表也是一对多关系，通过这个关系可以获取用户发表的所有评论，还能间接知道用户发表了多少篇评论。用户发表的评论数量可以显示在用户资料页中。

下面时`Comment`模型定义：

```python
class Comment(db.Model):
    body = pw.TextField(null=True)
    body_html = pw.TextField(null=True)
    timestamp = pw.DateTimeField(index=True, default=datetime.utcnow)
    disabled = pw.BooleanField(null=True, default=False)
    author = pw.ForeignKeyField(User, related_name='comments', null=True)
    post = pw.ForeignKeyField(Post, related_name='comments', null=True)

    @require_instance
    def update_body_html(self):
        allowed_tags = ['a', 'abbr', 'acronym', 'b', 'code', 'em', 'i',
                        'strong']
        self.__class__.update(body_html=bleach.linkify(bleach.clean(
            markdown(self.body, output_format='html'),
            tags=allowed_tags, strip=True))).where(self._pk_expr()).execute()

    class Meta:
        db_table = 'comments'
```

`Comment`模型的属性几乎和`Post`模型一样，不过多了一个`disabled`字段。这是个布尔值字段，协管员通过这个字段查禁不当评论。和`Post`模型一样，评论也定义了一个`update_body_html`方法，用来更新`body_html`字段，其中 允许使用的HTML标签更严格，要删除与段落相关的标签，只留下格式化字符的标签。

接着在`Post`中定义`comments_timeline`方法，来返回文章实例的评论列表：

```python
class Post(db.Model):
    # ...
    def comments_timeline(self, order='asc'):
        if order == 'asc':
            order = Comment.timestamp.asc()
        else:
            order = Comment.timestamp.desc()

        return (Comment.select(Comment, User)
                .join(User, on=(Comment.author == User.id))
                .where(Comment.post == self.id)
                .order_by(order))
```


## 提交和显示评论

评论显示在单篇博客文章页面中，在这个页面中还要有一个提交评论的表单。

评论输入表单示例：

```python
# app/main/forms.py
class CommentForm(FlaskForm):
    body = StringField('Enter your comment', validators=[Required()])
    submit = SubmitField('Submit')
```

下面是为了支持评论而更新的`/post/<int:id>`路由：

```python
# app/main/views.py
@main.route('/post/<int:id>', methods=['GET', 'POST'])
def post(id):
    post_query = Post.select()
    post = futils.get_object_or_404(post_query, (Post.id == id))
    form = CommentForm()
    if form.validate_on_submit():
        comment = Comment(body=form.body.data,
                          post=post,
                          author=current_user._get_current_object())
        comment.save()
        comment.update_body_html()
        flash('Your comment has been published.')
        return redirect(url_for('.post', id=post.id, page=-1))
    page = request.args.get('page', 1, type=int)
    if page == -1:
        page = ((post.comments.count() - 1) //
                current_app.config['FLASKR_COMMENTS_PER_PAGE'] + 1)
    pagination = Pagination(post.comments_timeline(),
                            current_app.config['FLASKR_COMMENTS_PER_PAGE'],
                            page,
                            check_bounds=False)
    comments = pagination.items
    return render_template('post.html', posts=[post], form=form,
                           comments=comments, pagination=pagination)
```

评论按照时间戳顺序排列，新评论显示在列表的底部。提交评论后，请求结果是一个重定向，转回之前的URL，但是在`url_for()`函数的参数中把`page`设为`-1`，这个页数 用来请求评论的最后一页，所以刚提交的评论才会出现在页面中。程序从查询字符串中获取页数，发现值为`-1`时，会计算评论的总量和总页数，得出真正要显示的页数。

评论的渲染过程在新模板`_comments.html`中进行，类似于`_posts.html`，但使用的 CSS 类不同。`_comments.html`模板要引入`post.html`中，放在文章正文下方，后面再显示分页导航。

接着在首页和资料页中加上指向评论页面的链接：

```django
{# app/templates/_posts.html #}
<a href="{{ url_for('.post', id=post.id) }}#comments">
    <span class="label label-primary">{{ post.comments.count() }} Comments</span>
</a>
```

分页导航所用的宏也要做些改动。评论的分页导航链接也要加上`#comments`片段，因此在`post.html`模板中调用宏时，传入片段参数。

**🔖 执行`git checkout 13a`签出程序的这个版本。**


## 管理评论

为了管理评论，要在导航条中添加一个链接，具有权限的用户才能看到。

示例如下：

```django
{# app/templates/base.html #}
{% if current_user.can(Permission.MODERATE_COMMENTS) %}
    <li><a href="{{ url_for('main.moderate') }}">Moderate Comments</a></li>
{% endif %}
```

管理页面在同一个列表中显示全部文章的评论，最近发布的评论会显示在前面。每篇评论的下方都会显示一个按钮，用来切换`disabled`属性的值。

管理评论的路由定义如下：

```python
# app/main/views.py
@main.route('/moderate')
@login_required
@permission_required(Permission.MODERATE_COMMENTS)
def moderate():
    pagination = Pagination(
        Comment.timeline(),
        current_app.config['FLASKR_COMMENTS_PER_PAGE'],
        check_bounds=False)
    comments = pagination.items
    return render_template('moderate.html', comments=comments,
                           pagination=pagination, page=pagination.page)
```

下面是评论管理页面的模板：

```django
{# app/templates/moderate.html #}
{% extends "base.html" %}
{% import "_macros.html" as macros %}

{% block title %}Flaskr - Comment Moderation{% endblock %}

{% block page_content %}
    <div class="page-header">
        <h1>Comment Moderation</h1>
    </div>
    {% set moderate = True %}
    {% include '_comments.html' %}
    {% if pagination and pagination.pages > 1 %}
        <div class="pagination">
            {{ macros.pagination_widget(pagination, '.moderate') }}
        </div>
    {% endif %}
{% endblock %}
```

这个模板将渲染评论的工作交给`_comments.html`模板完成，但把控制权交给从属模板之前，会使用Jinja2提供的`set`指令定义一个模板变量`moderate`，并将其值设为`True`。这个变量用在`_comments.html`模板中，决定是否渲染评论管理功能。

`_comments.html`模板中显示评论正文的部分要做两方面修改。对于普通用户（没设定`moderate`变量），不显示标记为有问题的评论。对于协管员（`moderate`设为`True`），不管评论是否被标记为有问题，都要显示，而且在正文下方还要显示一个用来切换状态的按钮。

下面是渲染评论正文的模板：

```django
{# app/templates/_comments.html #}
<div class="comment-body">
    {% if comment.disabled %}
        <p><i>This comment has been disabled by a moderator.</i></p>
    {% endif %}
    {% if moderate or not comment.disabled %}
        {% if comment.body_html %}
            {{ comment.body_html | safe }}
        {% else %}
            {{ comment.body }}
        {% endif %}
    {% endif %}
</div>
{% if moderate %}
    <br>
    {% if comment.disabled %}
        <a class="btn btn-info btn-xs"
           href="{{ url_for('.moderate_enable', id=comment.id, page=page) }}">Enable</a>
    {% else %}
        <a class="btn btn-danger btn-xs"
           href="{{ url_for('.moderate_disable', id=comment.id, page=page) }}">Disable</a>
    {% endif %}
{% endif %}
```

改动之后，用户将看到一个关于有问题评论的简短提示。协管员既能看到这个提示，也能看到评论的正文。在每篇评论的下方，协管员还能看到一个按钮，用来切换评论的状态。点击按钮后会触发两个新路由中的一个，但具体触发哪一个取决于 协管员要把评论设为什么状态。

下面是评论管理路由的定义：

```python
# app/main/views.py
@main.route('/moderate/enable/<int:id>')
@login_required
@permission_required(Permission.MODERATE_COMMENTS)
def moderate_enable(id):
    comment = futils.get_object_or_404(Comment.select(),
                                       (Comment.id == id))
    comment.disabled = False
    comment.save()
    return redirect(url_for('.moderate',
                            page=request.args.get('page', 1, type=int)))


@main.route('/moderate/disable/<int:id>')
@login_required
@permission_required(Permission.MODERATE_COMMENTS)
def moderate_disable(id):
    comment = futils.get_object_or_404(Comment.select(),
                                       (Comment.id == id))
    comment.disabled = True
    comment.save()
    return redirect(url_for('.moderate',
                            page=request.args.get('page', 1, type=int)))
```

启用路由和禁用路由先加载评论对象，把`disabled`字段设为正确的值，再把评论对象写入数据库。最后，重定向到评论管理页面，如果查询字符串中指定了`page`参数，会将其传入重定向操作。`_comments.html`模板中的按钮指定了`page`参数，重定向后会返回之前的页面。

**🔖 执行`git checkout 13b`签出程序的这个版本。**
