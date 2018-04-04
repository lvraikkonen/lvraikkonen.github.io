---
title: 'Flask 从入门到放弃6: 网站结构最佳实践'
tags:
  - Python
  - Flask
categories: Flask从入门到放弃
date: 2017-08-28 17:50:39
---


来自[狗书](https://item.jd.com/11594082.html)第七章：大型程序的结构

 ![Flask Web Development](https://covers.oreillystatic.com/images/0636920031116/lrg.jpg)

## 项目结构

下图显示多文件的Flask程序的基本结构

![FlaskProjectTree](http://7xkfga.com1.z0.glb.clouddn.com/structure.png)

<!-- more -->

这个结构有四个顶层目录：

- Flask应用一般放置在名为`app`的目录下
- `migrations`目录包含数据库迁移脚本
- 单元测试放置在`test`目录下
- `venv`目录包含Python虚拟环境

还有一些新的文件：

- `requirements.txt`列出一些依赖包，这样就可以很容易的在不同的计算机上部署一个相同的虚拟环境。
- `config.py`存储了一些配置设置。
- `manage.py`用于启动应用程序和其他应用程序任务。

## 配置选项

在这里配置应用相关的配置，例如Form用到的`SECRET_KEY`, 数据库配置的`SQLALCHEMY_DATABASE_URI`等配置信息。也可以在这里配置开发、测试和生产环境所需要的不同配置等。

 `config.py`文件：

``` python
import os
basedir = os.path.abspath(os.path.dirname(__file__))

class Config:
    SECRET_KEY = os.environ.get('SECRET_KEY') or 'hard to guess string'
    SQLALCHEMY_COMMIT_ON_TEARDOWN = True
    FLASKY_MAIL_SUBJECT_PREFIX = '[Flasky]'
    FLASKY_MAIL_SENDER = 'Flasky Admin <flasky@example.com>'
    FLASKY_ADMIN = os.environ.get('FLASKY_ADMIN')

    @staticmethod
    def init_app(app):
        pass

class DevelopmentConfig(Config):
    DEBUG = True

    MAIL_SERVER = 'smtp.googlemail.com'
    MAIL_PORT = 587
    MAIL_USE_TLS = True
    MAIL_USERNAME = os.environ.get('MAIL_USERNAME')
    MAIL_PASSWORD = os.environ.get('MAIL_PASSWORD')
    SQLALCHEMY_DATABASE_URI = os.environ.get('DEV_DATABASE_URL') or \
        'sqlite:///' + os.path.join(basedir, 'data-dev.sqlite')

class TestingConfig(Config):
    TESTING = True
    SQLALCHEMY_DATABASE_URI = os.environ.get('TEST_DATABASE_URL') or \
        'sqlite:///' + os.path.join(basedir, 'data-test.sqlite')

class ProductionConfig(Config):
    SQLALCHEMY_DATABASE_URI = os.environ.get('DATABASE_URL') or \
        'sqlite:///' + os.path.join(basedir, 'data.sqlite')

config = {
    'development': DevelopmentConfig,
    'testing': TestingConfig,
    'production': ProductionConfig,
    'default': DevelopmentConfig
}
```

## 程序包

应用程序包放置了所有应用程序代码、模板(templates)和静态文件(static)。数据库模型和电子邮件支持功能也要置入到这个包中，以`app/models.py`和`app/email.py`形式存入自己的模块当中

### 使用工厂函数创建程序实例

单个文件中开发程序，创建程序实例在全局作用域中，无法动态修改配置，对于单元测试来说很重要，必须在不同的配置环境中运行程序。

这里把创建程序实例的过程放到可显示调用的**工厂函数**中。(`app.__init__.py`)

``` python
from flask import Flask
from flask_bootstrap import Bootstrap
from flask_sqlalchemy import SQLAlchemy
from config import config

bootstrap = Bootstrap()
db = SQLAlchemy()


def create_app(config_name):
    app = Flask(__name__)
    app.config.from_object(config[config_name])
    config[config_name].init_app(app)

    bootstrap.init_app(app)
    mail.init_app(app)
    moment.init_app(app)
    db.init_app(app)

    ## route and errorhandler

    return app
```

工厂函数返回创建的Flask程序实例。`create_app()`函数就是程序的工厂函数，接收一个参数就是程序所使用的配置名。这时候，是动态获得了程序实例了，但是定义路由需要在调用工厂函数之后才能使用`app.route`装饰器定义路由。这时候就引入蓝图(Blueprint)来进行路由的定义。

### 蓝图中实现程序功能

蓝图定义的路由处于休眠状态，直到蓝图注册到应用实例上后，路由才真正成为程序的一部分，蓝图的使用对于大型程序的模块化开发提供了方便。

`app/main/__init__.py`

``` python
from flask import Blueprint

main = Blueprint('main', __name__)

from . import views, errors
```

这里定义蓝图main，程序的路由在views.py中，错误处理程序在errors.py中

在工厂函数`create_app()`中将蓝图注册到程序实例上

``` python
def create_app(config_name):
    app = Flask(__name__)
    # ...

    from .main import main as main_blueprint
    app.register_blueprint(main_blueprint)

    return app
```

## 启动脚本

在顶级文件夹中的manage.py文件用于启动应用程序 manage.py

``` python
#!/usr/bin/env python
import os
from app import create_app, db
from app.models import User, Role
from flask_script import Manager, Shell
from flask_migrate import Migrate, MigrateCommand

app = create_app(os.getenv('FLASK_CONFIG') or 'default')
manager = Manager(app)
migrate = Migrate(app, db)


def make_shell_context():
    return dict(app=app, db=db, User=User, Role=Role)

manager.add_command("shell", Shell(make_context=make_shell_context))
manager.add_command('db', MigrateCommand)


if __name__ == '__main__':
    manager.run()
```

## 需求文件

包含一个requirement.txt文件，用于记录所有依赖包以及精准的版本号，pip使用下面命令生成需求文件

``` shell
(venv) $ pip freeze > requirements.txt
```

在一个新环境中，使用下面的命令创建一个相同的环境

``` shell
(venv) $ pip install -r requirements.txt
```
