---
name: colab-notebook
description: >-
  Google Colab 上で AI モデル推論エンジンやマイクロサービス Studio を構築・運用するノートブック作成規約と設計パターン。
  設定の外部化 (.env)、単一責務コードブロック原則、API 単位のファイル分離、モデルロード/実行時エラーハンドリング、
  非同期トレースログ出力、Cloudflare Tunnel 外部連携などの標準ベストプラクティスを定義。
---

# colab-notebook

本スキルは、Google Colab 上で AI モデル（vLLM, Ollama, TTS, HyperFrames 等）の推論サーバーやマイクロサービス Studio ノートブックを開発・保守する際の厳格な標準規約です。

---

## 1. コア原則 (Core Principles)

### 原則 1: 単一責務コードブロック (Single-Responsibility Code Cells)
- **1セル＝1処理**: 相互に関係のない処理を同一のコードセルに混在させることを禁止する。
- **ファイル生成の分離**: 設定、ロガー、状態管理、各APIルーター、メインサーバーは、セルを1つずつ完全に分けて記述する。
- **メリット**: Colab 上でステップごとの進捗が可視化され、異常終了時にどの処理・ファイル生成で失敗したかが一目で判明する。

### 原則 2: 設定の完全外部化 (`.env` & Colab Secrets)
- すべての可変値（ポート番号、ホスト、トークン、モデル名、VRAM割り当て率、最大コンテキスト長など）は `.env` ファイルに定義する。
- Colab の環境変数 `google.colab.userdata` をフォールバックとしてシームレスに参照できるように実装する。
- ノートブック内にシークレットやハードコードされた認証情報を直接書き込まない。

### 原則 3: API エンドポイントごとのファイル分離 (`routers/`)
- 全エンドポイントを単一の `server.py` に記述することを禁止する。
- モジュール構成標準:
  - `routers/health.py`: 死活監視用 (`GET /healthz`)
  - `routers/status.py`: モデル状態・詳細エラー・GPU/VRAM情報取得用 (`GET /api/v1/status`)
  - `routers/<feature>.py`: 各業務・推論ロジック (例: `routers/chat.py`, `routers/voice.py`)
  - `server.py`: 各ルーターの `include_router` のみを行う薄いエントリーポイント

### 原則 4: モデルロード & 実行時エラーの完全ハンドリング
- vLLM や Torch モデルのロード処理は `try-except` で例外を捕捉し、スタックトレースとエラーメッセージを状態管理オブジェクト（`ServerState`）に保持する。
- モデルロードエラーが発生しても FastAPI サーバー自体をクラッシュさせず、外部から `GET /api/v1/status` を叩くことで「何が原因で失敗したか」を把握できるようにする。

### 原則 5: 非同期トレースログの Colab コンソール出力
- 重要な処理チェックポイント (`CHECKPOINT`)
- API リクエスト/レスポンス要約および所要時間 (`API_IO`)
- 実行時例外や警告 (`ERROR`)
これらを非同期キュー経由でスレッドセーフに Colab コンソールへフラッシュ出力し、リアルタイム監視セルで追跡可能にする。

### 原則 6: 隔離された仮想環境 (`uv venv`)
- Colab システムのグローバル Python 環境を破壊しないよう、作業ディレクトリ配下に `uv venv .venv` で独立環境を作成し、その中の Python/Uvicorn からバックグラウンド起動する。

---

## 2. 標準ノートブック構造 (24ステップ・テンプレート)

```text
Markdown: タイトル・概要
Cell 01 [GPU確認]: !nvidia-smi
Cell 02 [システムリソース確認]: !free -h & !df -h
Cell 03 [設定外部化 (.env 作成)]: デフォルト .env テンプレートの書き出し
Cell 04 [設定読み込み]: dotenv & google.colab.userdata 展開
Cell 05 [OSパッケージ導入]: curl, psmisc 等の apt-get インストール
Cell 06 [cloudflared導入]: Cloudflare Tunnel CLI バイナリの配備
Cell 07 [uv仮想環境初期化]: uv の導入と独立 .venv の初期化
Cell 08 [Pythonライブラリ導入]: vllm, fastapi, uvicorn 等のインストール
Cell 09 [ディレクトリ初期化]: core/, routers/, logs/ の作成
Cell 10 [core/config.py]: 共通設定・APIキー検証モジュール
Cell 11 [core/logger.py]: 非同期トレースロガーモジュール
Cell 12 [core/state.py]: サーバー状態・エラー履歴・GPUメトリクスモジュール
Cell 13 [core/engine.py]: 推論エンジン（vLLM 等）の安全な非同期初期化
Cell 14 [routers/health.py]: ヘルスチェック API (GET /healthz)
Cell 15 [routers/status.py]: ステータス・詳細エラー確認 API (GET /api/v1/status)
Cell 16 [routers/chat.py]: 推論 API (POST /v1/chat/completions)
Cell 17 [routers/models.py]: モデル一覧 API (GET /v1/models)
Cell 18 [server.py]: FastAPI メインアプリケーション結合
Cell 19 [バックグラウンド起動]: Uvicorn プロセスのバックグラウンド立ち上げ
Cell 20 [ロード待機ループ]: /api/v1/status をポーリングし準備完了を待機
Cell 21 [Cloudflare Tunnel起動]: 外部トンネル URL の発行と表示
Cell 22 [ステータスAPI検証]: ステータス確認の疎通確認
Cell 23 [推論テスト]: OpenAI 互換リクエストの実行とレスポンス確認
Cell 24 [トレースログ監視]: 直近ログのリアルタイム表示
```
