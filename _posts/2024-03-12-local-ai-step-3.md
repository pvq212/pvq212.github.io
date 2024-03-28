---
title: 自建 LocalAI，本地使用 ai 模型 - 文字轉語音
author: doggy
date: 2024-03-11 00:30:00 +0800
categories: [AI, LocalAI]
tags: [Docker, AI, LocalAI, tts, 文字轉語音]
---

## 前言

這是 `LocalAI` 系列的第三篇文章，要來介紹如何將文字轉成語音，以及切換不同的模型

首先可以看到官網的使用方式，有很多模型可以切換，這邊推薦兩個

1. `Bark`，轉換後的文字是帶有情緒的
2. `Piper`，更成熟的模型，基於 `Piper` 訓練的中文模型也很多

相較於情緒、背景音等元素，我更看重產出速度，所以選擇了這個模型 [**zh_CN-huayan-medium.onnx**][1]

我們一樣先將模型下載下來，並放進 `models` 資料夾中

```console
wget https://huggingface.co/csukuangfj/vits-piper-zh_CN-huayan-medium/blob/main/zh_CN-huayan-medium.onnx?download=true -O zh_CN-huayan-medium.onnx
```

接著重啟容器

```console
docker-compose restart
```

然後就可以拿這個模型來試試看產出效果啦

```bash
curl --location 'http://127.0.0.1:8080/tts' \
--header 'Content-Type: application/json' \
--data '{
  "model": "zh_CN-huayan-medium.onnx",
  "backend": "piper",
  "input":"觀自在菩薩。行深般若波羅蜜多時。照見五蘊皆空。度一切苦厄。舍利子。色不異空。空不異色。色即是空。空即是色。受想行識。亦復如是。"
  }'

```
> 等待 `3` 秒鐘左右，就可以產出中文語音啦

{% include embed/music.html src='/local-ai-step-3/tts.mp3' %}

[1]: https://huggingface.co/csukuangfj/vits-piper-zh_CN-huayan-medium/tree/main
