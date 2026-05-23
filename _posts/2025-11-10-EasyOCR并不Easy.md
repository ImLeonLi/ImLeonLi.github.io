---
layout: post
title: "EasyOCR 并不 Easy"
date: 2025-11-10 00:00:00 +0800
tags: [EasyOCR, OCR, PaddleOCR, Google Earth]
category: 技术分享
---

# EasyOCR

需求：识别 Google Earth Web 版中的历史图像控制条下的年份。

## 坑1：tessedit_char_whitelist

如果设置了 tessedit_char_whitelist 参数，为所有数字。

可能导致同行临近的两组数字识别成一组。例如：2020             2021    2022             2023             2024

把 2021    2022  识别成了一组  20212022

API Documentation: <https://www.jaided.ai/easyocr/documentation/>

## 并不简单

并不简单是因为，默认参数识别率较低。

需要对很多参数进行尝试调优。才会提高些识别率。

另外，初始化时的速度也比较慢。占用内存比较大。

在识别的尝试中，也使用了下 PaddleOCR 默认情况下就有比较的识别率。

初始化速度、内存占用，与 EasyOCR 差别不大。

具体多少不记得了，等有时间做个测试。

初始化时间大概15秒+，反正能感知到。内存占用大概950MB左右。
