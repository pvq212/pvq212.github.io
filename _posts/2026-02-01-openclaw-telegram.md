---
title: OpenClaw 串 Telegram，我目前的真實感想：能玩，但還不是萬靈丹
author: Cooper
date: 2026-02-01 09:00:00 +0800
categories: [AI Tools]
tags: [ai, openclaw, telegram, bot, self-hosted]
image:
  path: /openclaw-telegram/banner.svg
---

## 前言

最近有不少人在玩 `OpenClaw`，尤其是拿來串 `Telegram` 做自己的 AI bot。

我一開始也被這個方向吸引到，因為它看起來真的很容易讓人腦補：

- 有自己的 agent
- 可以走訊息平台
- 可以接多種模型
- 還有自託管味道

但我真的把官方文件、release note 跟 Telegram 這條線摸過一輪之後，感覺就比較冷靜了：**它不是不能用，而是優點跟限制都非常明顯。**

你本來就想做自己的 AI 入口，它滿有意思。可如果你期待的是「十五分鐘做出一個超穩、超萬用、放著就能跑的助理」，那真的先別想太美。

## OpenClaw 串 Telegram，現在大概到哪裡了

OpenClaw 官方 Telegram 文件目前把這個通道標成 production-ready；至少在 bot DM 和群組場景上，文件算完整，連 `dmPolicy`、群組提及規則、live preview streaming、webhook / long polling 都有寫到。[1]

最基本的做法，大致是這樣：

1. 去 Telegram 的 `@BotFather` 建 bot
2. 把 bot token 寫進 OpenClaw 設定
3. 啟動 gateway
4. 用 pairing 或 allowlist 控制誰能用

官方文件給的最基本設定大概長這樣：

```json
{
  "channels": {
    "telegram": {
      "enabled": true,
      "botToken": "123:abc",
      "dmPolicy": "pairing",
      "groups": {
        "*": {
          "requireMention": true
        }
      }
    }
  }
}
```

如果只是想先讓它在私訊能回話，這一步其實沒有想像中難。

## 我自己會怎麼起手

第一次碰的話，我真的不建議一開始就把群組、topic、多 agent routing 全部打開。

先從最單純的 DM 開始就好。

### 第一步：建立 bot token

這一步就去找 `@BotFather`，照官方流程做就好。[1]

### 第二步：先用 DM 測通

啟動後先看 pairing 和 logs。

```bash
openclaw gateway
openclaw pairing list telegram
openclaw pairing approve telegram <CODE>
```

先走這一輪，你至少會知道：

- bot 能不能起來
- token 有沒有設對
- pairing 流程有沒有走通

### 第三步：再碰群組

群組才是最容易讓人踩坑的地方。

因為 Telegram 本身就有 privacy mode、group allowlist、mention 規則，OpenClaw 又多了一層自己的設定。[1]

你如果一開始就把群組、topic、多個 agent 一起開，最後出問題很難查。

## 為什麼我會說它能玩，但別期待太滿

### 優點一：文件現在比我原本預期的完整

這點我先給肯定。

官方 Telegram 文件不是只有簡單貼個範例而已，而是真的把 `dmPolicy`、`groupPolicy`、`allowFrom`、`streaming`、webhook、proxy、topic 行為都補得很細。[1]

而且最近 release 也明顯一直在補 Telegram 相關功能，例如：

- 預設 streaming 改成 `partial`
- DM live preview 用 `sendMessageDraft`
- 群組語音訊息 mention 檢查補了 `disableAudioPreflight`
- 還新增 `openclaw config validate` 幫你先驗設定再啟動。[2]

這代表它不是放著沒人管，而是真的一直在補。

### 優點二：很適合做自己的單一入口

如果你平常就習慣 Telegram，那把 AI 入口放這裡確實很順。

你不用每次開另外一個 app，也不用硬把所有東西塞進瀏覽器。很多快速提問、簡單查詢、或叫它幫你做單一步驟，放在 Telegram 裡很自然。

### 優點三：有自託管的味道，對喜歡折騰的人很有吸引力

這也是它會讓人想繼續玩的地方。

你可以自己決定：

- 用哪個模型
- 誰能對 bot 講話
- 群組怎麼限制
- 要不要開 streaming
- 要不要掛 proxy

這種自由度，對喜歡自己掌控的人會很香。

## 但麻煩也是真的不少

### 缺點一：設定面很多，對新手不算友善

這點是我最直接的感想。

雖然文件完整，但你一旦真的開始看，會發現設定項目很多。

例如一個 Telegram bot，光是 access control 就有：

- `dmPolicy`
- `allowFrom`
- `groupPolicy`
- `groupAllowFrom`
- `requireMention`
- `groups`
- `topics`

這還沒算 webhook、streaming、reaction、actions、proxy 這些東西。

如果你只是想趕快有個能用的 bot，第一次看這堆設定真的會有點煩。

### 缺點二：真正好用的場景，其實偏侷限

我現在覺得它最適合的場景，大概就是：

- 個人 bot
- 小圈圈群組 bot
- 幫自己做查詢、整理、回覆、提醒

但如果你期待的是一個可以穩穩扛住大型團隊工作流、跨頻道、跨 agent、長期執行而且超省心的系統，那目前還不到那個成熟度。

### 缺點三：訊息平台天生就不適合複雜開發任務

這倒不完全是 OpenClaw 的錯，更多是 Telegram 這種訊息平台本來就有它的天花板。

你在 Telegram 裡叫 agent 做事，很適合短任務、查狀態、快速回覆。

但只要事情變成：

- 多檔案修改
- 長時間推理
- 要看大量輸出
- 要驗證細節

你最後多半還是會想回 terminal 或 IDE。講白一點，Telegram 比較像入口，不太像主戰場。

## 幾個坑我會先提醒一下

### 1. 群組看不到訊息，不一定是 OpenClaw 壞掉

很多時候是 Telegram privacy mode 沒處理好。官方文件就直接寫了，如果 bot 要看到非 mention 訊息，你可能得關 privacy mode，或者直接把 bot 設成 admin，還要重新加回群組。[1]

### 2. VPS 網路環境不穩，可能要補 proxy

官方 troubleshooting 有提到，如果 Telegram API 連線失敗、IPv6 有問題、或主機對 `api.telegram.org` 不穩，可以考慮在 `channels.telegram.proxy` 走 SOCKS/HTTP 代理。[1]

這種問題如果沒碰過，很容易第一時間怪到程式身上，其實很多時候是網路環境在搞事。

### 3. 先用 `config validate`

這個我覺得很實用。

最近 release 已經補上 `openclaw config validate --json`，至少在啟動前，可以先知道你是不是寫錯 key 或結構。[2]

如果你問我值不值得玩，我的答案還是：**值得，但期待值要放對。** 你要把它當成一個還在高速演進、文件跟 release 都很活躍的自架入口，不是零設定成品。[2][3]

它的優點不是「所有場景都比別人穩」，而是你可以把一個訊息平台 bot 做得很有自己的味道，而且功能面越來越完整。

但它的缺點也很明顯：

- 設定不算少
- 場景偏窄
- 真正複雜的任務還是 terminal / IDE 比較舒服

所以我現在對它的定位比較像：

> 一個很有潛力、很適合拿來做自己的 AI 入口，但還沒有到人人都該直接上線重度依賴的東西。

## 參考資料

[1]: https://docs.openclaw.ai/channels/telegram
[2]: https://github.com/openclaw/openclaw/releases/tag/v2026.3.2
[3]: https://github.com/openclaw/openclaw/releases
