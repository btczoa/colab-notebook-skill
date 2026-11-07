---
name: colab-studio-notebook
description: >-
  Google Colab 上で AI モデル推論エンジンやマイクロサービス Studio を構築・運用するノートブック作成規約と設計パターン。
  設定の外部化 (.env によるシークレット一元管理)、単一責務コードブロック原則、%%writefile による宣言的ファイル生成、
  API 単位のファイル分離、モデルロード/実行時エラーハンドリング、サーバー & トンネル非同期トレースログ出力などの標準ベストプラクティスを定義。
---

# Colab Studio Notebook Architecture & Design Standards

本スキルは、Google Colab 上で AI モデル（vLLM, Ollama, TTS, HyperFrames 等）の推論サーバーやマイクロサービス Studio ノートブックを開発・保守する際の厳格な標準規約です。

---

## 1. コア原則 (Core Principles)

### 原則 1: 単一責務コードブロック (Single-Responsibility Code Cells)
- **1セル＝1処理**: 相互に関係のない処理を同一のコードセルに混在させることを禁止する。
- **ファイル生成の分離**: 設定、ロガー、状態管理、各APIルーター、メインサーバーは、セルを1つずつ完全に分けて記述する。
- **メリット**: Colab 上でステップごとの進捗が可視化され、異常終了時にどの処理・ファイル生成で失敗したかが一目で判明する。

### 原則 2: 外部 `.env` への完全依存原則（Colab 内生成およびデフォルト値フォールバックの厳禁）
- **ノートブックと外部 `.env` ファイルの同時作成義務（ペアリング納品規約）**:
  - エージェントが新規に Colab ノートブック（`.ipynb`）を作成または納品する際は、**必ず同一ディレクトリ内に環境変数値の定義を格納した外部 `.env` ファイルを同時に作成しなければならない**。
  - これにより、ユーザーがノートブックと同時に `.env` を Colab へ配置すれば即座に稼働できる完全な実行環境を提供する。
- **ノートブック内での `.env` 生成の絶対禁止**:
  - ノートブック内部で `%%writefile .env`、Python 変数代入、ヒアドキュメント等を用いて `.env` の雛形・テンプレートファイルを自前生成することを**厳格に禁止**する。
  - Colab ノートブックは、ユーザーが事前にアップロードまたはストレージ連携（Google Drive 等）から配置した**「外部の `.env` ファイル」にのみ純粋に依存するコード**として実装しなければならない。
- **環境変数の初期定義・フォールバック（デフォルト値）の禁止**:
  - コードセルや Python スクリプト内で `os.getenv("KEY", "default_value")` のようにデフォルト値をハードコードしてフォールバックすることを**厳格に禁止**する。
  - すべての必要な設定値（モデル名、ポート、メモリ比率、各種トークン等）は、**必ず `.env` ファイルから直接取得**しなければならない。
- **厳格なファイル存在確認・必須キー検証と即時エラー停止**:
  - `.env` ファイルが存在しない場合、直ちに明瞭なエラーログ（例: `❌ エラー: .env ファイルが見つかりません。事前にランタイム直下に配置してください。`）を出力し、例外を送出して処理を即時中断する。
  - 必須設定項目（`HF_TOKEN`, `LLM_API_KEY`, `TUNNEL_TOKEN`, `MODEL_NAME`, `TUNNEL_PORT` 等）が取得できなかった（未定義または空文字）場合、**「どの環境変数が取得できなかったか」をエラーログに明記して即座に実行を停止**する。ダミー値で継続することは認められない。
- **ベンダーロックインの排除（Colab Secret API の廃止）**:
  - `google.colab.userdata` への依存は完全に廃止し、すべての環境変数・シークレットは `python-dotenv` 経由で外部 `.env` から注入する。

### 原則 3: API エンドポイントごとのファイル分離 (`routers/`)
- 全エンドポイントを単一の `server.py` に記述することを禁止する。
- モジュール構成標準:
  - `routers/health.py`: 死活監視用 (`GET /healthz`)
  - `routers/status.py`: モデル状態・詳細エラー・GPU/VRAM情報取得用 (`GET /api/v1/status`)
  - `routers/chat.py` (推論API): OpenAI互換チャット補完 (`POST /v1/chat/completions`)。ChatMLフォーマット変換、ストリーミング (`text/event-stream`)、非ストリーミング、および `<think>` タグ（推論トークン）の長文出力に対応。
  - `routers/models.py`: 利用可能モデル一覧 (`GET /v1/models`)
  - `routers/<feature>.py`: 各業務・専用推論ロジック (例: `routers/voice.py`, `routers/frames.py`)
  - `server.py`: 各ルーターの `include_router` のみを行う薄いエントリーポイント

