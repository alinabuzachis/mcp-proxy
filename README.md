# Secure MCP Gateway & Developer Scaffolding

This repository provides a "Secure-by-Design" proxy and scaffolding foundation for Model Context Protocol (MCP) servers. It exposes an OpenAI / Anthropic–compatible API backed by LiteLLM, while securely brokering requests to:

* Google Vertex AI (Claude, Gemini)
* AWS Bedrock (Claude)
* Local Ollama models

All authentication, auditing, and governance are centralized in the proxy — tools never need cloud credentials.

## Overview

Many organizations require proprietary MCP servers to maintain control over data and logic. This project simplifies that lifecycle by:

1. _Hiding Complexity_: Translates cloud-native authentication (OAuth, ADC, IAM) into standard API-key headers.

2. _Security First_: A single, auditable gateway for authentication, access control, logging, and policy enforcement.

3. _Developer DX_: Provides a "Golden Template" so developers can focus on building tools, not infrastructure.

## Architecture

```
┌──────────────┐
│ Tools / CLIs │  (Claude Code, MCP servers, agents)
└──────┬───────┘
       │  Authorization: Bearer PROXY_MASTER_KEY
       ▼
┌──────────────────────┐
│ LiteLLM MCP Gateway  │  (localhost:4000)
│  - Auth enforcement  │
│  - Model routing     │
│  - Audit logging     │
└──────┬───────────────┘
       │
       ├─ Vertex AI (ADC / OAuth)
       ├─ AWS Bedrock (IAM)
       └─ Ollama (local)

````

## Prerequisites

- Docker & Docker Compose
- One or more of:
    - Google Cloud SDK (for Vertex AI)
    - AWS CLI (for Bedrock)
    - Ollama (for local models)

## Setup

### 1. Authenticate with Google Cloud (Vertex AI)

Instead of managing service account JSON files, this setup uses Application Default Credentials (ADC):

```Bash
gcloud auth application-default login
```

This creates credentials under:

```Bash
~/.config/gcloud/
```

They are mounted read-only into the container.

### 2. Authenticate with AWS (Bedrock)

If you plan to use Bedrock models:

```Bash
aws sts get-caller-identity
```

If not configured yet:

```Bash
aws configure
```

Required IAM permissions:

```Bash
bedrock:InvokeModel
bedrock:InvokeModelWithResponseStream
```

Credentials are read from:

```Bash
~/.aws/
```

### 3. Configure Environment Variables

Create a `.env` file in the root directory:

```Bash
# GCP Configuration
GCP_PROJECT="your-project-id"
GCP_LOCATION="us-east5"

# Security: Generate a Master Key for your local tools
# Run: openssl rand -hex 16
PROXY_MASTER_KEY="sk-your-generated-key-here"

```

### 4. Launch the Gateway

```Bash
docker-compose up -d
```

The gateway is now running at `http://localhost:4000`.


## How to Use the Gateway

The gateway speaks OpenAI-compatible APIs.

### 1. List models

```Bash
curl http://localhost:4000/v1/models \
  -H "Authorization: Bearer $PROXY_MASTER_KEY"
```

## 2. Chat completion (curl)

```Bash
curl http://localhost:4000/v1/chat/completions \
  -H "Authorization: Bearer $PROXY_MASTER_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "model": "llama-local",
    "messages": [
      { "role": "user", "content": "What model are you?" }
    ]
  }'
```

### 3. Use with Claude Code
Point Claude Code at the proxy:

```Bash
export CLAUDE_CODE_BASE_URL="http://localhost:4000/v1"
export ANTHROPIC_API_KEY="$PROXY_MASTER_KEY"
```

Now Claude Code will:
1. Send Anthropic-style requests
2. Authenticate with the proxy
3. The proxy routes to Vertex / Bedrock / local models

No cloud keys required.

### 4. Use with OpenAI-compatible SDKs

```Python
from openai import OpenAI
import os

client = OpenAI(
    base_url="http://localhost:4000/v1",
    api_key=os.environ["PROXY_MASTER_KEY"]
)

resp = client.chat.completions.create(
    model="llama-local",
    messages=[{"role": "user", "content": "Hello"}]
)

print(resp.choices[0].message.content)
```

## License

This project is licensed under the GPL-3.0 License.