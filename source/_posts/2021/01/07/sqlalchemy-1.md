---
title: Python(1) SQLAlchemy库——连接与基本ORM操作
date: 2021-01-07 17:55:55
tags:
- python
- sqlalchemy
categories:
- python
---

`SQLAlchemy`是`Python`公认最好的`ORM`库，它同时也是最好的数据库操作库。

<!-- more -->

## 连接

`SQLAlchemy`本身是不能直接连接各种数据库的，所以需要安装对应的驱动库。


```python
# mysql
# pip install python-mysql
engine = create_engine('mysql+mysqldb://user:pwd@localhost/foo')
# pip install pymysql 
engine = create_engine('mysql+pymysql://user:pwd@localhost/foo')

# postgresql
# pip install psycopg2
engine = create_engine('postgresql://user:pwd@localhost/foo')

# sqlite
# 自带驱动，db文件路径
engine = create_engine('sqlite:///foo.db')

# Microsoft SQL Server
# pip install pyodbc
engine = create_engine('mssql+pyodbc://user:pwd@foo')

# oracle
# pip install cx_oracle
engine = create_engine('oracle://user:pwd@127.0.0.1:1521/foo')

# clickhouse
# pip install clickhouse-sqlalchemy
# pip install clickhouse-driver
engine = create_engine('clickhouse://user:pwd@127.0.0.1:8123/foo)

```


## ORM Model定义

通过继承`Base`定义的`User`类与数据库表`users`完成的`Mapping`，`id`对应数据库主键，其余`name`等字段也与表中的字段一一对应。

```python
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy import Column, Integer, String

Base = declarative_base()

class User(Base):
    # 数据库表名
    __tablename__ = 'users' 
    # 主键
    id = Column(Integer, primary_key=True)

    name = Column(String)
    fullname = Column(String)
    nickname = Column(String)

    def __repr__(self):
      return "<User(name='%s', fullname='%s', nickname='%s')>" % (
                            self.name, self.fullname, self.nickname)
```

## 创建Session

```python
from sqlalchemy import create_engine
engine = create_engine('sqlite:///:memory:', echo=True)

from sqlalchemy.orm import sessionmaker
Session = sessionmaker(bind=engine)
```

## 添加记录

```python
# add
ed_user = User(name='ed', fullname='Ed Jones', nickname='edsnickname')
session.add(ed_user)

# batch add 
session.add_all([
    User(name='wendy', fullname='Wendy Williams', nickname='windy'),
    User(name='mary', fullname='Mary Contrary', nickname='mary'),
    User(name='fred', fullname='Fred Flintstone', nickname='freddy')])
```

## 更新

直接修改`user`对象的`nickname`属性

```python
>>> ed_user.nickname = 'eddie'
```

此时`session`中记录了变更

```python
>>> session.dirty
IdentitySet([<User(name='ed', fullname='Ed Jones', nickname='eddie')>])
```

执行提交

```python
>>> session.commit()
```

## 删除

```python
>>> session.delete(jack)
SQL>>> session.query(User).filter_by(name='jack').count()
0
```


## 查询

基本查询，`filter`中写入`where`条件

```python
>>> our_user = session.query(User).filter_by(name='ed').first() 
>>> our_user
<User(name='ed', fullname='Ed Jones', nickname='edsnickname')>

# __eq__
query.filter(User.name == 'ed')
# ___ne__
query.filter(User.name != 'ed')
# like
query.filter(User.name.like('%ed%'))
# in
query.filter(User.name.in_(['ed', 'wendy', 'jack']))
# not in
query.filter(~User.name.in_(['ed', 'wendy', 'jack']))
# and
query.filter(and_(User.name == 'ed', User.fullname == 'Ed Jones'))
# or
query.filter(or_(User.name == 'ed', User.name == 'wendy'))
# limit offset
query(User).order_by(User.id)[1:3]
```

排序，`order_by`任意字段

```python
>>> for instance in session.query(User).order_by(User.id):
...     print(instance.name, instance.fullname)
ed Ed Jones
wendy Wendy Williams
mary Mary Contrary
fred Fred Flintstone
```


## group by

```python
>>> from sqlalchemy import func
SQL>>> session.query(func.count(User.name), User.name).group_by(User.name).all()
[(1, u'ed'), (1, u'fred'), (1, u'mary'), (1, u'wendy')]
```