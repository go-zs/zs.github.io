---
title: Python SQLAlchemy库——(2)create_engine分析
date: 2021-01-09 10:03:04
tags:
- python
- sqlalchemy
categories:
- python
---

`engine`是`SQLAlchemy`的核心，是`SQLAlchemy`实现对各种数据库统一操作的基础。

<!-- more -->

`create_engine`是`engine`的工厂函数，支持`strategy`参数使用指定的`Strategy类`进行构造。

```python
default_strategy = "plain"

def create_engine(*args, **kwargs):
    """省略"""
    strategy = kwargs.pop("strategy", default_strategy)
    strategy = strategies.strategies[strategy]
    return strategy.create(*args, **kwargs)
```

`EngineStrategy`是定义了`Strategy类`的行为，需要实现`create`方法。

```python
strategies = {}

class EngineStrategy(object):
    """An adaptor that processes input arguments and produces an Engine.

    Provides a ``create`` method that receives input arguments and
    produces an instance of base.Engine or a subclass.

    """

    def __init__(self):
        strategies[self.name] = self

    def create(self, *args, **kwargs):
        """Given arguments, returns a new Engine instance."""

        raise NotImplementedError()
```

`DefaultEngineStrategy`是`Strategy类`默认基类。

```python
class DefaultEngineStrategy(EngineStrategy):
    """Base class for built-in strategies."""

    def create(self, name_or_url, **kwargs):
        # 解析uri
        u = url.make_url(name_or_url)

        plugins = u._instantiate_plugins(kwargs)

        u.query.pop("plugin", None)
        kwargs.pop("plugins", None)

        entrypoint = u._get_entrypoint()
        dialect_cls = entrypoint.get_dialect_cls(u)

        ......

        return engine

    def _get_entrypoint(self):
        """Return the "entry point" dialect class.

        This is normally the dialect itself except in the case when the
        returned class implements the get_dialect_cls() method.

        """
        if "+" not in self.drivername:
            name = self.drivername
        else:
            name = self.drivername.replace("+", ".")
        cls = registry.load(name)
        # check for legacy dialects that
        # would return a module with 'dialect' as the
        # actual class
        if (
            hasattr(cls, "dialect")
            and isinstance(cls.dialect, type)
            and issubclass(cls.dialect, Dialect)
        ):
            return cls.dialect
        else:
            return cls
```

`make_url`使用正则的方式解析`uri`。

`_get_entrypoint`是通过数据库类型和驱动类型来注册插件，返回`dialect`类。这里需要再看下插件是怎么实现的。


首先`registry`是一个全局`PluginLoader`类的实例。`load`方法用`pkg_resources`过滤出`name`匹配的包。

```python
registry = util.PluginLoader("sqlalchemy.dialects", auto_fn=_auto_fn)

class PluginLoader(object):
    def __init__(self, group, auto_fn=None):
        self.group = group
        self.impls = {}
        self.auto_fn = auto_fn

    def clear(self):
        self.impls.clear()

    def load(self, name):
        if name in self.impls:
            return self.impls[name]()

        if self.auto_fn:
            loader = self.auto_fn(name)
            if loader:
                self.impls[name] = loader
                return loader()

        try:
            import pkg_resources
        except ImportError:
            pass
        else:
            # 搜索所有的package
            for impl in pkg_resources.iter_entry_points(self.group, name):
                self.impls[name] = impl.load
                return impl.load()

        raise exc.NoSuchModuleError(
            "Can't load plugin: %s:%s" % (self.group, name)
        )

    def register(self, name, modulepath, objname):
        def load():
            mod = compat.import_(modulepath)
            for token in modulepath.split(".")[1:]:
                mod = getattr(mod, token)
            return getattr(mod, objname)

        self.impls[name] = load
```

如果一个插件在`setup.py`中注册了`sqlalchemy.dialects`，就能被`pkg_resources`包找到。

```python
# clickhouse-sqlalchemy setup.py
setup(
    name='clickhouse-sqlalchemy',
    version=read_version(),
    ...

    # Registering `clickhouse` as dialect. 动态发现服务和插件
    entry_points={
        'sqlalchemy.dialects': dialects
    },

    ...
)
```



`dialect`类是一个抽象或者接口（并没有用abc包定义，而是用`NotImplementedError`方式定义），定义了插件需要实现的方法

```python
class Dialect(object):
    """Define the behavior of a specific database and DB-API combination."""
    
```

`DefaultDialect`实现是`SQLAlchemy`中`Dialect`类默认的实现，它一般作为第三方数据库扩展的父类存在。

