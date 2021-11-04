sphinx安装与使用
#################

安装python
*******************

**以windos为例**

我选择安装的是python3.9

链接为 `python`_

.. _python: https://www.python.org/downloads/release/python-397/

下载windws安装程序（64）位，为所有用户安装，将环境变量添加到系统里。



等待安装完成，结束之后，打开cmd，输入python 查看是否正常运行。

.. figure:: ../_static/2.png
    
    正常运行python
    
安装sphinx
**************


打开cmd终端，使用

.. code-block:: bash

    pip install -U sphinx

命令进行安装

.. figure:: ../_static/3.png

    安装

安装在哪里不清楚，不管了。

.. figure:: ../_static/4.png

    安装完成页面

.. figure:: ../_static/5.png

    显示版本号

创建一个项目
*************

转到一个空的文件夹目录下。

然后输入命令

.. code-block:: bash

    sphinx-quickstart


是否需要创建隔离的文件夹，默认为不需要
.. image:: ../_static/6.png

我选择的是需要，就输入y就行了。

然后是项目名称

.. figure::　../_static/7.png

    名称

然后作者姓名，版本号（版本号可以不填，直接回车）。

.. image:: ../_static/8.png

.. figure:: ../_static/9.png

    设置语言为中文

配置结束

更多配置都在conf.py中。

生成html文件
******************

直接在项目目录里执行

.. code-block:: bash

    make html


确保目录里包含Makefile文件。

使用Markdown文件
*****************

安装 Markdown 解析器 MyST-Parser:

.. code-block:: python

    pip install --upgrade myst-parser


把 myst_parser 添加到列表 已配置的扩展 中:

.. code-block:: python
    
    extensions = ['myst_parser']


.. image:: ../_static/11.png

如果要使用不以“.md”为扩展名的 Markdown 文件，请调整 source_suffix 变量。下面的示例将配置 Sphinx 把所有扩展名为“.md”和“.txt”的文件解析为 Markdown

.. code-block:: python

    source_suffix = {
        '.rst': 'restructuredtext',
        '.txt': 'markdown',
        '.md': 'markdown',
    }


更换html主题
*************

官方提供了很多主题，也配备了教程， `主题网站`_ .

.. _主题网站: https://sphinx-themes.org/

平常见的最多的还是, `阅读文档`_ .

.. _阅读文档: https://sphinx-themes.org/sample-sites/sphinx-rtd-theme/

reStructuredText初级入门
*****************************

`官方教程`_

`私人的教程`_

.. _官方教程: https://www.sphinx-doc.org/zh_CN/master/usage/restructuredtext/index.html

.. _私人的教程: https://iridescent.ink/HowToMakeDocs/Basic/reST.html

后续再补充简明教程。

网站托管
*****************

在 `read the docs`_ 注册一个账号。

.. _read the docs: https://readthedocs.org/

**如果与github账号就可以不用注册，直接关联**

然后github新建一个公共的仓库，然后将项目里的文件全同步到仓库里。

在read the docs网站上导入工程。

.. image:: ../_static/12.png

等待构建完成。

.. figure:: ../_static/13.png

    网址

.. tip:: 这些网址都可以打开网页。

**服务器地址为美国，访问可能会比较慢。同时免费版会有一些广告**

gitee还没有使用，后续再写。

更换域名还没有测试，后续再搞。


中文搜索问题
****************

pip安装jieba，然后在conf.py里增加配置，再重新编译生成文档。

pip安装的时候可能会超时，更换 `清华源`_ 之后，顺利安装。

.. _清华源: https://mirrors.tuna.tsinghua.edu.cn/help/pypi/

这是 `教程链接`_.

.. _教程链接: https://www.shuzhiduo.com/A/MyJxQPXR5n/

.. tip:: 这是windows的解决方案，但在linux下就失效了

linux下，使用一个 `支持简体中文搜索的插件`_ 来解决问题。

.. _支持简体中文搜索的插件: https://github.com/bosbyj/sphinx.search.zh_CN

在我的服务器中

.. figure:: ../_static/14.png

    Sphinx的目录

sphinx简明教程
########################

sphinx使用时需要注意的问题
###############################

.. Attention:: 编辑完源码之后，保存，然后make html，再刷新网页，同时按下ctrl+F5重新加载网页。

.. note:: 有时候会忘记保存就编译了，这样是不会出现效果的