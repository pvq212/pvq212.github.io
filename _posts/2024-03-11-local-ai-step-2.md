---
title: 自建 LocalAI，本地使用 ai 模型 - 文字生成圖片
author: doggy
date: 2024-03-11 00:30:00 +0800
categories: [AI, LocalAI]
tags: [Docker, AI, LocalAI, Stable diffusion, 文字生成圖片]
image:
  path: /blog/local-ai-step-2/dog2.png
---

## 文字生成圖片

繼[**第一篇**][1]的文章後，第二個要介紹的是如何在本地透過文字生成圖片

### 模型選擇

首先是模型選擇，第一篇有提到，我拿來測試的硬體沒有獨立的 GPU，可以用來 cuda 加速，所以模型的選擇目標會優先挑選 `step` 次數少，輕量的模型。

關於 `step` 的介紹可以看[**這邊**][2]，簡而言之就是模型選染的次數，越多細節會越高，當然也不是越高越好。

所以測試過後，選擇了[**sdxl-turbo**][3] 這個模型，支援 `onnx`，優點是 `step` 甚至只要設置為 `1` 就可以有不錯的效果

### 安裝

跟第一篇不同的是，撰寫此文的當下，`sdxl-turbo` 並沒有在官方的模組內，所以這邊我們自己撰寫一個 `yaml` 檔，`LocalAI` 會幫我們自己下載對應的模型，並使用 gRPC 去自動執行

首先我們一樣進到 `models` 內，新增一個 `sdxl-turbo.yaml` 檔，內容如下

```yaml
name: sdxl-turbo
parameters:
  model: stabilityai/sdxl-turbo
backend: diffusers
step: 1
cuda: false
f16: false
diffusers:
  scheduler_type: euler_a
  cfg_scale: 1
```

其中 `step` 設置為 `1`，`cuda` 設置為 `false` 不啟用 `cuda` 加速

接著回到 `docker-compose.yml` 所在目錄下執行

```console
docker-compose restart
```

重新啟動容器

### 測試

接下來就要執行安裝模型以及測試 api 的階段，我們直接打下面的 api

```bash
curl --location 'http://127.0.0.1:8080/v1/images/generations' \
--header 'Content-Type: application/json' \
--data '{
    "prompt": "A white Gundam robot, over 20 meters tall, standing on the ruins of a city with burning buildings and smoky sky in the background. High-quality image of a majestic white Gundam towering over a post-apocalyptic cityscape, showcasing intricate details and dramatic composition. Perfect for fans of mecha and sci-fi art, this artwork captures the essence of futuristic warfare | cartoon, 3d, ((disfigured)), ((bad art)), ((deformed)),((extra limbs)),((close up)),((b&w)), wierd colors, blurry, (((duplicate))), ((morbid)), ((mutilated)), [out of frame], extra fingers, mutated hands, ((poorly drawn hands)), ((poorly drawn face)), (((mutation))), (((deformed))), ((ugly)), blurry, ((bad anatomy)), (((bad proportions))), ((extra limbs)), cloned face, (((disfigured))), out of frame, ugly, extra limbs, (bad anatomy), gross proportions, (malformed limbs), ((missing arms)), ((missing legs)), (((extra arms))), (((extra legs))), mutated hands, (fused fingers), (too many fingers), (((long neck))), Photoshop, video game, ugly, tiling, poorly drawn hands, poorly drawn feet, poorly drawn face, out of frame, mutation, mutated, extra limbs, extra legs, extra arms, disfigured, deformed, cross-eye, body out of frame, blurry, bad art, bad anatomy, 3d rende",
    "model": "sdxl-turbo",
    "size": "512x512"
  }'
```

其中有幾個參數介紹

- `model`，config 內定義的 name，也就是 `sdxl-turbo`
- `size`，如同第一篇所說，依照 openapi 規範設計，所以這邊僅支援 `256`、`512`、`1024` 三種尺寸
- `prompt`，提示詞，有幾個訣竅，例如想要產生高品質的圖片，可以打上 `High-quality`、`8k`、`detailed`，等，網路上有許多中文的 [**stable-diffusion 生成器**][4] 可以使用
- `negative-prompt`，負面提示詞，在 api 中的使用，是串接於正面提示詞後，並用 `|` 區分，例如生成人體時，會用來禁止生成畸形的身體或是奇怪的手指

