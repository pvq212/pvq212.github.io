---
title: Google Cloud Run，自動調整擴容的實例
author: doggy
date: 2024-03-14 00:30:00 +0800
categories: [Server]
tags: [k8s, GCP, Go, ddos]
---

## 前言

對於中小公司來說，自己維護 `k8s` 不只坑多，尋找維護的人手也是一個問題，這時候這種代管 `k8s` 的服務就很方便，
最近工作上遇到了客戶反映流量達到峰值時，伺服器常常會回應 `502` 或是 timeout，但機器更多時間又是空閒的，所以機器開更大會造成資源的浪費，也不一定能保證能滿足以後日益增加的流量需求，所以我將服務遷移到 `Google Cloud Run`，這幾篇會開始簡單的介紹，以及解決了甚麼問題

## 準備

首先簡單說明一下開發以及部屬的流程

1. 撰寫 `.gitlab-ci.yml` 透過 gitlab 去執行 `ci/cd`
2. `ci` 通過後將專案包成一個 `image`，推送到 `google artifact registry`，或者 `dockerhub`，都可以
3. 將 `image` 推送至 `Cloud Run`，並將流量 100% 轉送至新版本

這是一套自動化部屬服務的方法，這篇重點在於介紹 `Cloud Run` 這個服務本身，自動部屬的服務或是方法每個人的不一樣，但結果相信會是相同的

## 建立一個 demo 專案

我們先準備一個測試用的 go 專案，首先準備一個 `main.go`，這支程式暴露了一個 `8080` port，並提供了一個簡單的 api 服務，簡單的模擬了轉帳的流程

```golang
package main

import (
	"database/sql"
	"log"
	"net/http"
	"os"

	"github.com/gin-gonic/gin"
	_ "github.com/go-sql-driver/mysql"
)

type Transaction struct {
	Name     string `json:"name"`
	Balance  int    `json:"balance"`
	Function string `json:"function"`
}

func main() {
	db, err := sql.Open("mysql", "username:password@tcp(ip:port)/tablename")
	if err != nil {
		log.Fatal(err)
	}
	defer db.Close()
	r := gin.Default()
	r.POST("/transact", func(c *gin.Context) {
		var t Transaction
		if err := c.ShouldBindJSON(&t); err != nil {
			c.JSON(http.StatusOK, gin.H{"message": err.Error(), "code": 1})
			return
		}

		var sqlQuery string
		switch t.Function {
		case "deposit":
			sqlQuery = `INSERT INTO users (balance, name) VALUES (?, ?)
             ON DUPLICATE KEY UPDATE balance = balance + VALUES(balance)`
		case "withdraw":
			sqlQuery = "UPDATE users SET balance = balance - ? WHERE name = ?"
		default:
			c.JSON(http.StatusOK, gin.H{"message": "Invalid function", "code": 2})
			return
		}
		_, err := db.Exec(sqlQuery, t.Balance, t.Name)
		if err != nil {
			log.Fatal(err)
		}
		c.JSON(http.StatusOK, gin.H{"message": "transaction completed", "code": 0})
	})
	port, exists := os.LookupEnv("PORT")
	if !exists {
		port = "8080"
	}

	r.Run(":" + port)
}

```

然後是 `go.mod`

```golang
go 1.22

require github.com/go-sql-driver/mysql v1.8.0

require (
	filippo.io/edwards25519 v1.1.0 // indirect
	github.com/bytedance/sonic v1.9.1 // indirect
	github.com/chenzhuoyu/base64x v0.0.0-20221115062448-fe3a3abad311 // indirect
	github.com/gabriel-vasile/mimetype v1.4.2 // indirect
	github.com/gin-contrib/sse v0.1.0 // indirect
	github.com/gin-gonic/gin v1.9.1 // indirect
	github.com/go-playground/locales v0.14.1 // indirect
	github.com/go-playground/universal-translator v0.18.1 // indirect
	github.com/go-playground/validator/v10 v10.14.0 // indirect
	github.com/goccy/go-json v0.10.2 // indirect
	github.com/json-iterator/go v1.1.12 // indirect
	github.com/klauspost/cpuid/v2 v2.2.4 // indirect
	github.com/leodido/go-urn v1.2.4 // indirect
	github.com/mattn/go-isatty v0.0.19 // indirect
	github.com/modern-go/concurrent v0.0.0-20180306012644-bacd9c7ef1dd // indirect
	github.com/modern-go/reflect2 v1.0.2 // indirect
	github.com/pelletier/go-toml/v2 v2.0.8 // indirect
	github.com/twitchyliquid64/golang-asm v0.15.1 // indirect
	github.com/ugorji/go/codec v1.2.11 // indirect
	golang.org/x/arch v0.3.0 // indirect
	golang.org/x/crypto v0.9.0 // indirect
	golang.org/x/net v0.10.0 // indirect
	golang.org/x/sys v0.8.0 // indirect
	golang.org/x/text v0.9.0 // indirect
	google.golang.org/protobuf v1.30.0 // indirect
	gopkg.in/yaml.v3 v3.0.1 // indirect
)

module game

```

最後是 `Dockfile`

```docker
FROM golang:1.22

WORKDIR /app

COPY . .

RUN go mod download

RUN go build -o main .

CMD ["./main"]
```

然後我們將專案 `build` 起來後 push 至 `dockerhub`

```bash
docker build . -t username/demo && docker push username/demo
```
> 要注意推送的庫沒有額外設定的話會是公開的，請不要將正式的 db conf 等敏感資訊丟上
{: .prompt-warning }

到這邊你就做好了準備工作


## 將鏡像部屬至 Cloud Run

![alt text](/google-cloud-run/1.png)

首先點選創建服務

![alt text](/google-cloud-run/2.png)

接著進到服務裡來後，我們依序填上

1. 容器映像網址，`username/demo`
2. 服務名稱，`demo`
3. 允許未通過身分驗證的調用 `勾選`
4. 實例數下限 `1`，減少冷啟動時間
5. 區域這邊就自行選擇

下方的參數可以保留預設，當然也可以自己設定 `ENV` 或是 `Cloud Sql` 的連接，接著就可以點擊創建

![alt text](/google-cloud-run/3.png)

創建完畢後就可以看到會提供一個 `https` 的網址以及流量 `100%` 轉送至這個最新的版本
接下來可以用壓力測試，然後看統計中的容器實例數量，如果會隨著容器使用量而增加實例個數，代表部屬成功了
接下來一些進階項目就可以留給大家嘗試

1. 設定 `Cloud Armor`，阻擋 `ddos` 攻擊
2. 設定私有網域，將 `Cloudflare` 套用至 `Cloud Run` 服務，使用自己的域名部屬公開服務，並套用 `Cloudflare` 防禦規則
3. 新版本出問題時，自動退版將流量倒回上一個版本
