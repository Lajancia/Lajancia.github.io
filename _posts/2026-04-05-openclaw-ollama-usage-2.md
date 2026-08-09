---
title: "Openclaw + Ollama 사용기 - 2"
date: 2026-04-05 18:00:00 +0900
categories: [개발]
tags: [Openclaw, Ollama, LiteLLM, AI Agent, LLM]
author: L.J
---

### 시작 전...

Ollama + Openclaw일 때는 문제가 없었는데, Ollama를 Litellm 으로 연결할 경우 응답이 중간에 유실되는 일이 자주 발생했다. 아직까지 원인을 파악하지 못했는데, 커뮤니티를 확인해보니 Openclaw -> Litellm -> Ollama로 통신하는 과정에서 로컬 Ollama가 텍스트를 제대로 처리하지 못하고 끊기는 것 같다. OpenRouter에서 연결한 모델들은 모두 문제가 없는데 유독 Ollama에서만 끊김 현상이 있으니, Ollama와 Litellm 연결 관련 이슈로 추정중이다.

### Openclaw 설치에 대하여

화려한 인스타와 유튜브의 Openclaw 사용기를 보다가 진짜 생태계로 뛰어들면, 생각보다 설치부터 쉽지 않다는 사실을 금방 깨달을 수 있다. 놀랍게도 Openclaw는 정말 다양한 이유로 동작하지 않는다. 같은 명령어를 여러번 반복하다 보면 어쩌다가 한 번 동작할 때가 있고, 그렇지 않을 수도 있으며, 잘 돌다가 안될 때가 있고, 안되다가 될 때가 있다.

우선 필자의 경우에는 공식 홈페이지의 repository를 통해 pnpm으로 실행했다.

[https://docs.openclaw.ai/install#from-source](https://docs.openclaw.ai/install#from-source)

### 채팅 앱

입력창은 Discord를 선택했다. @로 직접 이름을 참조해야만 응답하는 것이 마음에 들지 않아 참조가 없어도 바로 대답하도록 커스텀 했다. openclaw.json에 아래와 같이 설정하면 된다.

```json
"channels":{
  "discord":{
    ... 기존 설정
    "guilds":{
      "CHANNAL_ID":{
        ...기존 설정
        "requireMention":false
      }
    }
  }
}
```

### LLM 구성

여러 모델을 사용한 결과 현재 정착한 모델은, Stepfun 3.5 flash free와 qwen2-vl-7b-instruct 모델이다. 가장 성능이 좋고 빠른 Stepfun이 대부분의 텍스트와 코딩을 처리하지만 이미지 분석 기능이 없기 때문에 이 부분은 맥미니에서 로컬로 돌아가는 qwen2-vl-7b-instruct가 담당한다.

여러 모델을 동시에 사용할 경우 litellm을 사용하면 openclaw의 config 설정은 그대로 두고 litellm쪽에서만 모델을 교체하여 안정적인 운영이 가능하다.

### LiteLLM

최근 악성 코드 이슈가 크게 터졌던 LiteLLM이다. 현재는 패치된 버전이 나왔으니 해당 버전을 사용해야 한다. 문제가 되는 버전은 1.82.7, 1.82.8 버전이다. stable은 1.82.3.

LiteLLM은 Docker로 올렸다.

**docker-compose.yaml**

```yaml
services:
  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: ${POSTGRES_DB}
    volumes:
      - ./postgres_data:/var/lib/postgresql/data
  redis:
    image: redis:alpine
  litellm:
    build:
      context: .
      args:
        target: runtime
    image: docker.litellm.ai/berriai/litellm:main-stable
    volumes:
      - ./litellm-config.yaml:/app/config.yaml
      - /Users/Ghost/.openclaw/media/inbound:/app/openclaw_data
    command:
      - "--config=/app/config.yaml"
    ports:
      - "1234:4000"
    extra_hosts:
      - "host.docker.internal:host-gateway"
    environment:
      DATABASE_URL: ${DATABASE_URL}
      LITELLM_MASTER_KEY: ${LITELLM_MASTER_KEY}
      OPENROUTER_API_KEY: ${OPENROUTER_API_KEY}
      STORE_MODEL_IN_DB: "True"
    env_file:
      - .env
    depends_on:
      - db
```

**litellm-config.yaml**

```yaml
model_list:
  - model_name: m4-manager
    litellm_params:
      model: openai/stepfun/step-3.5-flash:free
      api_base: https://openrouter.ai/api/v1
      api_key: "os.environ/OPENROUTER_API_KEY"
      timeout: 300
  - model_name: m4-vision
    litellm_params:
      model: openai/qwen3-vl:4b
      api_base: http://host.docker.internal:11434/v1
      api_key: "os.environ/OPENROUTER_API_KEY"
      drop_params: True
      timeout: 6000
general_settings:
  master_key: "os.environ/LITELLM_MASTER_KEY"
```

### Openclaw.json

```json
{
  "models": {
    "mode": "replace",
    "providers": {
      "litellm-proxy": {
        "baseUrl": "http://127.0.0.1:1234",
        "apiKey": "LITELLM_KEY",
        "api": "openai-completions",
        "models": [
          {
            "id": "m4-manager",
            "name": "Daily Gemma",
            "input": ["text"],
            "contextWindow": 128000,
            "maxTokens": 8192
          },
          {
            "id": "m4-vision",
            "name": "Vision Qwen",
            "input": ["text", "image"],
            "contextWindow": 64000,
            "maxTokens": 8192,
            "api": "openai-completions"
          }
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "imageModel": {
        "primary": "litellm-proxy/m4-vision",
        "fallbacks": ["litellm-proxy/m4-vision"]
      },
      "models": {
        "litellm-proxy/m4-manager": {},
        "litellm-proxy/m4-vision": {}
      },
      "timeoutSeconds": 300,
      "maxConcurrent": 4,
      "subagents": {
        "maxConcurrent": 8
      }
    },
    "list": [
      {
        "id": "manager",
        "name": "Project Manager",
        "model": "litellm-proxy/m4-manager",
        "subagents": {
          "allowAgents": ["cayde", "sagira"]
        }
      },
      {
        "id": "cayde",
        "name": "Agent Cayde",
        "model": "litellm-proxy/m4-daily"
      },
      {
        "id": "sagira",
        "name": "Agent Sagira",
        "model": "litellm-proxy/m4-vision"
      }
    ]
  }
}
```

provider로 litellm-proxy를 하고 litellm에서 지정한 llm key를 등록하면 동작한다. 이제 모델이 바뀌면 간단하게 config쪽만 수정하면 openclaw는 그대로 두고 계속 사용할 수 있다.