---
title: python(2)——构建工具setup.py
date: 2021-01-13 10:57:27
tags:
- python
categories:
- python
---

通常我们安装依赖都是通过`pip`安装，那`pip`是怎么工具的呢，我们要把一个`package`打包发布又该如何操作呢？


<!-- more -->

通常我们需要编写一个`setup.py`文件，文件中需要提供这个`package`的依赖、版本及环境信息。

## setup.py介绍

```pyton
from setuptools import setup, find_packages

from distutils.core import setup

setup(name='Distutils',
      version='1.0',
      description='Python Distribution Utilities',
      author='Greg Ward',
      author_email='gward@python.net',
      url='https://www.python.org/sigs/distutils-sig/',
      packages=['distutils', 'distutils.command'],
     )
```

基本参数说明：

* --name: 包名称
* version: 当前版本
* license: 许可证，常用的 `MIT`,`GPL`
* url: 通常指向项目的`github`仓库
* packages: 需要打包进去的包
* package_dir: 如需要包含全部的可以这样写 `package_dir = {'': 'lib'}`
* install_requires: 需要安装的依赖 


## entry_points

`entry_points`可生成可执行文件，通常`cli`工具会使用这个参数与`click`库配合使用。

```
entry_points={
    'console_scripts': [
        'cursive = test.main:main',
    ],
}
```

上面的示例中，`cursive`是可执行的命令，`cmd:cursive_command`表示映射到`test`包`main`模块下的`main`方法上。

```python
├── test
│   ├── __init__.py
│   └── main.py
├── setup.py

# main.py
import click


@click.command()
def main():
    print("run")

# setup.py
from setuptools import setup, find_packages

from distutils.core import setup

setup(name='test',
      version='1.0',
      description='Python Distribution Utilities',
      author='Greg Ward',
      author_email='gward@python.net',
      url='https://www.python.org/sigs/distutils-sig/',
      packages=find_packages(),
      entry_points={
          'console_scripts': [
              'qwer = test.main:main',
          ],
      }
      )

```

执行`python setup.py install`之后，我们就可以在当前环境下，使用`qwer`这个命令了

```shell
$ qwer --help
Usage: qwer [OPTIONS]

Options:
  --help  Show this message and exit.
```


## 打包上传至pypi

写好了`setup.py`之后，执行打包命令

```shell
python setup.py sdist bdist_wheel
```

命令执行成功后，查看`dist`目录，生成了如下打包文件

![目录](https://pic.hupai.pro/img/20210113114708.png)


如果没有`pypi`账号，则需要前往官网注册，[pypi官网](https://pypi.org/)

注册之后，需要`twine`这个上传工具，执行`pip install twine`

安装完成后，执行`twine upload dist/*`即可

发布完成后所有人都可以通过`pip install test`安装此包