---
title: "Ollama + MCP Client 붙이기 - 1"
date: 2025-07-07 18:00:00 +0900
categories: [개발]
tags: [Ollama, MCP, Notion, LLM, Python]
author: L.J
---

### 로컬 Ollama에 MCP 붙이기

Claude, VSCode, Cursor 등은 자체적으로 MCP Client를 제공해주기 때문에 간단하게 원하는 MCP를 붙여 사용할 수 있지만, 로컬 llama는 안타깝게도 MCP Client가 없다. 때문에 직접 만들어주거나 남들이 만들어둔 MCP Client를 사용하는 방법이 있다.

### MCP Client 동작 방식

1. **요청 수신**: AI 모델 또는 Host가 요청
2. **요청 파싱 및 매핑**: MCP 표준 포맷(JSON-RPC)으로 변환
3. **MCP Server로 요청 전송**: 비동기 방식
4. **응답 수신 및 역매핑**: 응답을 AI 모델이 이해할 수 있는 형식으로 변환
5. **응답 전달**: 최종적으로 결과 전달

MCP를 사용할 때 더 정확한 출력을 위해 llama3 모델에서 llama3.1:8b로 교체했다.

### Python Code

GitHub: [https://github.com/Lajancia/mcp-ollama](https://github.com/Lajancia/mcp-ollama)

`MCPClient` 클래스는 MCP 설정 파일을 로드하고, 서버를 시작하며, 도구 목록을 가져오고, 도구를 호출하는 기능을 수행한다.

`OllamaMCPBridge` 클래스는 Ollama와 MCP를 연결하는 브리지 역할을 한다.

주요 흐름:
1. `ask_ollama()`에서 원하는 포맷을 예시로 주며 해당 포맷 형태로 return하도록 요청
2. Ollama 응답을 파싱하여 도구와 인자 추출
3. MCP Server로 API 요청 전송

```python
def ask_ollama(self, prompt: str, tools_info: str = "") -> str:
    full_prompt = f"""
    사용자 요청: {prompt}
    사용 가능한 Notion 도구들: {tools_info}
    위 도구들 중에서 사용자 요청을 처리하기 위해 적절한 도구를 선택하고,
    필요한 인자들을 JSON 형태로 제공해주세요.

    응답 형식:
    도구이름: [선택한 도구 이름]
    제목: [사용자가 요청한 제목]
    내용: [사용자가 요청한 내용]
    인자: {{"parent": {{"page_id": "PARENT_PAGE_ID"}}, ...}}
    """
    data = {"model": "llama3.1:8b", "prompt": full_prompt, "stream": False}
    response = requests.post(self.ollama_url, json=data)
    return response.json()["response"]
```

### 하지만 문제가 많다...

제대로 돌아가기는 하지만 아직 튜닝이 되지 않은 날것의 모델이고, Cursor나 VSCode에서 제공하는 MCP Client를 이길 수가 없다. 앞으로 깃허브와 figma 등 연결이 필요할 때마다 일일이 MCP Client를 만들다가는 튜닝은 어림도 없을 것 같았다.

다음 포스트에서는 dolphin-mcp 라이브러리를 장고 앱에 붙여보려고 한다.