以上面的提示詞經過我的 N100 約 50 秒的生成後，產出了這些結果

![alt text](/blog/local-ai-step-2/gundam.png)

`大樓背景以及煙霧等細節都還挺不錯`

#### prompt
> A white Gundam robot, over 20 meters tall, standing on the ruins of a city with burning buildings and smoky sky in the background. High-quality image of a majestic white Gundam towering over a post-apocalyptic cityscape, showcasing intricate details and dramatic composition. Perfect for fans of mecha and sci-fi art, this artwork captures the essence of futuristic warfare | cartoon, 3d, ((disfigured)), ((bad art)), ((deformed)),((extra limbs)),((close up)),((b&w)), wierd colors, blurry, (((duplicate))), ((morbid)), ((mutilated)), [out of frame], extra fingers, mutated hands, ((poorly drawn hands)), ((poorly drawn face)), (((mutation))), (((deformed))), ((ugly)), blurry, ((bad anatomy)), (((bad proportions))), ((extra limbs)), cloned face, (((disfigured))), out of frame, ugly, extra limbs, (bad anatomy), gross proportions, (malformed limbs), ((missing arms)), ((missing legs)), (((extra arms))), (((extra legs))), mutated hands, (fused fingers), (too many fingers), (((long neck))), Photoshop, video game, ugly, tiling, poorly drawn hands, poorly drawn feet, poorly drawn face, out of frame, mutation, mutated, extra limbs, extra legs, extra arms, disfigured, deformed, cross-eye, body out of frame, blurry, bad art, bad anatomy, 3d rende


![alt text](/blog/local-ai-step-2/dog1.png)

`也會自動優化景深效果，強調物體本身`

#### prompt
> A chubby and adorable Husky with big black spots on its fluffy body, happily smiling with bright eyes, cute and short limbs, standing on green grass. High-quality photo of a plump and lovable Husky dog, showcasing its unique markings and joyful expression. Perfect for pet lovers and animal enthusiasts, this image captures the essence of cuteness and happiness | skinny, ugly, ((angry)), ((scary)), ((scars)), ((injuries)), ((missing limbs)), ((mutilated)), (extra eyes), (extra ears), (extra nose), (extra mouth), (disproportionate body), (unattractive markings), (poorly groomed), (unhealthy), (unhappy), (unnatural posture), (unnatural smile), (aggressive pose), (unpleasant expression), (demonic), (pixelated), (low resolution), (bad lighting), (blurry), (bad composition), (out of focus), (unnatural colors)

![alt text](/blog/local-ai-step-2/dog2.png)

`水波的效果雖然不夠真實，但對於 step 只有一次的生成，效果算是很不錯了`

#### prompt
> A chubby dog swimming in clear water, its round body floating and gently paddling with its limbs, displaying a cheerful and cute expression. High-quality image of a plump dog enjoying a swim, capturing the joy and innocence of pets in water. Perfect for animal lovers and those who appreciate adorable moments, this photo radiates happiness and relaxation | dry, ((sad)), ((injured)), ((angry)), ((scared)), ((missing limbs)), ((mutilated)), (extra eyes), (extra ears), (extra nose), (extra mouth), (disproportionate body), (unattractive markings), (poorly groomed), (unhealthy), (unhappy), (unnatural posture), (unnatural smile), (aggressive pose), (unpleasant expression), (demonic), (pixelated), (low resolution), (bad lighting), (blurry), (bad composition), (out of focus), (unnatural colors)

每張圖片大概用時都在 30s ~ 70s 之間，出來的效果出乎意料的都不差，重點是自己生成的圖片沒有版權問題
自己上文章少了示意圖時非常有實用性

[1]: https://blog.learntw.com/posts/local-ai-step-1/
[2]: https://vocus.cc/article/64410910fd89780001d5fca7
[3]: https://huggingface.co/stabilityai/sdxl-turbo
[4]: https://flowgpt.com/p/stable-diffusion
