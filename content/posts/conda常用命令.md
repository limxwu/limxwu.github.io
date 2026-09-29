---
title: conda常用命令
date: 2025-09-08 12:00:00
---

更新conda

    conda update conda

创建虚拟环境

    conda create -n env_name python=3.8

查看有哪些虚拟环境

    conda env list

激活虚拟环境

    conda activate env_name

删除虚拟环境

执行以下命令可以将该指定虚拟环境及其中所安装的包都删除：

    conda remove --name env_name --all

如果只删除虚拟环境中的某个或者某些包则是：

    conda remove --name env_name  package_name