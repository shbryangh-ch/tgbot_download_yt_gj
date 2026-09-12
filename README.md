# YT-DLP Telegram Bot

一個簡單的 Telegram Bot，讓使用者傳送影片連結，Bot 會用 [yt-dlp](https://github.com/yt-dlp/yt-dlp) 下載影片並回傳到 Telegram。支援串接 **Local Bot API Server**，突破官方 50MB 上傳限制，最高可傳送到 2GB。

## 功能

- 支援 yt-dlp 涵蓋的網站（YouTube、Twitter/X、Instagram 等）
- 自動選擇最佳畫質，並合併音訊與影像軌
- 下載後自動清理暫存檔案
- 可選擇串接 Local Bot API Server，突破 50MB 上傳限制
- 上傳前自動檢查檔案大小，避免無謂的失敗上傳

## 前置需求

- Python 3.9 以上
- [ffmpeg](https://ffmpeg.org/)（合併影音軌時需要）
- 一個 Telegram Bot Token（透過 [@BotFather](https://t.me/BotFather) 申請）
- （選用）Docker，用於架設 Local Bot API Server

## 安裝

### 1. 下載專案

```bash
git clone <你的 repo 網址>
cd <專案資料夾>
```

### 2. 安裝 Python 套件

```bash
python -m pip install python-telegram-bot yt-dlp
```

> 建議使用虛擬環境（`python -m venv venv`），避免套件版本互相干擾。

### 3. 安裝 ffmpeg

**Windows：**
```powershell
winget install ffmpeg
```
若 `winget` 無法使用，改為手動下載並加入 PATH：
1. 到 https://www.gyan.dev/ffmpeg/builds/ 下載 `ffmpeg-release-essentials.zip`
2. 解壓縮到例如 `C:\ffmpeg`
3. 將 `C:\ffmpeg\bin` 加入系統環境變數 PATH

**macOS：**
```bash
brew install ffmpeg
```

**Linux：**
```bash
sudo apt install ffmpeg
```

### 4. 申請 Telegram Bot Token

1. 在 Telegram 搜尋 `@BotFather`
2. 傳送 `/newbot`，依指示設定名稱與 username
3. 取得 Token（格式類似 `123456789:ABCdefGHIjklMNOpqrsTUVwxyz`）

## 設定

打開 `yt_dlp_telegram_bot.py`，修改以下設定：

```python
BOT_TOKEN = "YOUR_BOT_TOKEN_HERE"   # 換成你的真實 Token
DOWNLOAD_DIR = "downloads"
MAX_FILESIZE_MB = 50                # 官方 API 上限 50，架設 Local Server 後可調高

USE_LOCAL_API_SERVER = False        # 若有架設 Local Bot API Server，改成 True
LOCAL_API_BASE_URL = "http://localhost:8081/bot"
```

## 執行

```bash
python yt_dlp_telegram_bot.py
```

看到 `Bot 啟動中...` 代表成功，去 Telegram 跟 bot 對話並傳送影片連結測試。

## 突破 50MB 限制：架設 Local Bot API Server

官方 Telegram Bot API 上傳檔案上限是 50MB。若需要傳送更大的影片，需自行架設 [Local Bot API Server](https://github.com/tdlib/telegram-bot-api)，上限可到 2GB。

### 1. 到 my.telegram.org 申請 api_id 與 api_hash

1. 登入 https://my.telegram.org（用你自己的 Telegram 帳號，不是 bot）
2. 點選「API development tools」
3. 填寫 App title、Short name
4. 建立後取得 `api_id`（數字）與 `api_hash`（英數字串）

### 2. 用 Docker 啟動 Local Bot API Server

```bash
docker volume create telegram-bot-api-data

docker run --rm -v telegram-bot-api-data:/var/lib/telegram-bot-api alpine chown -R 101:101 /var/lib/telegram-bot-api

docker run -d --name telegram-bot-api \
  -p 8081:8081 \
  -v telegram-bot-api-data:/var/lib/telegram-bot-api \
  -e TELEGRAM_API_ID=你的api_id \
  -e TELEGRAM_API_HASH=你的api_hash \
  -e TELEGRAM_LOCAL=1 \
  aiogram/telegram-bot-api:latest
```

> **重要：** 請使用 Docker named volume（如上面的 `telegram-bot-api-data`），而不是掛載主機資料夾路徑。在 Windows 上掛載主機路徑會因為 Bot Token 內含冒號 `:` 而導致容器啟動失敗（NTFS 不允許路徑含冒號）。
>
> 另外，`chown -R 101:101` 這一步是必要的：這個 image 內部以非 root 使用者（UID 101）執行，若 volume 權限不正確，容器會在啟動時崩潰並顯示 `Failed to rename binlog ... No such file or directory`。

確認容器有正常啟動：

```bash
docker ps        # STATUS 應為 Up ...
docker logs telegram-bot-api   # 確認有 "Bot API server is listening on 0.0.0.0:8081"
```

### 3. 修改設定並重新啟動 bot

把 `yt_dlp_telegram_bot.py` 裡的：
```python
USE_LOCAL_API_SERVER = True
MAX_FILESIZE_MB = 2000
```

重新執行即可。

## 疑難排解

| 錯誤訊息 | 可能原因與解法 |
|---|---|
| `ModuleNotFoundError: No module named 'telegram'` | `pip install` 裝到了不同的 Python 環境。改用 `python -m pip install python-telegram-bot yt-dlp` 確保裝到執行時用的直譯器 |
| `Requested format is not available` | yt-dlp 版本過舊，執行 `pip install -U yt-dlp` 更新；YouTube 常改版，需要跟上版本 |
| `httpx.RemoteProtocolError: Server disconnected` | 連線到 Local Bot API Server 但對方無回應，先確認 `docker ps` 容器狀態是否正常 |
| Docker 容器 `Exited (1)`，log 顯示 `Failed to rename binlog...No such file or directory` | Volume 權限問題，執行上方的 `chown -R 101:101` 步驟 |
| Docker 容器 `Exited (1)`，其他原因 | 檢查 `TELEGRAM_API_ID` / `TELEGRAM_API_HASH` 是否正確填入且未殘留佔位字（如 `你的api_id`） |
| 影片下載成功但無法傳送，提示檔案過大 | 確認是否已啟用 Local Bot API Server，`USE_LOCAL_API_SERVER` 是否為 `True`，`MAX_FILESIZE_MB` 是否已調高 |
| Windows 無 `winget` 指令 | 到 Microsoft Store 更新 App Installer；或直接手動下載 ffmpeg 並加入系統 PATH |

## 常駐執行（選用）

### Windows

可搭配工作排程器（Task Scheduler）設定開機自動啟動，或建立 `.bat` 啟動腳本方便手動執行。

### Linux（systemd）

建立 `/etc/systemd/system/ytdlpbot.service`：

```ini
[Unit]
Description=YT-DLP Telegram Bot
After=network.target

[Service]
User=你的使用者名稱
WorkingDirectory=/path/to/project
ExecStart=/path/to/venv/bin/python yt_dlp_telegram_bot.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl enable ytdlpbot
sudo systemctl start ytdlpbot
```

## 注意事項

- 請只下載自己有權下載的內容，並遵守各平台的服務條款與著作權規定
- 若啟用 Local Bot API Server，請勿將 8081 埠開放至公開網路——任何能連到該埠的人都可以透過 Bot Token 完全控制你的 bot
- `yt-dlp` 建議定期更新（`pip install -U yt-dlp`），以因應各平台格式異動

## License

MIT（或依你的專案需求調整）
