---
title: Cloudflare Workers，新手最容易上手的邊緣函式之一
author: Cooper
date: 2026-02-10 09:00:00 +0800
categories: [Cloudflare]
tags: [cloudflare, workers, serverless, edge]
image:
  path: /cloudflare-workers/banner.svg
---

## 前言

這兩年只要有人問我：「我想先弄個小 API，真的需要先買一台主機嗎？」我腦中第一個跳出來的，常常就是 `Cloudflare Workers`。

不是因為它最潮，也不是因為它能包辦所有事。說穿了，就是省事。你不用先養 VM，不用先配 nginx，也不用先把整套部署流程想得很重。很多本來只想回個 webhook、做個轉址、擋個簡單驗證的小東西，扔進 Worker 就能先跑起來。

Cloudflare 官方現在對 Workers 的定位也講得很直：它就是跑在 Cloudflare 全球網路上的 serverless runtime，前端、後端、背景任務、AI、觀測這些都能接。[1] 我自己比較在意的倒不是名詞有多大，而是它真的讓個人開發者比較容易開始。

## 為什麼我會先想到它

### 不用先養一台 VM

很多麻煩，你一開始就可以先跳過：

- 主機要買哪台
- systemd 要怎麼顧
- nginx 要怎麼配

有些需求本來就不值得先拉一台機器。這種時候，Worker 那種「先上線再說」的感覺就很對味。

### 很適合做小而確定的邏輯

像這些我都會先想到它：

- webhook 接收器
- 簡單 API
- 網頁前置檢查
- 轉址與 A/B 測試
- 定時背景任務

### 東西長大了，也不會立刻卡死

Workers 不是只會回 `fetch`。現在 bindings、Queues、Workflows、D1、R2、AI 這些都已經接得很近。[1]

我喜歡這點，是因為 side project 常常不是一開始就規劃得很漂亮。先做一個能用的小東西，之後再慢慢補資料層、背景工作、儲存，這條路在 Workers 生態裡還算順。

## 最簡單的 Worker 長什麼樣子

先看這種最小版本就好：

```ts
export default {
  async fetch(request: Request): Promise<Response> {
    return new Response('hello from cloudflare workers');
  }
};
```

這種簡單到有點不像在做後端的東西，反而就是它迷人的地方。你不用先鋪太多基礎設施，先讓網址真的能回東西，腦袋會輕很多。

我自己帶新手碰這個，通常只做三件事：

1. 先建一個只回 `hello` 的 Worker
2. 先把網址打通
3. 再往上補 D1、R2 或 Queue

先確定它活著。後面要加 D1、R2、Queue，再說。

## 最近幾個我覺得有感的方向

我翻這陣子的官方文件跟 changelog，對個人開發比較有感的大概是這幾塊：[1][2]

- **bindings**：把 KV、R2、D1、Queues 接進 Worker
- **Queues / Workflows**：把不適合卡在 request 裡的事情丟去背景做
- **Browser Rendering**：現在額度還在往上加，對截圖、爬站、自動化很實用
- **AI Gateway**：如果你有在玩多模型 API，這塊也很值得看

## 第一次上手，真的不要一次灌滿功能

很多人第一次碰 Workers，就想順手把 `D1`、`KV`、`Queue` 全開。老實說，我不太建議。

我自己會照這個順序走：

1. `npm create cloudflare@latest` 先建專案
2. 用最基本的 `fetch()` 回一段字串
3. `wrangler dev` 先在本機看結果
4. 能正常回應之後，再 `wrangler deploy`
5. 最後才開始接 binding

這樣做有個很實際的好處：哪一步壞掉，你會看得很清楚。

我看過比較常見的卡點反而不是程式本身，而是：

- 沒搞懂 local dev 跟 deploy 後環境的差異
- 以為綁完變數就跟傳統 `.env` 一樣
- 還沒打通網址就開始懷疑整個平台

把第一步切小一點，通常會順很多。

## 費用這件事，我的看法很簡單

Cloudflare 現在把 Workers 的免費與付費方案寫得比以前清楚不少，但我自己的做法一直沒變：**先拿免費層把流程做通，真的有流量、有背景任務、有周邊服務成本，再來看要不要升。**[1]

不要一開始就被定價頁嚇到。對個人專案來說，比較該想的是：

- 你的請求量大不大
- 你有沒有用到會額外影響成本的周邊服務
- 你是不是要跑比較長時間或比較重的工作

如果只是 webhook、簡單 API、驗證、轉址，很多時候你真的可以先跑一陣子再說。

## 幾個很容易踩的坑

### 1. 把它當成完全等同 Node 的環境

很多 npm 套件能用，不代表你平常在 Node 伺服器那套可以原封不動搬過來。

### 2. request 裡做太多重工作

本來就該丟到背景做的事，硬塞在一次 request 裡，最後通常只會把自己搞得很累。

### 3. 沒先設計 log 與錯誤觀察

小專案也一樣。前面偷懶不看 log，後面真的壞掉時就會變成「它有錯，但我完全不知道哪裡錯」。

## 什麼情況下，我反而不會先推 Workers

這也得講白，不然文章看起來就像在吹。

如果你要跑的是：

- 很重的長時間工作
- 強依賴某些傳統 Node 執行環境或系統套件
- 已經很成熟、而且大量依賴某雲原生服務的既有系統

那我通常不會硬推 Workers。

它真正討喜的地方，是讓小而明確的服務很快落地，不是叫你把所有東西一股腦搬去 edge。

## 我自己比較常把它拿來做什麼

### 放那些適合靠近邊緣的東西

像 webhook 驗證、API middleware、轉址、簡單 token 檢查，這些放 Workers 很剛好。

### 拿來當小專案的 API 層

如果專案不大，我其實很願意直接把 API 放 Workers，搭配 D1 或 R2 先做。

### 當成 Cloudflare 生態的入口點

因為你一旦進來，後面要接 R2、Queues、D1、AI Gateway 都很自然。

## 幾個要先有心理準備的地方

### 1. 它不是傳統 Node 伺服器

如果你腦中還是 `express + VM` 的思維，第一次寫 Workers 會有點不習慣。

### 2. 你要開始理解 bindings，而不是只看環境變數

Cloudflare 的味道，差不多就是從這裡開始跑出來。

### 3. 小功能很香，大系統要先規劃好

Workers 很適合先跑起來，但如果系統慢慢長成複雜平台，你還是得想清楚資料層和背景流程怎麼拆。

如果你現在問我，Cloudflare 裡哪個東西最值得個人先摸，我多半還是會先說 `Workers`。[1]

理由沒有很玄。很多原本會讓人直覺想開一台小主機的工作，放到這裡反而更輕。不是每件事都適合它，但只要場景對了，它真的會讓你少繞很多路。

## 參考資料

[1]: https://developers.cloudflare.com/workers/
[2]: https://developers.cloudflare.com/changelog/post/2026-03-02-agents-sdk-v070/
[3]: https://developers.cloudflare.com/changelog/post/2026-03-04-br-rest-api-limit-increase/
