# Ollama Colab Private Chat

[![Python](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/)
[![License](https://img.shields.io/badge/license-MIT-green.svg)](LICENSE)
[![Colab](https://img.shields.io/badge/platform-Google%20Colab-orange.svg)](https://colab.research.google.com/github/hiroaki-com/colab-ollama-server/blob/main/ollama_colab_private_chat_ja.ipynb)
[![Ollama](https://img.shields.io/badge/Ollama-compatible-blueviolet.svg)](https://ollama.com/)

[English](./README.EN.md) | [日本語](./README.md)

### 概要

Google Colab の GPU 上で Ollama を動作させ、同一ノートブック内でローカル LLM とプライベートにチャットできます。外部 API への送信なし、ログ保存なし。ブラウザリロードで即時消去されるステートレス設計です。

#### 通信フロー

**Upstream（リクエストフロー）**

```mermaid
flowchart LR
    subgraph UI_Layer ["Client UI Layer"]
        UI_D[Colab Cell UI<br>Direct]
        UI_T[Colab Cell UI<br>Tunnel]
        UI_S[Standalone<br>Browser UI]
    end
    subgraph Route_Direct ["1. Inline: Direct"]
        Kernel[Python Kernel]
    end
    subgraph Route_Tunnel ["2. Inline: Tunnel Backup"]
        Edge_1[Cloudflare Edge]
        CF_1[cloudflared]
    end
    subgraph Route_Standalone ["3. Standalone Mode"]
        Edge_2[Cloudflare Edge]
        CF_2[cloudflared]
        FastAPI[FastAPI Proxy 11435]
    end
    Ollama[Ollama Server<br>11434]
    %% Data Flow
    UI_D -- "invokeFunction<br>(Colab RPC)" --> Kernel
    Kernel -- "HTTP POST" --> Ollama
    UI_T -- "HTTPS fetch" --> Edge_1
    Edge_1 --> CF_1
    CF_1 -- "HTTP POST" --> Ollama
    UI_S -- "HTTPS fetch" --> Edge_2
    Edge_2 --> CF_2
    CF_2 --> FastAPI
    FastAPI -- "Proxy POST" --> Ollama
```

**Downstream（レスポンスフロー）**

```mermaid
flowchart RL
    Ollama[Ollama Server<br>11434]
    subgraph Route_Direct ["1. Inline: Direct"]
        Kernel["Python Kernel<br>(Job store / buffer)"]
    end
    subgraph Route_Tunnel ["2. Inline: Tunnel Backup"]
        CF_1[cloudflared]
        Edge_1[Cloudflare Edge]
    end
    subgraph Route_Standalone ["3. Standalone Mode"]
        FastAPI["FastAPI Proxy 11435"]
        CF_2[cloudflared]
        Edge_2[Cloudflare Edge]
    end
    subgraph UI_Layer ["Client UI Layer"]
        UI_D[Colab Cell UI<br>Direct]
        UI_T[Colab Cell UI<br>Tunnel]
        UI_S[Standalone<br>Browser UI]
    end
    %% Data Flow
    Ollama -. "NDJSON stream" .-> Kernel
    Kernel -. "Base64 delta + Status<br>(poll reply)" .-> UI_D
    Ollama -. "NDJSON stream" .-> CF_1
    CF_1 -. "Tunnel stream" .-> Edge_1
    Edge_1 -. "ReadableStream chunk<br>(TextDecoder)" .-> UI_T
    Ollama -. "NDJSON stream" .-> FastAPI
    FastAPI -. "StreamingResponse" .-> CF_2
    CF_2 -. "Tunnel stream" .-> Edge_2
    Edge_2 -. "ReadableStream chunk<br>(TextDecoder)" .-> UI_S
```

| 構成要素 | 役割 |
|:---|:---|
| **Chat UI — Inline** | Colab 出力セル内に描画。`Direct`（カーネル内部通信）と `Tunnel (Backup)` をタブで切替可能 |
| **Chat UI — Standalone** | ブラウザ別タブで動作する独立 UI。Cloudflare Tunnel 経由で他端末からもアクセス可能 |
| **Ollama Server** | `OLLAMA_HOST=0.0.0.0:11434` で起動。Flash Attention・KV キャッシュ最適化済み |
| **Cloudflare Tunnel** | 登録・トークン不要で公開 URL を発行。Standalone モードは専用ポート（11435）で FastAPI をプロキシ |
| **Model (VRAM)** | T4 GPU の VRAM にロード。起動時にウォームアップまで自動実行 |

#### 主な特徴

- 完全無料・外部 API 非依存・ローカル推論によるプライバシー保護
- ログを一切保存しないステートレス設計（ブラウザリロードで即時消去）
- Cloudflare Tunnel を使用するため、ngrok アカウント・トークン不要
- モデルを UI で選択し、プルからウォームアップまで自動実行
- Inline と Standalone の 2 つのチャットモードを選択可能
- メッセージのコピー・編集・リトライ、Markdown レンダリングをサポート

### クイックスタート

#### 実行環境

```
https://colab.research.google.com/github/hiroaki-com/colab-ollama-private-chat/blob/main/ollama_colab_private_chat_ja.ipynb
```

#### 基本的な実行手順

1. Google Colab でノートブックを開きます
2. Runtime > Change runtime type > **T4 GPU** を選択します
3. `Model Registry` セルでモデルリストを読み込み、起動するモデルを選択します
4. `Server` セルを実行します（初回はモデルのダウンロードに数分かかります）
5. チャットモードを選択して実行します
   - `Chat UI — Inline`：Colab 出力セル内でチャット
   - `Chat UI — Standalone`：ブラウザ別タブ・他端末からチャット

### チャットモード

#### Inline モード

Colab の出力セル内で動作します。通信方式をタブで切替可能です。

| モード | 説明 |
|:---|:---|
| **Direct**（推奨） | Colab カーネル内部通信を使用。高速で安定しています |
| **Tunnel (Backup)** | Cloudflare Tunnel 経由で接続。Direct が不安定な場合のバックアップ |

#### Standalone モード

ブラウザの別タブで動作する独立した UI です。FastAPI サーバーと専用 Cloudflare Tunnel が起動し、表示される URL にアクセスするだけで使用できます。スマートフォンなど他の端末への共有も可能です。

### モデル設定

`Model Registry` セルで、起動するモデルをカンマ区切りで指定します。

```python
model_list = "nemotron-3-nano:4b, ministral-3:3b, qwen3.5:4b"
num_ctx    = 4096
```

モデル名は [https://ollama.com/search](https://ollama.com/search) でご確認いただけます。

**T4 GPU 環境での推奨モデルサイズ**

| サイズ | パフォーマンス | 備考 |
|:---:|:---:|:---|
| 4B | 最速 | 推奨 |
| 8B | 高速 | 推奨 |
| 14B | 中速 | 実用範囲 |
| 20B+ | 低速 | 非推奨 |

### 技術スタック

- Runtime: Google Colab (Python 3.10+)
- LLM Engine: Ollama
- Tunnel: Cloudflare Tunnel (cloudflared)
- Standalone Server: FastAPI / uvicorn / httpx
- UI: ipywidgets · marked.js · DOMPurify

### ライセンス

MIT License. 詳細は [LICENSE](LICENSE) を参照してください。

### クレジット

- [Ollama](https://ollama.com/) — ローカル LLM 実行エンジン
- [Google Colab](https://colab.research.google.com/) — 無料 GPU 環境
- [Cloudflare Tunnel](https://developers.cloudflare.com/cloudflare-one/connections/connect-networks/) — 登録不要のセキュアトンネリング

### サポート

- バグ報告: [Issues](../../issues)
- 質問・議論: [Discussions](../../discussions)
