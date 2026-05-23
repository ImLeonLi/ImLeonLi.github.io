---
layout: post
title: "在 X-AnyLabeling 中使用自定义模型"
date: 2025-01-07 00:00:00 +0800
tags: [PaddleX, X-AnyLabeling, 目标检测, ONNX, RT-DETR]
category: 技术分享
---

# 在 X-AnyLabeling 中使用自定义模型

我使用的飞桨的模型。

## 模型转换

X-AnyLabeling 使用的模型都 ONNX 格式。正好几天前"PaddleX 目标检测技术交流群"群里有人问过这个问题，有人给了这个链接 <https://github.com/PaddlePaddle/Paddle2ONNX>。

使用命令转换模型，--model_filename 和 --params_filename 这两个参数为 --model_dir 参数目录下的文件名，不可带有绝对或相对路径：

```bash
paddle2onnx --model_dir model_dir \
            --model_filename inference.pdmodel \
            --params_filename inference.pdiparams \
            --save_file model.onnx
```

## 第一个问题

使用 PaddleX 进行微调训练，使用的基础模型是 RT-DETR-L。训练后做了评估和预测还都可正常工作。训练环境是 wsl 下的 Ubuntu 24.04 使用 GPU 训练。

提示：加载自定义模型时出错：无效的配置文件格式。于是提了个 issues：<https://github.com/CVHub520/X-AnyLabeling/issues/779>

软件作者给了详细的回复，解决配置文件问题，问题是模型名称和类型是枚举的，可能是有对应的处理类。不能随便写。

作者提供了个网站：<https://netron.app/> ，这个应该是用来看模型结构的。但我现在其实还看不懂这个结构。

## 第二个问题

加载自定义后面后检测不到目标。

考虑是不是这个工具不支持我的模型。

[RT-DETR-L模型详情](https://github.com/PaddlePaddle/PaddleX/blob/release/3.0-beta2/paddlex/repo_apis/PaddleDetection_api/configs/RT-DETR-L.yaml)

PaddleX RT-DETR-L

X-AnyLabeling：rtdetrv2_r50vd_6x_coco.onnx	RT-DETRv2-L-COCO	rtdetrv2l.yaml	161.38MB	百度网盘 | github

## 尝试基于 RT-DETR-R50 微调、评估、推理

### 微调

```bash
python main.py -c paddlex/configs/object_detection/RT-DETR-R50.yaml \
    -o Global.mode=train \
    -o Global.device=gpu \
    -o Global.dataset_dir=./dataset/crossroad \
    -o Global.output=./output-RT-DETR-R50 \
    -o Train.epochs_iters=20 \
    -o Train.num_classes=1 \
    -o Train.learning_rate=0.0005 \
    -o Train.batch_size=4
```

### 评估

```bash
python main.py -c paddlex/configs/object_detection/RT-DETR-R50.yaml \
    -o Global.mode=evaluate \
    -o Global.dataset_dir=./dataset/crossroad \
    -o Global.output=./output-RT-DETR-R50 \
    -o Evaluate.weight_path=./output-RT-DETR-R50/best_model/best_model.pdparams
```

### 推理

```bash
python main.py -c paddlex/configs/object_detection/RT-DETR-R50.yaml \
    -o Global.mode=predict \
    -o Global.output=./output-RT-DETR-R50 \
    -o Predict.model_dir="./output-RT-DETR-R50/best_model/inference" \
    -o Predict.input="test/map1.png"
```

## 参考资料

- [RT-DETR_HGNetV2 简介](https://aistudio.baidu.com/modelsdetail/273)
