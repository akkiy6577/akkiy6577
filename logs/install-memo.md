# OpenClaw 2026 導入メモ — トラブルと解決策

> 2026年3月、OpenClaw 2026 を Windows 11 + Node.js v24 環境に導入した際に発生したトラブルの記録。  
> 同じ沼にハマる人が減ることを願ってまとめた。

---

## 環境

| 項目 | 詳細 |
|------|------|
| OS | Windows 11 (x64) |
| Node.js | v24.13.0 |
| Shell | PowerShell 7 |
| OpenClaw | 2026 (latest) |

---

## トラブル①：ポート競合 (EADDRINUSE)

### 症状

```
Error: listen EADDRINUSE: address already in use :::3700
    at Server.setupListenHandle [as _listen2] (node:net:xxx)
```

`openclaw gateway start` を叩いた瞬間にこれ。  
Gateway が起動しない。

### 原因

ポート `3700` を別プロセスが掴んでいた。  
調べたら前回の起動が zombie プロセスとして残っていた。

```powershell
# ポートを使っているプロセスを確認
netstat -ano | findstr :3700
# → TCP  0.0.0.0:3700  LISTENING  PID 18472

# プロセスを特定
tasklist | findstr 18472
# → node.exe  18472  ...

# 強制終了
taskkill /PID 18472 /F
```

### 解決策

**① 残留プロセスを kill してから再起動**

```powershell
taskkill /IM node.exe /F
openclaw gateway start
```

**② config でポートを変更する（恒久対応）**

```powershell
openclaw gateway config.get
# gateway.port を 3700 → 3701 に変更
openclaw gateway config.patch
```

> 💡 **教訓：** `openclaw gateway stop` せずに端末を閉じると zombie が残る。  
> 必ず `openclaw gateway stop` で止めてから終了する癖をつけよう。

---

## トラブル②：APIキー設定が反映されない

### 症状

```
Error: 401 Unauthorized — invalid API key
```

`ANTHROPIC_API_KEY` を環境変数に設定したのに、OpenClaw が認識してくれない。

### 原因その1：PowerShell のスコープ問題

```powershell
# ❌ これはセッションローカル — 別プロセスには伝わらない
$env:ANTHROPIC_API_KEY = "sk-ant-..."

# ✅ システム全体に永続化する
[System.Environment]::SetEnvironmentVariable(
  "ANTHROPIC_API_KEY", "sk-ant-...", "User"
)
```

環境変数をセッションローカルで設定していたため、Gateway プロセス（別プロセス）に伝わっていなかった。

### 原因その2：設定ファイルの優先順位

OpenClaw は以下の順番で設定を読む：

```
1. openclaw.config.json  (最優先)
2. 環境変数
3. .env ファイル
```

`openclaw.config.json` に古い（空の）キーが残っていたため、環境変数より優先されてしまっていた。

```powershell
# config の中身を確認
openclaw gateway config.get

# 不正なキーを削除 → 環境変数で管理する場合は config から消す
openclaw gateway config.patch
```

### 解決策まとめ

```powershell
# 1. システム環境変数に永続化
[System.Environment]::SetEnvironmentVariable("ANTHROPIC_API_KEY", "sk-ant-xxxx", "User")

# 2. config.json 内の古いキーをクリア（あれば）
openclaw gateway config.patch

# 3. 新しいターミナルを開いてから Gateway を再起動
openclaw gateway restart
```

> 💡 **教訓：** APIキーは `openclaw gateway config.patch` で直接 config に書くのが一番確実。  
> 環境変数で管理する場合はシステムスコープで設定 → 新しいセッションで Gateway 起動。

---

## トラブル③：`tool-loop-detection` モジュール not found

### 症状

```
Cannot find module '.../openclaw/dist/tool-loop-detection-B1_rZFTj.js'
```

OpenClaw のアップデート後にツールが全て動かなくなった。

### 原因

アップデートが不完全だった（npm のキャッシュ問題）。  
ビルドハッシュ付きのファイル名が変わったのに古いキャッシュが残り、参照が壊れた。

### 解決策

```powershell
# npm キャッシュをクリアして再インストール
npm cache clean --force
npm install -g openclaw

# Gateway 再起動
openclaw gateway restart
```

> 💡 **教訓：** OpenClaw の更新後にモジュールエラーが出たら、まず `npm cache clean --force` + 再インストール。  
> `openclaw gateway update` だけでは不十分な場合がある。

---

## まとめ：導入チェックリスト

```markdown
- [ ] Node.js v20+ がインストールされている
- [ ] npm install -g openclaw 完了
- [ ] APIキーをシステム環境変数 or config に設定
- [ ] openclaw gateway start で起動確認
- [ ] openclaw status でエージェント疎通確認
- [ ] SOUL.md / USER.md / MEMORY.md を初期設定
```

---

## 参考

- [OpenClaw 公式ドキュメント](https://docs.openclaw.ai)
- [OpenClaw GitHub](https://github.com/openclaw/openclaw)
- [Community Discord](https://discord.com/invite/clawd)

---

<sub>by akkiy6577 · OpenClaw agent assisted · 2026-03</sub>
