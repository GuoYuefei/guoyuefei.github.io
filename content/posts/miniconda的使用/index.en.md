---
title: "Using Miniconda"
date: 2021-06-18
draft: false
tags: ["python"]
categories: ["Tech"]
description: "Basic usage of Miniconda"
---

# Using Miniconda

Reason: Anaconda includes most packages, especially for artificial intelligence, but takes up too much space. Miniconda is more lightweight - install only the packages you need. Following the principle of "use only what you need," I chose to use Miniconda.

## Installation

(Skipped...)

## Creating Virtual Environments

```shell
conda create -n python_3.9 python=3.9

conda create -n env_name python=version
```

## Managing Environments

```shell
# Switch to the newly created python_3.9 environment
conda activate python_3.9

# Delete environment (think twice before doing this)
conda remove -n python_3.9 --all

# Exit current environment
conda deactivate

# Return to default environment
conda activate base
```

## Installing Packages

```shell
conda install package_name
```

## Using Requirements Files

pip can use requirements files for quick installation and can also generate them. pip usage specifications:

```shell
pip freeze > requirements.txt
pip install -r requirements.txt
```

conda can install from this file:

```shell
conda install --yes --file requirements.txt
```

## Exporting and Importing Environments

conda can completely export environments to .yml files:

```shell
conda env export > freeze.yml
```

Then create conda environments directly from exported files:

```shell
conda env create -f freeze.yml
```