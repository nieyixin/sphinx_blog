## 使用sphinx搭建blog

### 安装sphinx

安装sphinx以及依赖  
```
pip3 install sphinx
```

安装其他软件包

```
pip3 install sphinx-autobuild # 热部署，改动能够实时更新在页面上
pip3 install sphinx_red_theme # 主题
pip3 install recommonmark # 插件，支持markdown
pip3 install sphinx_markdown_tables
```

### 快速开始

创建并进入sphinx_blog目录，执行命令

```
sphinx-quickstart
```

窗口会询问你是否选择独立的源文件和构建目录，这里选择y

然后会要求你输入项目名称、作者以及版本信息，这里直接填入即可，后续可在配置文件中修改

项目创建完成，目录结构如下所示

![项目结构](asserts/img/project_structure.png "Project Structure")

其中Makefile是构建脚本，这里不用过多关心；make.bat用于Windows系统下的构建；build目录存放构建产物，执行命令`make html`会在build目录下生成html文件，通过浏览器打开html文件，可以看到sphinx生成的网页

#### 修改配置文件 
修改配置文件conf.py，配置插件和主题：

```
extensions = ['recommonmark', 'sphinx_markdown_tables']

html_theme = 'sphinx_rtd_theme'
```

#### 启动 

我们选择使用autobuild进行热部署，这样我们修改了配置文件或者文本文件可以实时更新，执行命令
```
sphinx_autobuild source build/html --host=<host> --port=<port>
```
其中host和port不用指定，默认为127.0.0.1地址和8080端口

然后访问http://ip:port即可看到sphinx的默认页面


### 编写文档

#### 编写index.rst

通过index.rst编写文档目录树结构，能够帮助从主页面逐级索引到目标文档

例如，主页面有如下的目录结构：  

Contents:  
|  
|--content1.md  
|  
|--menu   
&nbsp;&nbsp;&nbsp;|  
&nbsp;&nbsp;&nbsp;|--content2.md  

主目录下有一篇content1.md和二级目录menu，menu下有一篇content2.md

所以我们在source/index.rst编辑目录树：
```
.. toctree::
   :maxdepth: 2
   :caption: Contents:
   
   content1
   menu/index
```
表示主目录下会有content1文档和menu这个二级目录

再编辑source/menu/index.rst:
```
.. toctree::
   :maxdepth: 2
   :caption: menu
   
   content2
```
表示menu二级目录下有content2这篇文档

#### 编写文档
创建source/content1.md、source/menu/content2.md，编辑并保存。由于打开了热部署，保存时，页面就会实时刷新

![示例页面](asserts/img/demo_page.png "Demo Page")
