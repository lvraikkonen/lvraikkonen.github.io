---
title: 'Flask 从入门到放弃4: Web表单'
date: 2017-08-15 16:08:46
tags:
  - Python
  - Flask
categories: Flask从入门到放弃
---


HTML表单用于搜集不同类型的用户输入，是Web应用和用户交互的一种HTML元素。关于表单的基础知识可以去 [W3School HTML 表单](http://www.w3school.com.cn/html/html_forms.asp) 去复习一下。

先歪个楼回顾一下HTTP请求的基础知识

> [HTTP 方法：GET 对比 POST](http://www.w3school.com.cn/tags/html_ref_httpmethods.asp)

Flask中的request对象可以存储来自客户端的所有信息，其中可以通过 `request.form` 来获得POST请求所提交的表单数据。在处理表单的时候，像构建表单、表单数据验证是重复并且很繁琐的。可以使用 [Flask-WTF](https://flask-wtf.readthedocs.io/en/stable/) 来简化相关操作。

![Flask-WTF_logo](http://7xkfga.com1.z0.glb.clouddn.com/Flask-wtf_logo.png)

``` shell
(venv) $ pip install flask-wtf
```

Flask-WTF是一个集成了WTForms的Flask扩展，使用它你可以在python文件里创建表单类，然后在HTML使用它提供的函数渲染表单。

<!-- more -->

## 创建Flask-WTF表单

### 跨站请求伪造保护CSRF

Flask-WTF默认支持CSRF（跨站请求伪造）保护，只需要在程序中设置一个密钥。Flask-WTF使用这个密钥生成加密令牌，再用令牌验证表单中数据的真伪。

``` python
app = Flask(__name__)
app.config['SECRET_KEY'] = 'DontTellAnyone'
```

### 表单类

从Flask-WTF导入FlaskForm类，再从WTForms导入表单字段和验证函数

``` python
from flask_wtf import FlaskForm
from wtforms import StringField, PasswordField, SubmitField
from wtforms.validators import DataRequired, Length, Email, AnyOf
```

每个表单都用一个继承自FlaskForm的类表示，每个字段都用一个对象表示，每个对象可以附加多个验证函数。常见的验证函数有Required()，Length()，Email()等。 下面创建自己的表单类：

``` python
class LoginForm(FlaskForm):
    email = StringField(u'邮箱', validators=[DataRequired(message=u'邮箱不能为空'), Length(1, 64), Email(message=u'请输入有效的邮箱地址，比如：username@domain.com')])
    password = PasswordField(u'密码', validators=[DataRequired(message=u'密码不能为空'), Length(min=5, max=13), AnyOf(['secret', 'password'])])
    submit = SubmitField(u'登录')
```

自定义的表单类里面包含了三个元素：email文本框、password密码框和提交按钮。其中文本框和密码框引入了一些验证函数。

### 表单渲染

在视图函数中引入自定义的表单类的实例，然后在返回模版的时候传入这个实例：

``` python
@app.route('/login', methods=['GET', 'POST'])
def login():
    form = LoginForm()
    if form.validate_on_submit():
        email = form.email.data
        password = form.password.data
        print "email: %s password: %s" % (email, password)
        print 'Form Successfully Submitted!'
        flash(u'登录成功，欢迎回来！', 'info')
    return render_template('wtf.html', form=form)
```

在模板中使用下面方式渲染模版：(wtf.html)

``` html
<form class="form" method="POST">
    {{ form.hidden_tag() }}
    {{ form.email.label }}{{ form.email() }}
    {{ form.password.label }}{{ form.password() }}
    {{ form.submit() }}
</form>
```

### 使用Bootstrap渲染表单

上篇渲染模版，提到了使用Flask-Bootstrap提供比较美观的模版，现在就用这种方式渲染表单。

wtf.html模版首先继承bootstrap的基模版，然后再引入bootstrap对wtf支持的文件。

``` html
{% extends "bootstrap/base.html" %}
{% import 'bootstrap/wtf.html' as wtf %}

{% block title %}
WTForm extends Bootstrap
{% endblock %}

{% block content %}
<form class="form" method="POST">
    <dl>
        {{ form.csrf_token }}
        {{ wtf.form_field(form.email) }}
        {{ wtf.form_field(form.password) }}
        <button type="submit" class="btn btn-primary">Sign in</button>
    </dl>

</form>
{% endblock %}
```

到这里，一个简单的表单就已经创建好了，当用户第一次访问到这个页面的时候，服务器接收到一个GET方法请求，validate_on_submit() 返回False，if分支内的内容会跳过；当用户通过POST方法来提交请求时，validate_on_submit()调用Required()来验证name属性，如验证通过，if内的逻辑会被执行。而模板内容最后会被渲染到页面上。

## 处理表单数据

### 表单数据验证

当点击了submit提交按钮的时候，`form.validate_on_submit()` 方法会判断：

1. 通过is_submitted()来判断是否提交了表单
2. 通过WTForms提供的validate()验证方法验证表单数据是否符合规则。

当然，在自定义的表单类中也可以写自己的验证方法，比如从数据库中查找用户名是否被占用：(`class LoginForm`)

``` python
def validate_username(self, field):
    if User.query.filter_by(username=field.data).first():
        raise ValidationError(u'用户名已被注册，换一个吧。')
```

在表单数据验证通过后，使用 form.<NAME>.data来访问表单的单个值

### 存储表单数据

当接收POST请求的时候，可以从form.<NAME>.data中获取数据，请求结束后数据就丢失了，这时候可以使用session来存储起来，提供给以后的请求使用，在下面的代码中，就把POST提交之后的

``` python
form = NameForm()
if form.validate_on_submit():
    old_name = session.get('name')
    if old_name is not None and old_name != form.name.data:
        flash("Looks like you have changed your name!")
    session['name'] = form.name.data
    return redirect(url_for('hello_somebody'))
return render_template('userform_index.html', form=form, name=session.get('name'))
```
