# Python 打包问题
平时Python用的比较多，编码和测试都很愉快，到了把程序分发到机器就开始头疼。通常会遇到下面几个问题：
1. 依赖模块的版本
2. 运行环境不能使用`pip`安装依赖

第一个问题其实就是需要一个自包含的Python运行环境。我们期望运行环境上安装的模块的版本和我们测试环境下是一致的。目前Python还不能像Ruby那样在同一台机器上安装同一模块的不同版本，从而也不能简单的做到自包含。

第二个问题也现实场景中也经常遇到，比如说线上机器不能连接公网，从而也无法从官方`pip`源上安装。最好的情况有个值得信任的第三方pip源可供使用，但这个pip源未必有你需要的模块或者特定的版本。另外，我们也想避免在运行环境上为了一次性执行的脚本，安装一堆模块。

解决这个问题也有多种办法，如采用Docker进行分发，采用virtualenv等。但都不怎么直接或者轻量化。

最终我们采用了[pex](https://github.com/pantsbuild/pex)来解决Python应用的分发问题。pex最终会把所有的依赖和你自己的代码打包成一个`.pex`为后缀的可执行文件。在运行环境下直接执行该文件即可。需要注意的是，pex不能用于Windows，这个确实比较遗憾。

# 生成PEX文件
下面通过一个简单的Python示例程序，来介绍一下使用pex的典型方式。这里不对pex的原理做过多说明，有兴趣的可以直接看它的文档。

我们首先创建一个Python package。创建和组织一个Python package的相关细节可以参考[The Hitchiker's Guide to Python](https://docs.python-guide.org/writing/structure/)。项目的目录结构是：
```bash
.
├── MANIFEST.in
├── hellopex
│   ├── __init__.py
│   └── hello.py
├── requirements.txt
└── setup.py
```
其中`hello.py`是程序的主要逻辑所在，内容也非常简单：
```python
#!/usr/bin/env python

import sys
from termcolor import colored

if __name__ == '__main__':
    print colored('Hello pex!', 'green', attrs=['bold'])
```
可以看出这个小项目依赖了[termcolor](https://pypi.org/project/termcolor/)，从而`requirements.txt`的内容就是
```
termcolor==1.1.0
```
好了，现在我们准备把这个小项目编程一个自包含的可执行文件。首先创建一个源代码发布包：
```bash
$ python setup.py sdist 
```
会在项目目录下面生成`dist`子目录。然后生成`.pex`文件：
```bash
$ pex -v --disable-cache -r requirements.txt hellopex -f dist -e hellopex.hello -o hello.pex
```
其中
* `-v`：运行时输出更多信息，这对调试非常有帮助
* `--disable-cache`：避免使用缓存。
* `-r requirements.txt`：指定依赖文件，该选项可以多次使用
* `-f dist` 指定本地依赖包。这里的`dist`就是`python setup.py sdist`创建出来的文件夹
* `-e hellopex.hello`：指定运行入口。这里就是`hellpex` package的`hello.py`文件
* `-o helle.pex`：输出文件
至此，`.pex`文件就生成好了，把它拷贝到想要运行的机器上执行：
```bash
$ ./hello.pex
```

# 小结
使用`pex`我们可以生成自包含的`.pex`，把依赖和程序打包在一起，方便在其他环境中运行。facebook也开源了一个类似的项目[XAR](https://github.com/facebookincubator/xar)，以后有时间了再折腾。

# 参考
* [pex](https://github.com/pantsbuild/pex)
* [Packaging a Simple Python Script with PEX](https://idle.run/simple-pex)
