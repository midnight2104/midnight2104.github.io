---
layout: post
title: Anaconda的简单使用
tags: deep-learning
---

##### Anaconda : 管理 Python 所用的包和环境。
#### 安装
Anaconda 实际上是一个软件发行版，它附带了 conda、Python 和 150 多个科学包及其依赖项。
[点击这里进行下载]( https://www.continuum.io/downloads )

在 Windows 上，会随 Anaconda 一起安装一批应用程序：

- Anaconda Navigator，它是用于管理环境和包的 GUI
- Anaconda Prompt 终端，它可让你使用命令行界面来管理环境和包
- Spyder，它是面向科学开发的 IDE
- Jupter Notebook,web应用，数据可视化

为了避免报错，推荐在默认环境下更新所有的包。
打开 Anaconda Prompt （或者 Mac 下的终端），键入：


```
conda upgrade --all
```

并在提示是否更新的时候输入 y（Yes）以便让更新继续。初次安装下的软件包版本一般都比较老旧，因此提前更新可以避免未来不必要的问题。




#### 管理包
要安装包，请在终端中键入
``` 
conda install package_name 
```

例如，要安装 numpy，请键入 conda install numpy。

你还可以同时安装多个包。类似 conda install numpy scipy pandas 的命令会同时安装所有这些包。还可以通过添加版本号（例如 conda install numpy=1.10）来指定所需的包版本。

Conda 还会自动为你安装依赖项。例如，scipy 依赖于 numpy，因为它使用并需要 numpy。如果你只安装 scipy (conda install scipy)，则 conda 还会安装 numpy（如果尚未安装的话）。

要卸载包，请使用 
```
conda remove package_name
```

要更新包，请使用
```
conda update package_name
```

如果想更新环境中的所有包，请使用 
```
conda update --all
```

最后，要列出已安装的包，请使用
```
conda list。
```
如果不知道要找的包的确切名称，可以尝试使用 
```
conda search search_term 
```
进行搜索。

例如，我知道我想安装 Beautiful Soup，但我不清楚确切的包名称。因此，我尝试执行 conda search beautifulsoup。


#### 管理环境
如前所述，你可以使用 conda 创建环境以隔离项目。
要创建环境，请在终端中使用 
```
conda create -n env_name list of packages
```
在这里，-n env_name 设置环境的名称（-n 是指名称），而 list of packages 是要安装在环境中的包的列表。例如，要创建名为 my_env 的环境并在其中安装 numpy，请键入``` conda create -n my_env numpy```

创建环境时，可以指定要安装在环境中的 Python 版本。这在你同时使用 Python 2.x 和 Python 3.x 中的代码时很有用。

要创建具有特定 Python 版本的环境，请键入类似于``` conda create -n py3 python=3 ```或 ```conda create -n py2 python=2``` 的命令。这些命令将分别安装 Python 3 和 Python 2 的最新版本。

要安装特定版本（例如 Python 3.3），请使用 ``` conda create -n py python=3.3 ```

进入环境
创建了环境后，在 OSX/Linux 上使用``` source activate my_env``` 进入环境。
请使用 

```activate my_env```  在 Windows 上，进入环境 


```conda install package_name```在环境中安装包

不过，这次你安装的特定包仅在你进入环境后才可用。要离开环境，请键入``` source deactivate```（在 OSX/Linux 上）。

```deactivate```  在 Windows 上,离开环境

``` conda env export > environment.yaml  ``` 环境导出,将包保存为 YAML

``` conda env create -f environment.yaml ``` 环境导入

```conda env list ``` 列出你创建的所有环境

``` conda env remove -n env_name ```  删除指定的环境


对于不使用 conda 的用户，可以使用 
```py
pip freeze > requirements.txt      #导出
pip install -r requirements.txt    #导入

```

