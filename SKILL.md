---
name: colab-notebook
description: >-
  Google Colab 上で AI モデル推論エンジンやマイクロサービス Studio を構築・運用するノートブック作成規約と設計パターン。
  設定の外部化 (.env によるシークレット一元管理)、単一責務コードブロック原則、%%writefile による宣言的ファイル生成、
  API 単位のファイル分離、モデルロード/実行時エラーハンドリング、サーバー & トンネル非同期トレースログ出力などの標準ベストプラクティスを定義。
---

# colab-notebook

本スキルは、Google Colab 上で AI モデル（vLLM, Ollama, TTS, HyperFrames 等）の推論サーバーやマイクロサービス Studio ノートブックを開発・保守する際の厳格な標準規約です。

---

## 1. コア原則 (Core Principles)

### 原則 1: 単一責務コードブロック (Single-Responsibility Code Cells)
- **1セル＝1処理**: 相互に関係のない処理を同一のコードセルに混在させることを禁止する。
- **ファイル生成の分離**: 設定、ロガー、状態管理、各APIルーター、メインサーバーは、セルを1つずつ完全に分けて記述する。
- **メリット**: Colab 上でステップごとの進捗が可視化され、異常終了時にどの処理・ファイル生成で失敗したかが一目で判明する。

### 原則 2: 設定の完全外部化 (`.env` による環境変数一元管理)
- **ベンダーロックインの排除（Colab Secret API の廃止）**:
  - Google Colab 固有の `google.colab.userdata`（Secrets サービス）への依存はベンダーロックインを引き起こし、ローカル環境、Kaggle、クラウド VM（RunPod, Lambda 等）への移植性を著しく損なうため**廃止・使用禁止**とする。
  - すべての認証情報・シークレットは標準的な `.env` ファイルに記述し、`python-dotenv` または標準環境変数経由でアプリケーションへ注入する構成に統一する。
- **必須指定トークン・シークレット**:
  `.env` ファイルには、以下の3つの認証トークンを**必ず指定**する:
  1. `HF_TOKEN`: Hugging Face アクセストークン（ゲーテッドモデルや非公開リポジトリのダウンロードに必須）
  2. `LLM_API_KEY`: 外部クライアントや OpenAI 互換 API 呼び出し時の認証 API キー
  3. `TUNNEL_TOKEN`: Cloudflare Tunnel 認証トークン（固定トンネル確立・セキュアな外部公開に必須）
- **設定値の網羅**:
  - トークン類に加え、ポート番号、ホスト、モデル名、VRAM割り当て率、最大コンテキスト長などの可変値もすべて `.env` に集約する。
  - ノートブック内のソースコードやセル内にシークレットや認証情報を直接ハードコードすることを厳格に禁止する。

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

### 原則 5: サーバー & Cloudflare Tunnel ログの非同期コンソール出力
- **FastAPI / 推論エンジンのトレースログ**:
  - 重要な処理チェックポイント (`CHECKPOINT`)
  - API リクエスト/レスポンス要約および所要時間 (`API_IO`)
  - 実行時例外や警告 (`ERROR`)
  これらを非同期キュー経由でスレッドセーフに Colab コンソールへフラッシュ出力し、リアルタイム監視セルで追跡可能にする。
- **Cloudflare Tunnel (`cloudflared`) ログの非同期出力**:
  - バックグラウンド実行される `cloudflared` の出力（接続確立、トンネルURL発行、ヘルスチェック、切断/再試行イベント等）をログファイル（例: `logs/tunnel.log`）に保存するとともに、非同期で Colab コンソールへストリーム出力可能にする。
  - これにより、トンネル切断やネットワークエラー発生時に Colab 画面上で即座に検知・トラブルシューティングできるようにする。

### 原則 6: 隔離された仮想環境 (`uv venv`) への依存インストール
- **Colab OS グローバル Python へのインストール禁止**: Colab ホスト環境のシステム Python に対する `!pip install` やグローバルパッケージ導入を**厳格に禁止**する。CUDA バージョンの不整合や依存パッケージの衝突、ランタイム破壊を防ぐため。
- **`uv venv` による専用仮想環境の作成**: 作業ディレクトリ直下に高速パッケージマネージャー `uv` を用いて独立環境 `uv venv .venv` を作成する。
- **`.venv` 配下へのライブラリ導入**: vLLM、FastAPI、Uvicorn などのすべての Python 依存ライブラリは、必ず仮想環境を対象としてインストールする（例: `!uv pip install --python /path/to/.venv/bin/python ...`）。
- **プロセスの隔離実行**: サーバー起動時も Colab のシステム Python ではなく、必ず `.venv/bin/python` または `.venv/bin/uvicorn` を明示してバックグラウンド実行する。

