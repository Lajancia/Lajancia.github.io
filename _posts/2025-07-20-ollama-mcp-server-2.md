---
title: "Ollama + MCP Server 붙이기 - 2"
date: 2025-07-20 18:00:00 +0900
categories: [개발]
tags: [Ollama, MCP, Django, dolphin-mcp, LLM]
author: L.J
---

### 지난 포스트...

일전에 MCP client를 구현해서 notion을 붙인 일이 있었다. 이걸 Django에서도 사용하면 좋을 것 같아, ollama와 Django를 같이 사용해서 채팅앱을 만들었던 프로젝트에 MCP Server를 자유롭게 붙일 수 있도록 수정했다. dolphin-mcp를 통해 진행했다.

#### MCP config

```json
{
  "models": [
    {
      "title": "llama3.1:8b",
      "provider": "ollama",
      "model": "llama3.1:8b",
      "default": true,
      "base_url": "http://ollama:11434"
    }
  ],
  "mcpServers": {
    "search-stock-news-mcp": {
      "command": "npx",
      "args": ["-y", "search-stock-news-mcp@latest"]
    },
    "weather": {
      "command": "npx",
      "args": ["-y", "@timlukahorstmann/mcp-weather"]
    },
    "ddg-search": {
      "command": "npx",
      "args": ["-y", "duckduckgo-mcp-server"],
      "transport": "stdio"
    }
  }
}
```

3가지 MCP 서버: 주식, 날씨, 검색 기능. 주식과 날씨 MCP는 API키가 필요하다.

#### Django + Dolphin-mcp

```python
from dolphin_mcp import run_interaction
import asyncio, json, subprocess
from datetime import date
from pathlib import Path
import os
from dotenv import load_dotenv
load_dotenv()

DEFAULT_LOCATION = "Seoul, South Korea"
CONFIG_PATH = Path("mcp_config.json")
MCP_CFG = json.loads(CONFIG_PATH.read_text())["mcpServers"]
CHAT_HISTORY = []
```

dolphin-mcp에서 API키가 제대로 주입되지 않는 이슈가 있어 직접 환경변수를 합치도록 수정했다.

```python
async def call_tool(tool_call: dict) -> str:
    name = tool_call["name"]
    server_key = name.split("-", 1)[0]
    cfg = MCP_CFG.get(server_key)
    if not cfg:
        return f"[Error] No MCP server for '{server_key}'"
    merged_env = {**os.environ, **cfg.get("env", {})}
    cmd = [cfg["command"], *cfg.get("args", [])]
    proc = subprocess.Popen(
        cmd, stdin=subprocess.PIPE, stdout=subprocess.PIPE,
        stderr=subprocess.PIPE, text=True, env=merged_env
    )
    payload = json.dumps(tool_call) + "\n"
    out, err = proc.communicate(payload)
    if err:
        return f"[Tool Error] {err.strip()}"
    try:
        resp = json.loads(out)
        return resp.get("content") or resp.get("results") or out.strip()
    except json.JSONDecodeError:
        return out.strip()
```

Ollama는 실시간 날짜/시간을 가져올 수 없기 때문에 프롬프트에 항상 현재 날짜를 포함시켰다.

```python
async def chat_once(user_input):
    today = date.today().isoformat()
    system_preface = (
        f"[System] Today's date is {today}. Your default location is {DEFAULT_LOCATION}. "
        "You're name is Neo. "
        "If user asks in korean, answer in korean."
    )
    combined_query = system_preface + "\n\n" + user_input
    raw = await run_interaction(
        user_query=combined_query,
        model_name="llama3.1:8b",
        config_path=str(CONFIG_PATH),
        quiet_mode=False
    )
    # tool call 결과를 다시 Llama에 피드백
```

dolphin-mcp가 MCP Server를 인식하고 Ollama가 tool call을 하도록 요청하면 제대로 응답하는 것을 확인했다. 하지만 여전히 영어로 작성해야 제대로 답변하고, 한 번에 하나의 tool만 사용하는 문제가 있다.