### 原則 4: モデルロード & 実行時エラーの完全ハンドリング
- vLLM や Torch モデルのロード処理は `try-except` で例外を捕捉し、スタックトレースとエラーメッセージを状態管理オブジェクト（`ServerState`）に保持する。
- モデルロードエラーが発生しても FastAPI サーバー自体をクラッシュさせず、外部から `GET /api/v1/status` を叩くことで「何が原因で失敗したか」を把握できるようにする。
- **ポーリングによるロード待機ループ**: クライアントやノートブック側で `/api/v1/status` をポーリングし、`model_status == 'ready'`（または `failed`）になるまでVRAM消費状況をログ表示しながら安全に待機する。

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
  - **ホスト Python の動的自動検出**: Colab ランタイムの Python バージョン更新（3.10〜3.13等）に柔軟に追従するため、固定バージョン指定ではなく `current_py = sys.executable` を取得して作成する。
  - **`--system` 衝突警告の抑止 & 冪等性**: `!UV_SYSTEM_PYTHON=0 uv venv --allow-existing --python {current_py} {VENV_DIR}` を標準記法とし、Colab 環境変数による不要なフラグ干渉を防ぐ。
- **`.venv` 配下へのライブラリ導入**: vLLM、FastAPI、Uvicorn などのすべての Python 依存ライブラリは、必ず仮想環境を対象としてインストールする（例: `!uv pip install --python /path/to/.venv/bin/python ...`）。
- **プロセスの隔離実行**: サーバー起動時も Colab のシステム Python ではなく、必ず `.venv/bin/python` または `.venv/bin/uvicorn` を明示してバックグラウンド実行する。
  - **マルチプロセス競合防止**: vLLM 起動環境変数には `VLLM_WORKER_MULTIPROC_METHOD="spawn"` を設定し、CUDA 初期化のコンテキスト競合を防止する。

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
Cell 03 [外部 .env 検証 & 読み込み]: 外部 .env の存在確認、必須環境変数の厳格バリデーション（不足時エラー停止）
Cell 04 [OSパッケージ導入]: curl, psmisc 等の apt-get インストール
Cell 05 [cloudflared導入]: Cloudflare Tunnel CLI バイナリの配備
Cell 06 [uv仮想環境初期化]: uv の導入と独立 .venv の初期化
Cell 07 [Pythonライブラリ導入]: uv を用い .venv 配下に vllm, fastapi, uvicorn 等をインストール
Cell 08 [ディレクトリ初期化]: core/, routers/, logs/ の作成
Cell 09 [core/config.py]: %%writefile による外部 .env 厳格読み込み・APIキー検証モジュール
Cell 10 [core/logger.py]: %%writefile による非同期トレースロガーモジュール
Cell 11 [core/state.py]: %%writefile によるサーバー状態・エラー履歴・GPUメトリクスモジュール
Cell 12 [core/engine.py]: %%writefile による推論エンジン（vLLM 等）の安全な非同期初期化
Cell 13 [routers/health.py]: %%writefile によるヘルスチェック API (GET /healthz)
Cell 14 [routers/status.py]: %%writefile によるステータス・詳細エラー確認 API (GET /api/v1/status)
Cell 15 [routers/chat.py]: %%writefile による推論 API (POST /v1/chat/completions)
Cell 16 [routers/models.py]: %%writefile によるモデル一覧 API (GET /v1/models)
Cell 17 [server.py]: %%writefile による FastAPI メインアプリケーション結合
Cell 18 [バックグラウンド起動]: 既存プロセスの事前解放 (fuser/pkill) と Uvicorn プロセスのバックグラウンド立ち上げ
Cell 19 [ロード待機ループ]: /api/v1/status をポーリングし準備完了を待機
Cell 20 [Cloudflare Tunnel起動]: 既存トンネルの事前解放と外部トンネル URL の発行・表示（TUNNEL_TOKEN 活用）
Cell 21 [ステータスAPI検証]: ステータス確認の疎通確認
Cell 22 [推論テスト]: OpenAI 互換リクエストの実行とレスポンス確認
Cell 23 [トレースログ監視]: サーバー & Cloudflare Tunnel ログのリアルタイム統合表示 (tail -f)
```

---

## 3. 実践設計パターン集 (Implementation Best Practices)

### パターン A: OpenAI 互換 Chat ルーター (`routers/chat.py`)
- **ChatML フォーマット変換**: `<|im_start|>{role}\n{content}<|im_end|>\n` 構造の厳格な構築。
- **推論思考トークン (`<think>`) の配慮**: DeepSeek や Qwen 蒸留モデル等の推論プロセス出力に対応するため、`max_tokens` のデフォルト値を 2048 以上に確保。
- **デュアルレスポンス対応**: `stream=True` (Server-Sent Events) と `stream=False` (通常JSON) の完全サポート。

### パターン B: モデルロード待機ループ (`Cell 20`)
- **ポーリング制御**: `GET /api/v1/status` を 5秒間隔でポーリング（最大 10分〜15分待機）。
- **進捗と VRAM 可視化**: 各試行で `model_status`（loading / ready / failed）と `nvidia-smi` 経由の VRAM 使用量をコンソールに出力。
- **早期失敗検知**: `model_status == 'failed'` を検知した瞬間にループを中断し、`last_error` のスタックトレースを即座に表示。

### パターン C: リアルタイム・トレース監視 (`Cell 24`)
- **`tail -f` 方式の非同期ストリーミング**: `open(..., 'r')` と `seek(0, os.SEEK_END)` によるリアルタイムログ追跡。
- **重要タグの強調表示**: `CHECKPOINT`, `API_IO`, `POST /v1/chat/completions`, `ERROR` 等の特定タグをハイライト表示し、稼働状況を直感的に把握可能にする。