### 原則 7: `%%writefile` による宣言的ファイル生成（Python 変数代入の禁止）
- **変数の使用禁止**: ソースコードを Python 文字列変数（`code = """..."""` やヒアドキュメント）に代入し、`open().write()` 等でファイル書き出しを行うことを**厳格に禁止**する。
- **セル先頭の `%%writefile` 必須**: モジュールやスクリプトを生成する際は、コードセルの1行目に必ず IPython マジックコマンド `%%writefile <出力先パス>` を記述する。
- **メリット**:
  - Colab エディタ上で完全なシンタックスハイライト、コード補完、インデント保持が有効になる。
  - 文字列エスケープ事故（`\n`, `\"`, `{var}` の f-string 展開等）を根本から防止できる。
  - セルを実行するだけで直感的にファイルが作成・更新され、可読性と保守性が大幅に向上する。

### 原則 8: 既存プロセスの事前強制解放（起動の冪等性保証）
- **ポート競合エラー (`Address already in use`) の根絶**:
  - Colab のセルを再実行した際、前回立ち上げた Uvicorn、FastAPI、または `cloudflared` がバックグラウンドでポート（例: 8008）を占有し続け、サーバー起動に失敗するトラブルを防止する。
- **事前強制終了の義務付け**:
  - バックグラウンドサーバー起動セルおよびトンネル起動セルの実行冒頭において、必ず対象ポート・対象プロセスの事前解放コマンドを実行する。
  - 例: `!fuser -k -n tcp {PORT} 2>/dev/null || true`、`!pkill -9 -f uvicorn || true`、`!pkill -9 -f cloudflared || true`
  - これにより、何度セルを実行しても安全にクリーンな状態で再起動できる「冪等性」を保証する。

### 原則 9: 一時作業領域 & キャッシュの自動クリーンアップ（ディスク枯渇防止）
- **Colab ディスク枯渇（Disk Full）の防止**:
  - Colab インスタンスの限られたディスク容量（70〜100GB）において、大容量モデルの重み、一時動画・音声ファイル、顔画像キャッシュ、パッケージマネージャーのキャッシュによるディスク圧迫を防ぐ。
- **API 処理後の一時ファイル削除**:
  - ダウンロードした素材や一時変換ファイルは、処理完了時および例外発生時にも確実に `try-finally` ブロック内でクリーンアップ（`shutil.rmtree` や `os.unlink`）する。
- **キャッシュの定期ローテーション**:
  - パッケージインストール後の不要キャッシュ削除（例: `!uv cache clean`）や、古いログファイルの自動ローテーション（一定行数・容量超過時の破棄）を組み込み、長時間の安定稼働を維持する。

---

## 2. 標準ノートブック構造 (24ステップ・テンプレート)

```text
Markdown: タイトル・概要
Cell 01 [GPU確認]: !nvidia-smi
Cell 02 [システムリソース確認]: !free -h & !df -h
Cell 03 [設定外部化 (.env 作成)]: HF_TOKEN, LLM_API_KEY, TUNNEL_TOKEN 等を含む .env テンプレートの書き出し
Cell 04 [設定読み込み]: dotenv による環境変数展開（Colab Secret API に依存しない標準構成）
Cell 05 [OSパッケージ導入]: curl, psmisc 等の apt-get インストール
Cell 06 [cloudflared導入]: Cloudflare Tunnel CLI バイナリの配備
Cell 07 [uv仮想環境初期化]: uv の導入と独立 .venv の初期化
Cell 08 [Pythonライブラリ導入]: uv を用い .venv 配下に vllm, fastapi, uvicorn 等をインストール
Cell 09 [ディレクトリ初期化]: core/, routers/, logs/ の作成
Cell 10 [core/config.py]: %%writefile による共通設定・APIキー検証モジュール
Cell 11 [core/logger.py]: %%writefile による非同期トレースロガーモジュール
Cell 12 [core/state.py]: %%writefile によるサーバー状態・エラー履歴・GPUメトリクスモジュール
Cell 13 [core/engine.py]: %%writefile による推論エンジン（vLLM 等）の安全な非同期初期化
Cell 14 [routers/health.py]: %%writefile によるヘルスチェック API (GET /healthz)
Cell 15 [routers/status.py]: %%writefile によるステータス・詳細エラー確認 API (GET /api/v1/status)
Cell 16 [routers/chat.py]: %%writefile による推論 API (POST /v1/chat/completions)
Cell 17 [routers/models.py]: %%writefile によるモデル一覧 API (GET /v1/models)
Cell 18 [server.py]: %%writefile による FastAPI メインアプリケーション結合
Cell 19 [バックグラウンド起動]: 既存プロセスの事前解放 (fuser/pkill) と Uvicorn プロセスのバックグラウンド立ち上げ
Cell 20 [ロード待機ループ]: /api/v1/status をポーリングし準備完了を待機
Cell 21 [Cloudflare Tunnel起動]: 既存トンネルの事前解放と外部トンネル URL の発行・表示（ログ非同期記録）
Cell 22 [ステータスAPI検証]: ステータス確認の疎通確認
Cell 23 [推論テスト]: OpenAI 互換リクエストの実行とレスポンス確認
Cell 24 [トレースログ監視]: サーバー & Cloudflare Tunnel ログのリアルタイム統合表示
```
