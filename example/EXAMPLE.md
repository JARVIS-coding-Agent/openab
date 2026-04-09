# OpenAB Docker Compose 使用說明

## 概述

此 docker-compose 配置啟動兩個 AI 助手服務：
- **kiro**: OpenAB 的 Kiro 助手
- **claude**: OpenAB 的 Claude 助手
## 目錄結構

```
example/
├── data/
│   ├── home/
│   │   ├── agent/          # Kiro 的 home 目錄
│   │   └── node/           # Claude 的 home 目錄
│   │       └── .claude/
│   │           └── CLAUDE.md  # Claude 角色設定（佐助）
│   └── config/
│       ├── kiro/
│       │   └── config.toml    # Kiro 配置
│       └── claude/
│           └── config.toml    # Claude 配置
└── docker-compose.yml
```

## 啟動服務

```bash
docker-compose up -d
```

## 認證登入
### kiro
```bash
docker-compose exec -it kiro kiro-cli login --use-device-flow
```
### Claude
```bash
docker-compose exec -it claude setup-token
```
## 停止服務

```bash
docker-compose down
```

## 查看日誌

```bash
# 查看所有服務
docker-compose logs -f

# 查看特定服務
docker-compose logs -f kiro
docker-compose logs -f claude
```

## 重啟服務

```bash
docker-compose restart
```

## 配置設定

### Discord Bot 設定

啟動前必須修改以下配置文件：

**Kiro 配置** (`./example/data/config/kiro/config.toml`)：
```toml
[discord]
bot_token = "YOUR_KIRO_BOT_TOKEN"           # 替換為你的 Kiro Discord Bot Token
allowed_channels = ["YOUR_CHANNEL_ID"]      # 替換為允許的頻道 ID
```

**Claude 配置** (`./example/data/config/claude/config.toml`)：
```toml
[discord]
bot_token = "YOUR_CLAUDE_BOT_TOKEN"         # 替換為你的 Claude Discord Bot Token
allowed_channels = ["YOUR_CHANNEL_ID"]      # 替換為允許的頻道 ID
```

### 角色設定

Claude 服務使用宇智波佐助的角色設定，配置文件位於：
`./example/data/home/node/.claude/CLAUDE.md`

修改配置後需重啟對應服務：
```bash
docker-compose restart kiro    # 重啟 Kiro
docker-compose restart claude  # 重啟 Claude
```

## 環境變數

兩個服務都設定了 `RUST_LOG=debug` 以輸出詳細日誌。

## 持久化數據

所有數據都掛載到本地 `./example/data/` 目錄，容器重啟後數據不會丟失。
