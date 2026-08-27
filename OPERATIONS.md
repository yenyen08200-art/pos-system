# 維運手冊

給未來的自己（或接手的人）看的。三件事：**怎麼安全上線**、**怎麼在本機測**、**還有哪些沒設定完**。

---

## 1. 上線前先用預覽網址測（Preview Channels）

以前改完直接 `firebase deploy --only hosting`，等於**改動直接進到正在營業的門市**。
萬一哪次改壞了，店員跟顧客當下就會看到壞掉的畫面。

改用預覽頻道：先部署到一個**臨時網址**，自己點過一輪確認沒問題，再推正式站。

```bash
cd pos-system

# 1) 部署到預覽網址（保留 7 天後自動消失）
firebase hosting:channel:deploy preview --expires 7d
#    → 執行完會印出一個網址，像 https://texas-chick-event--preview-xxxx.web.app

# 2) 用那個網址測試：登入、核銷、領券、抽獎都走一遍

# 3) 確認沒問題，才推正式站
firebase deploy --only hosting
```

其他常用指令：

```bash
firebase hosting:channel:list              # 看目前有哪些預覽頻道
firebase hosting:channel:delete preview    # 手動刪掉某個預覽頻道
```

> 預覽網址是**獨立的**，用的是同一個 Firestore 資料庫。
> 所以測「核銷」這種**會寫入資料**的功能時，記得用測試用的券，不要拿真的顧客資料練習。
> 真的要完全隔離，用下面的本機模擬器。

---

## 2. 本機模擬器（完全不碰正式資料）

在自己電腦上跑一份假的 Firestore / Auth / Functions，測試新功能**完全不會動到正式資料**，也不吃雲端額度。

```bash
# 測前端（Hosting + Firestore + Auth）
cd pos-system
firebase emulators:start
#   網頁     → http://localhost:5000
#   控制台   → http://localhost:4000   （可以直接看／改假資料）

# 測後端 Cloud Functions
cd AutoExpireBot
firebase emulators:start
#   Functions → http://localhost:5001
```

> ⚠️ 模擬器裡的資料是**假的、關掉就沒了**。這是刻意的——要的就是「隨便玩壞都沒關係」。

---

## 3. 還沒設定完的：App Check

程式碼已經接好了，但**還沒啟用**——`RECAPTCHA_SITE_KEY` 目前是空字串，整段會自動跳過。

App Check 擋的是「有人直接拿 API 網址寫腳本狂打」。
密語、名額上限是**規則層**的防護；App Check 是**來源層**，兩層互補。

### 啟用步驟

1. Firebase Console → **App Check** → **Apps** → 註冊這個網站，供應商選 **reCAPTCHA v3**
2. 複製拿到的**網站金鑰**
3. 貼進這三個檔案的 `RECAPTCHA_SITE_KEY`：
   - `admin.html`
   - `test_index.html`
   - `test_register.html`
4. 部署，然後**先觀察幾天**（Console 的 App Check 頁面會顯示「已驗證 / 未驗證」的請求比例）
5. 確認正常顧客都是「已驗證」之後，**最後才開啟強制執行**

> ⚠️ **不要一開始就開強制執行。** 先觀察是為了確認沒有誤擋——
> 萬一某些顧客的瀏覽器過不了 reCAPTCHA，直接開強制會讓他們完全領不到券。

---

## 4. 排程任務失敗會自動通知

`autoExpireTickets`、`autoShelfEvents`、`refreshMetaTokens` 三支排程如果執行失敗，
會**自動發 LINE 訊息到日報群組**，內容包含任務名稱、時間、錯誤訊息。

以前失敗只會在 Cloud Logging 留一行，除非剛好去翻 log 否則不會發現——
可能整整幾天活動都沒正確上下架。

---

## 5. 顧客回報「領不到券」怎麼查

前端錯誤會自動送回雲端。到 **Firebase Console → Functions → 記錄檔**，搜尋：

```
[client]
```

會看到發生在哪一段流程（`liff-init` / `line-signin` / `load-config` / `load-tickets` / `claim`）、
錯誤代碼、以及當下操作的活動。不用再請顧客截圖。

---

## 6. 定期要看一眼的：執行環境版本

Cloud Functions 的 Node.js 版本**有生命週期**，Google 會定期淘汰舊版。
部署時如果看到這種警告，就是該升級了：

```
Runtime Node.js 20 was deprecated on 2026-04-30 and will be decommissioned...
```

升級方式：改 `AutoExpireBot/functions/package.json` 的 `engines.node`，然後重新部署。

```json
"engines": { "node": "22" }
```

> ⚠️ **不要拖到 decommission（停用）那天**。deprecated 只是警告，還能部署；
> 一旦 decommissioned，就再也部署不上去了，得先升級才能修任何東西——
> 那時候如果剛好遇到緊急狀況會很麻煩。

順帶一提，部署時可能還會看到「firebase-functions 版本過舊」的提醒。
那個**沒有時效壓力**，現有版本仍然正常運作；要升級的話屬於大版本更新，
建議先用預覽頻道與模擬器測過再上正式站。

---

## 7. 已經設定好、不用再管的

| 項目 | 狀態 |
|---|---|
| Firestore 每日自動備份 | 保留 7 天，Console → Firestore → 災難復原 |
| GCP 預算警報 | 50% / 90% / 100% 三段 email 通知 |
| Firestore Rules | 版控在 `firestore.rules`，改完用 `firebase deploy --only firestore:rules` |
| 效能監控 | 已接上，資料在 Console → Performance |
| 靜態資源快取 | HTML 不快取（改版立即生效）、圖片存一年 |

> 💡 **HTML 設成不快取**之後，改版不用再叫店員 Ctrl+Shift+R 了，重新整理就是最新版。
