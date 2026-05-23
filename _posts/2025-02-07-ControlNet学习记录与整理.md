---
layout: post
title: "ControlNet 学习记录与整理"
date: 2025-02-07 00:00:00 +0800
tags: [ControlNet, LoRA, 深度学习, Stable Diffusion]
category: 技术分享
---

# ControlNet 学习记录与整理

ControlNet 就是用来控制构图的，LoRA就是用来控制风格的 。

ControlNet 论文：<https://arxiv.org/abs/2302.05543>

ControlNet 论文解读：<https://zhuanlan.zhihu.com/p/633491149>

ControlNet 源码：<https://github.com/lllyasviel/ControlNet>

ControlNet 源码（Gitee）：<https://gitee.com/zqcyyf/ControlNet>

huggingface：<https://huggingface.co/lllyasviel/ControlNet>

## 预处理器

- 预处理器要与模型保持一致，不然效果会不理想。

### 预处理器类型

- scribble

## 训练

1. 下载源码（上面有地址），如果很慢，建议在借助码云的导入仓库功能：<https://gitee.com/projects/import/url>，从码云上克隆会非常快；

    ```cmd
    git clone https://gitee.com/zqcyyf/ControlNet.git
    ```

2. 安装环境：

    正常是可以使用ControlNet文档中的命令：

    ```cmd
    conda env create -f environment.yaml
    conda activate control
    ```

    但，实际情况可能是下载不下来，下面分步来进行：

    ```cmd
    # 先配置源：
    conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/main/
    conda config --add channels https://mirrors.ustc.edu.cn/anaconda/pkgs/free/
    conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/conda-forge/
    conda config --add channels https://mirrors.ustc.edu.cn/anaconda/cloud/pytorch/
    conda config --set show_channel_urls yes

    # 创建 conda 虚拟环境
    conda create --name control python=3.8.5
    conda activate control

    # 安装所需的依赖
    conda install cudatoolkit=11.3 pytorch=1.12.1 torchvision=0.13.1 numpy=1.23.1 -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/main/ -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/pkgs/free/ -c https://mirrors.tuna.tsinghua.edu.cn/anaconda/cloud/pytorch/

    执行上面命令为什么会安装上个cpu版本？？？？后面使用 conda uninstall pytorch 删除后，再使用 conda install pytorch==1.12.1 torchvision==0.13.1 torchaudio==0.12.1 cudatoolkit=11.3 -c pytorch 命令安装下，看看这回安装的是什么版本。

    # 安装其他包，这里有点小插曲，在安装basicsr时它依赖tb-nightly，但源里找不到tb-nightly，导致basicsr安装失败，后来在安装basicsr前加了一条命令手动从阿里源安装tb-nightly来解决这个问题。
    pip install gradio==3.16.2                   -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install albumentations==1.3.0            -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install opencv-contrib-python==4.3.0.36  -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install opencv-python==4.3.0.36          -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install opencv-python-headless==4.3.0.36 -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install imageio==2.9.0                   -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install imageio-ffmpeg==0.4.2            -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install pytorch-lightning==1.5.0         -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install omegaconf==2.1.1                 -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install test-tube>=0.7.5                 -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install streamlit==1.12.1                -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install einops==0.3.0                    -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install transformers==4.19.2             -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install webdataset==0.2.5                -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install kornia==0.6                      -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install open_clip_torch==2.0.2           -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install invisible-watermark>=0.1.5       -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install streamlit-drawable-canvas==0.8.0 -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install torchmetrics==0.6.0              -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install timm==0.6.12                     -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install addict==2.4.0                    -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install yapf==0.32.0                     -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install prettytable==3.6.0               -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install safetensors==0.2.7               -i https://pypi.tuna.tsinghua.edu.cn/simple
    pip install tb-nightly                       -i https://mirrors.aliyun.com/pypi/simple
    pip install basicsr==1.4.2                   -i https://pypi.tuna.tsinghua.edu.cn/simple
    ```

    > File "C:\Users\username\AppData\Local\anaconda3\envs\control\lib\site-packages\gradio\queueing.py", line 298, in call_prediction
    > assert data is not None, "No event data"
    > AssertionError: No event data

    ```cmd
    pip install gradio==3.38.0
    ```

## 下载模型

在这个页面里下载模型（huggingface.co镜像站）

https://hf-mirror.com/lllyasviel/ControlNet/tree/main/models

https://hf-mirror.com/lllyasviel/ControlNet/tree/main/annotator/ckpts

https://hf-mirror.com/stable-diffusion-v1-5/stable-diffusion-v1-5/tree/main

https://hf-mirror.com/lllyasviel/ControlNet/resolve/main/models/control_sd15_canny.pth

## 好资料

- [深入浅出完整解析ControlNet核心基础知识](https://zhuanlan.zhihu.com/p/660924126)
  - 讲解的非常全面每个模型都有介绍

- [ControlNet详细入门介绍](https://zhuanlan.zhihu.com/p/18658951069)
  - 给模型进行了分类

## 参考文献

- [ControlNet安装使用](https://blog.csdn.net/HeYong83/article/details/132039807)
- [ControlNet Issues: AssertionError: No event data](https://github.com/lllyasviel/ControlNet/issues/475)
