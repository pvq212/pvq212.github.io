---
title: Cloudflare Tunnel，對個人最實際的吸引力就是：不用先公開一台機器
author: Cooper
date: 2026-02-14 09:00:00 +0800
categories: [Cloudflare]
tags: [cloudflare, tunnel, zero-trust, self-hosted]
image:
  path: /cloudflare-tunnel/banner.svg
---

## 前言

很多人跑去看 `Cloudflare Tunnel`，不是因為突然對 Zero Trust 很有研究，而是心裡有個很普通的念頭：

> 我想把家裡、VPS、NAS、內網服務接出來，但不想先把 origin 直接暴露在網路上。

這種需求很日常，可是以前真的很容易把自己搞到防火牆、NAT、反向代理那堆設定裡面。Tunnel 之所以有吸引力，說穿了，就是它幫你少碰了不少這種麻煩。

官方現在對它的描述也很直白：`cloudflared` 會從你自己的環境主動往 Cloudflare 建立 outbound-only connection，所以你不需要先弄一個公開 IP 再接回來。[1]

## 對個人來說，吸引力到底在哪

### 不用先把來源主機裸露出去

這點最直接。你不用第一步就把 origin 攤在網路上。

### 很適合先把小服務接出來試

像這些都很常見：

- 內網 dashboard
- 家裡 NAS 小服務
- 測試站
- SSH / RDP / Web app

### 跟 Cloudflare 其他服務接在一起很順

如果你本來就有 DNS、Access、Workers，Tunnel 很容易接進來。

## 它到底是在幹嘛

官方文件寫得很清楚：`cloudflared` 會從 origin 對外建立連線，之後流量在 tunnel 上雙向走。[1]

這個概念其實不複雜，你就把它想成：

- 不是外面直接打到你機器
- 而是你機器自己先出去牽一條線
- 之後 Cloudflare 幫你接住這條線

如果你本來就不想開一堆 inbound port，這套做法會讓人安心很多。

第一次試的時候，我很建議把目標壓小一點：

1. 先選一個本機或內網 HTTP 服務
2. 先讓 `cloudflared` 能把它接出去
3. 確認外部能進，再考慮 Access 或更多權限控管

不要一上來就把 SSH、RDP、內部面板、私有 API 全掛上去。真的亂掉時，你只會很想把昨天的自己揍一頓。

## 我自己第一次會怎麼測

如果你只是想先驗證它到底能不能用，我會走很保守的版本：

1. 先在本機起一個最簡單的 HTTP 服務
2. 裝 `cloudflared`
3. 先用官方教的 quick tunnel 或登入後建立正式 tunnel
4. 先確認外網能打到你的測試頁
5. 最後才把真正服務換上去

這樣排錯最省腦，不會一次面對太多變數。

很多人第一次卡住，問題其實不在 Tunnel 本身，而是：

- 原本的本機服務就沒正常起來
- DNS 對應沒看清楚
- 想接的其實不是 HTTP，而是另外一種協定

## 有些服務我敢接，有些我會先忍住

### 我會先接的

- 個人面板
- 測試站
- 只給自己或少數人看的內部服務

### 我不會一開始就急著接的

- 權限本來就很複雜的後台
- 沒做身份驗證的敏感管理介面
- 你自己都還不確定穩不穩的老舊服務

比較容易接出去，不代表什麼都該接出去。這兩件事差很多。

## 真要用得安心，通常還是會配 Access

很多人裝完 Tunnel 就鬆一口氣，覺得任務完成。其實如果後面那個服務本身很敏感，我通常還是會再加一層 `Cloudflare Access`，至少把登入或身份驗證補起來。

不然你只是把「裸露 origin」換成「裸露在另一種入口」而已。

## 外面打不進來時，我通常先查這幾個地方

如果今天外面打不進來，我通常會先按這個順序排：

1. 本機服務自己有沒有活著
2. `cloudflared` 有沒有正常連上
3. Cloudflare dashboard 裡的 route / hostname 設定對不對
4. 如果有加 Access，身份驗證流程是不是擋住了

照這個順序一層一層拆，比直接懷疑「Cloudflare 又抽風了」有效得多。

## 為什麼它對個人很有吸引力

因為很多個人需求其實不是大規模架構，而是：

- 我想讓自己能安全連回家裡某個服務
- 我想暫時把測試站接出來
- 我不想動防火牆太多規則

這幾種情境，Tunnel 真的很實用。

## 最近的一些值得注意的變化

Cloudflare 2026 年 2 月還補了 Tunnel management 進主 dashboard，代表它在操作體驗上也繼續往一般使用者靠。[2]

這種更新不算大新聞，可是對一般使用者真的重要。很多工具不是功能不夠，而是入口太散、設定太碎，最後人就懶得用了。

## 哪些場景我真的會用它

### 1. 家裡 NAS 或自架面板

### 2. 臨時測試站

### 3. 不想公開 origin 的內部工具

### 4. 想搭配 Cloudflare Access 做更細的權限控管

## 新手最常踩的點

### 1. 以為裝完就等於完整安全方案

不是。Tunnel 幫你解決的是連線與暴露面的問題，不是所有安全問題。

### 2. 內部服務設計本身如果很鬆，Tunnel 不會自動幫你變安全

### 3. 本機 / 家網環境還是要看 DNS、憑證、服務本身穩不穩

如果你本來就有一些東西想接出來給自己用、給朋友用，或者只是想做個不那麼粗暴的公開入口，我覺得 `Cloudflare Tunnel` 很值得摸。[1]

它不像 Workers 那樣有平台感，也不像 R2 那樣一眼就知道是儲存服務。它做的事情比較單純：把「我要接出來，但又不想開太多洞」這個老問題，處理得稍微優雅一點。

## 參考資料

[1]: https://developers.cloudflare.com/cloudflare-one/networks/connectors/cloudflare-tunnel/
[2]: https://developers.cloudflare.com/changelog/2026-02-20-tunnel-core-dashboard/
