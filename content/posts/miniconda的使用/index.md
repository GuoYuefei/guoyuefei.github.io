---
title: "miniconda的使用"
date: 2021-06-18
draft: false
tags: ["python"]
categories: ["技术"]
keywords: ["miniconda", "python"]
description: "简单使用miniconda"
---

# miniconda的使用

缘由： Anaconda包含了大多数包，特别是人工智能方面，但是占据空间太大。Miniconda比较轻量级，需要什么包安装什么包。本着想用多少就用多少的原则，本人选择使用Miniconda使用。

## 安装

略。。。

## 创建虚拟环境

```shell
conda create -n python_3.9 python=3.9

conda create -n env_name python=version
```

## 切换环境

```shell
# 切换到刚刚创建的pythone_3.9环境
conda activate python_3.9

# 删除环境 三思
conda remove -n python_3.9 --all

# 退出
conda deactivate

# 回到默认环境
conda activate base
```

## 安装包

```shell
conda install 包名
```



pip可以使用requirements文件来快速安装，当然也可以生成requirements文件。pip使用规范如下：

```shell
pip freeze > requirements.txt
pip install -r requirements.txt
```

conda 可以使用这个文件来安装：

```shell
conda install --yes --file requirements.txt
```



conda也可以完全把环境导出，导出成.yml文件

```shell
conda env export > freeze.yml
```

然后通过导出的文件直接创建conda环境

```shell
conda env create -f freeze.yml
```

