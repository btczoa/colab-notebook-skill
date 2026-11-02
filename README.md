# colab-notebook-skill

`colab-notebook` スキルを別 PC やチーム環境に `npx skills add` で 1 コマンド導入するための配布用 Git リポジトリです。

---

## 概要

Google Colab 上で AI モデル（vLLM, Ollama, TTS, HyperFrames 等）の推論サーバーやマイクロサービス Studio ノートブックを開発・保守する際の厳格な標準規約・設計パターンです。

* **単一責務コードブロック**: 1セル＝1処理、設定・ロガー・APIルーターの生成セルを完全分離。
* **設定の完全外部化**: `.env` および Colab Secrets（`google.colab.userdata`）の統合利用。
* **API エンドポイント分離**: `routers/` 配下での疎結合な設計（health, status, chat 等）。
* **モデルロード & 実行時エラーの完全ハンドリング**: サーバーを落とさず状態を保持・監視可能。
* **独立仮想環境**: `uv venv` による Colab システム環境の保護。

---

## インストール方法

### 1. GitHub リポジトリから直接インストール
```bash
npx skills add btczoa/colab-notebook-skill
```

または明示的にスキル名を指定する場合：
```bash
npx skills add btczoa/colab-notebook-skill --skill colab-notebook
```

### 2. ローカル環境からの検証・追加
```bash
npx skills add ./path/to/colab-notebook-skill
```

---

## インストール後の確認

```bash
# スキル一覧の確認
npx skills list

# スキルの実行確認
npx skills run colab-notebook
```

---

## ディレクトリ構成

```text
colab-notebook-skill/
├── README.md
├── skill.json                     # スキル定義マニフェスト
├── SKILL.md                       # ルート定義
└── skills/
    └── colab-notebook/            # スキル本体
        └── SKILL.md               # 24ステップ構成・設計規約定義
```
