# OpenClaw + Tailscale 運用マニュアル

Mac mini上のDocker（OrbStack）でOpenClawを稼働させ、Tailscale経由でiPhoneなどの外部端末から安全にアクセスするための手順書です。

## 1. 構成概要

* **Host:** Mac mini (macOS)
* **Container Runtime:** Docker (OrbStack)
* **Network:** Host Mode (Tailscaleとの親和性のため)
* **Auth:** Token (`fuji1234`) + Device Pairing (承認制)
* **Public URL:** `https://katsushimac-mini.tail7815e4.ts.net/`

---

## 2. 設定ファイルの作成

### 2.1 権限の修正

設定ファイルを作成する前に、作業ディレクトリの権限を現在のユーザー（ai_agent）に統一します。これを行わないと、コンテナ内のAIが書き込みエラーを起こして起動しません。

```bash
cd /Users/ai_agent/Workspace
sudo chown -R ai_agent:staff .
```

### 2.2 openclaw.json (アプリケーション設定)

**注意:** JSONファイル内にコメント（`//` や `#`）を含めると起動しないため、`.openclaw/openclaw.json` の内容をそのまま使用してください。

設定ファイルは `.openclaw/openclaw.json` に配置済みです。

### 2.3 docker-compose.yml (コンテナ構成)

`network_mode: host` を使用することで、Mac側のTailscaleからの転送をスムーズに受け取れるようにします。

**シークレットの管理:** API キーは `.env` ファイルで管理します。`.env.example` をコピーして `.env` を作成し、値を設定してください。

```bash
cp .env.example .env
# .env を編集して ANTHROPIC_API_KEY を設定
```

---

## 3. 起動と公開

### 3.1 コンテナの起動

```bash
sudo docker-compose up -d --force-recreate
```

### 3.2 Tailscale Serve (HTTPS公開)

Mac側のTailscaleを使って、HTTPSアクセスをローカルのOpenClawポート(18789)へ転送します。これは一度設定すれば永続化されます。

```bash
/Applications/Tailscale.app/Contents/MacOS/Tailscale serve --bg https / http://127.0.0.1:18789
```

---

## 4. 新しい端末（iPhone等）の承認手順

セキュリティ設定により、新しい端末からのアクセスは「承認（Pairing）」が必要です。

1. **iPhoneでアクセスする**

   URL: `https://katsushimac-mini.tail7815e4.ts.net/?token=fuji1234`

   画面: 「Pairing Required」や接続エラーが表示されます（これでリクエストが飛びます）。

2. **MacのターミナルでリクエストIDを確認**

   ```bash
   sudo docker exec -it openclaw_secure node dist/index.js devices list
   ```

   Pending（保留中）の欄にあるID（例: `e4ae0c42-...`）をコピーします。

3. **承認コマンドを実行**

   ```bash
   sudo docker exec -it openclaw_secure node dist/index.js devices approve <コピーしたID>
   ```

4. **iPhoneで再読み込み**

   ページをリロードするとチャット画面に入れます。

---

## 5. 管理・トラブルシューティング

### 承認済み端末を削除する（アクセス禁止）

紛失や不正アクセスの疑いがある場合に使用します。

1. `devices list` で Paired 欄のIDを確認。
2. 以下のコマンドで削除。

```bash
sudo docker exec -it openclaw_secure node dist/index.js devices remove <デバイスID>
```

### 設定変更後の再起動

設定ファイル (`openclaw.json` 等) を書き換えた後は、必ず再起動が必要です。

```bash
sudo docker-compose restart openclaw
```

### ログの確認

エラーが出た場合、リアルタイムログを確認します。

```bash
sudo docker-compose logs -f
```
