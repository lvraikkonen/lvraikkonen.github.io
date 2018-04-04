---
title: Anaconda解决python包管理与环境管理
date: 2017-06-08 15:24:29
tags:
  - Anaconda
  - Python

---



Anaconda是一个用于科学计算的Python发行版，支持 Linux, Mac, Windows系统，提供了包管理与环境管理的功能，可以很方便地解决多版本python并存、切换以及各种第三方包安装问题。

conda可以理解为一个工具，也是一个可执行命令，其核心功能是包管理与环境管理

- 提供包管理，功能类似于 pip，Windows 平台安装第三方包经常失败的场景得以解决。
- 提供虚拟环境管理，功能类似于 virtualenv，解决了多版本Python并存问题

Anaconda具有跨平台、包管理、环境管理的特点，因此很适合快速在新的机器上部署Python环境。

Anaconda的下载页参见 [官网下载](https://www.continuum.io/downloads)

<!-- more -->

Anaconda安装成功之后，可以检查所安装的版本

``` bash
conda --version
```

由于不可名状的原因，需要修改其包管理镜像为国内源


``` bash
conda config --add channels https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/
conda config --set show_channel_urls yes
```

## Conda的环境管理

Conda的环境管理功能允许我们同时安装若干不同版本的Python，并能自由切换。对于上述安装过程，假设我们采用的是Python 2.7对应的安装包，那么Python 2.7就是默认的环境

现在，创建一个自定义的Python环境

``` bash
conda create -n py27 python=2.7
```

其中py27是新添加环境的名字，可以自定义修改。

之后通过activate py27和deactivate py27命令激活、退出该环境

``` bash
activate py27
```

现在把创建的环境都列出来，其中当前使用的环境前面用*号标注

``` bash
conda info --envs
```

![Change&ShowEnvs](http://7xkfga.com1.z0.glb.clouddn.com/changeEnv.png)

``` bash
# 创建一个名为py3的环境，指定Python版本是3.4（不用管是3.4.x，conda会为我们自动寻找3.4.x中的最新版本）
conda create --name py3 python=3.4
 
# 安装好后，使用activate激活某个环境
activate py3 # for Windows
source activate py3 # for Linux & Mac
# 激活后，会发现terminal输入的地方多了py3的字样，实际上，此时系统做的事情就是把默认2.7环境从PATH中去除，再把3.4对应的命令加入PATH
 
# 此时，再次输入
python --version
# 可以得到`Python 3.4.5 :: Anaconda 4.1.1 (64-bit)`，即系统已经切换到了3.4的环境
 
# 如果想返回默认的python 2.7环境，运行
deactivate py3 # for Windows
source deactivate py3 # for Linux & Mac
 
# 删除一个已有的环境
conda remove --name py3 --all
```


## Conda的包管理

现在要使用conda来管理包了，以前常用的是Python的pip包管理工具。

``` bash
# 安装scipy
conda install scipy
# conda会从从远程搜索scipy的相关信息和依赖项目，对于python 3.4，conda会同时安装numpy和mkl（运算加速的库）
 
# 查看已经安装的packages
conda list
# 最新版的conda是从site-packages文件夹中搜索已经安装的包，不依赖于pip，因此可以显示出通过各种方式安装的包
```


### 实例：让Python2和3在Jupyter Notebook中共存

多版本的Python或者R等语言，在Jupyter中被称作kernel。

如果这个Python版本已经存在（比如我们刚才添加的py27环境），那么你可以直接为这个环境安装 `ipykernel`包

``` bash
conda install -n py27 ipykernel
```

note:

> -n 后面的名字为所要安装到的环境名


然后激活这个环境

``` bash
python -m ipykernel install --user
```

如果所需版本并不是已有的环境，可以直接在创建环境时便为其预装 `ipykernel`。

``` bash
conda create -n py27 python=2.7 ipykernel
```

打开jupyter notebook

``` bash
jupyter notebook
```

![jupyterNotebook](http://7xkfga.com1.z0.glb.clouddn.com/jupyterNotebook.png)



## 最后

两个字：**省心**

附： [conda cheat sheet](http://conda.pydata.org/docs/_downloads/conda-cheatsheet.pdf)