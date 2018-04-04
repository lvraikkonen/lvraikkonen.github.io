---
title: 'Flask 从入门到放弃5: 数据库操作'
date: 2017-08-28 11:16:14
tags:
  - Python
  - Flask
categories: Flask从入门到放弃
---

Flask-SQLAlchemy是一个Flask扩展，它简化了在Flask应用程序中对SQLAlchemy的使用。SQLAlchemy是一个强大的关系数据库框架，支持一些数据库后端。提供高级的ORM和底层访问数据库的本地SQL功能。

通过pip安装Flask-SQLAlchemy：

``` shell
(venv) $ pip install flask-sqlalchemy
```

![Flask-SQLAlchemy Logo](http://7xkfga.com1.z0.glb.clouddn.com/Flask-SQLAlchemy_logo.png)

对于一个Flask应用，我们需要先创建Flask应用，选择加载配置，然后创建SQLAlchemy对象时候把Flask应用传递给它作为参数。

``` python
from flask_sqlalchemy import SQLAlchemy
from config import config

db = SQLAlchemy()


def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    db.init_app(app)
    return app
```

<!-- more -->

## 配置

下面是Flask-SQLAlchemy中常用的配置值。Flask-SQLAlchemy从Flask主配置(config.py)中加载这些值。

- SQLALCHEMY_DATABASE_URI：用于数据库的连接，例如sqlite:////tmp/test.db
- SQLALCHEMY_TRACK_MODIFICATIONS：如果设置成True(默认情况)，Flask-SQLAlchemy将会追踪对象的修改并且发送信号。这需要额外的内存，如果不必要的可以禁用它。
- SQLALCHEMY_COMMIT_ON_TEARDOWN：每次request自动提交db.session.commit()

更多的配置键请参考[Flask-SQLAlchemy官方文档](http://www.pythondoc.com/flask-sqlalchemy/config.html)。

常见数据库的连接URI格式如下所示：

| Database | URI          |
|:---------|:-------------|
|Postgres  | postgresql://scott:tiger@localhost/mydatabase |
|MySQL     | mysql://scott:tiger@localhost/mydatabase |
|Oracle    | oracle://scott:tiger@127.0.0.1:1521/sidname |
|SQLite    | sqlite:////absolute/path/to/foo.db |

## 模型定义

在ORM中，模型一般是一个Python类，类中的属性对应为数据表中的列 (app/models.py)

``` python
from . import db


class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)

    def __repr__(self):
        return '<User %r>' % self.username
```

这个模型创建了两个字段，他们是类db.Column的实例，id和username。db.Column 类构造函数的第一个参数是数据库列和模型属性的类型，下面列出了一些常见的列类型以及在模型中使用的Python类型：

- Integer：普通整数，一般是32bit
- String：变长字符串
- Text：变长字符串，对较长或不限长度的字符做了优化
- Boolean：布尔值
- Date：日期
- DateTime：日期和时间

db.Column 中其余的参数指定属性的配置选项。下面列出了一些常用选项：

- primary_key：如果设置为True，这列就是表的主键
- unique：如果设置为True，这列不允许出现重复的值
- index：如果设置为True，为这列创建索引，提升查询效率
- default：为这列定义默认值

### 一对多关系

关系型数据库使用主外键关系把不同表中的行联系起来。关系可能分为：一对一、一对多、多对多等。关系使用`db.relationship()`函数表示，外键使用`db.ForeignKey`来单独声明。

``` python
class Role(db.Model):
    __tablename__ = 'roles'
    id = db.Column(db.Integer, primary_key=True)
    name = db.Column(db.String(64), unique=True)
    users = db.relationship('User', backref='role', lazy='dynamic')

    def __repr__(self):
        return '<Role %r>' % self.name


class User(db.Model):
    __tablename__ = 'users'
    id = db.Column(db.Integer, primary_key=True)
    username = db.Column(db.String(64), unique=True, index=True)
    role_id = db.Column(db.Integer, db.ForeignKey('roles.id'))

    def __repr__(self):
        return '<User %r>' % self.username
```

user表中的`db.ForeignKey('roles.id')`属性说明，user表的role_id列被定义为外键去关联roles表的id列。在role表中添加的users属性将返回与角色相关联的用户组成的列表(代表这个关系的面向对象视角)

常用的配置选项如下所示：
- backref：在关系的另一个模型中添加反向引用
- primaryjoin：明确指定两个模型之间使用的联结条件。只在模棱两可的关系中需要指定
- lazy：决定了SQLAlchemy什么时候从数据库中加载数据。可选值有 select(首次访问时按需加载)、immediate(源对象加
载后就加载)、 joined(加载记录，但使用联结)、 subquery (立即加载，但使用子查询)，
noload(永不加载)和 dynamic(不加载记录，但提供加载记录的查询)

### 多对多关系

在学习数据库时，处理多对多关系的方法是用到第三张表，也就是关系表。

## 数据库基本操作

在创建好Flask应用、配置好，创建了SQLAlchemy对象之后，就可以对定义好的数据库进行操作了，主要操作就是增删改查四项。

### 创建表

``` python
db.create_all()
```

这时候在程序目录中创建了一个.sqlite的文件，这个就是数据库文件了。

### 插入数据

``` python
user_john=User(username='john')
#添加到数据库会话
db.session.add(user_john)
#提交
db.session.commit()
```

数据库会话(db.session)和Flask的session没有什么关系，数据库会话就是关系型数据库的事务，事务能够保证数据库的一致性，提交操作使用原子方式把会话中的对象全部写入数据库，如果发生错误，可以回滚(`db.session.rollback()`)。

### 修改数据

``` python
admin_role = 'new role name'
db.session.add(admin_role)
db.session.commit()
```

### 删除数据

数据库会话也有删除操作

``` python
#删除行
db.session.delete(user_john)
db.session.commit()
```

### 查询数据

Flask-SQLAlchemy 在Model类上提供了 query 属性。访问它，会得到一个新的所有记录的查询对象。

``` python
#查询行
User.query.all()
```

使用过滤器可以配置query对象进行更精确的数据库查询:

- filter()：把过滤器添加到原查询上，返回一个新查询
- filter_by()：把等值过滤器添加到原查询上，返回一个新查询
- limit()：使用指定的值限制原查询返回的结果数量，返回一个新查询
- offset()：偏移原查询返回的结果，返回一个新查询
- order_by()：根据指定条件对原查询结果进行排序，返回一个新查询
- group_by()：根据指定条件对原查询结果进行分组，返回一个新查询

在查询上应用指定的过滤器后，通过调用all()执行查询，以列表的形式返回结果。除了all()之外，还有其他方法能触发查询执行。下面列出常用的执行查询方法：

- all()：以列表形式返回查询的所有结果
- first()：返回查询的第一个结果，如果没有结果，则返回 None
- first_or_404()：返回查询的第一个结果，如果没有结果，则终止请求，返回 404 错误响应
- get()：返回指定主键对应的行，如果没有对应的行，则返回 None
- get_or_404()：返回指定主键对应的行，如果没找到指定的主键，则终止请求，返回 404 错误响应
- count()：返回查询结果的数量
- paginate()：返回一个 Paginate 对象，它包含指定范围内的结果
