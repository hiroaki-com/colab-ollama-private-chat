# colab-ollama-private-chat
Provides a stateless chat that connects to an Ollama server running on Google Colab, leaving no conversation history on either the browser or the server.


Upstream (リクエストフロー)
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

Downstream (レスポンスフロー)

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
