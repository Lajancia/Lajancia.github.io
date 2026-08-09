---
title: "MCP 도전기 - 바이브 코딩"
date: 2025-06-22 18:00:00 +0900
categories: [개발]
tags: [MCP, Notion, Cursor, VSCode, Copilot, 바이브 코딩]
author: L.J
---

## 바이브 코딩 바이브 코딩

지난 시간에 바이브 코딩이 무엇인지에 대해 알아보았고, 이제 실전으로 사용해볼 차례이다. 특히 MCP를 사용한 개발을 진행하여, 아침에 읽을 개발자 뉴스를 Notion에 한국어로 정리해주는 서비스를 만들어보려고 한다.

## Notion MCP

[https://developers.notion.com/docs/mcp](https://developers.notion.com/docs/mcp)

Cursor와 같은 AI에서 생성한 결과를 Notion으로 옮기는 과정을 MCP에게 맡길 수 있게 되었다.

[https://github.com/makenotion/notion-mcp-server](https://github.com/makenotion/notion-mcp-server)

Cursor를 사용해서 MCP를 작성할 예정이다. `.cursor/mcp.json` 폴더와 파일을 생성하고, Notion MCP 연결을 위한 API 키를 발급받으면 된다.

Cursor에게 특정 URL에 대한 내용을 정리해서 Notion에 페이지를 생성해달라고 했다.

```
## JavaScript Weekly #741 주요 뉴스
- Exploring JavaScript (ES2025 Edition) 출시: Dr. Axel Rauschmayer의 최신 책
- Biome v2 공개: 최초의 타입 인식 JS/TS 린터
- Bun v1.2.16 업데이트: 파일 라우팅 지원
- React Native 0.80 출시: React 19.1 지원
```

진짜 간단하게 핵심만 요약해서 정리해주었다. 그런데 이게 왠걸, Cursor가 다운되었다. SNS를 확인해보니 나 뿐만 아니라 모두가 커서 다운을 외치고 있었다.

이래서는 써먹지를 못한다. Cursor의 치명적인 문제는 인터넷이 연결되어있지 않으면 아무것도 할 수가 없다는 것이다.

### VS Code + MCP + Copilot

[https://code.visualstudio.com/mcp](https://code.visualstudio.com/mcp)

VSCode도 정식으로 MCP를 지원하게 되면서 Cursor에서 사용했던 API 키를 그대로 VSCode에 붙인 뒤, Copilot을 agent로 교체하면 Cursor와 유사하게 동작한다.

Copilot에게 이번주 React와 JavaScript 관련 이슈를 정리해달라 요청한 결과, React 생태계 피로감과 Next.js 15의 Vercel 의존성 문제 등이 잘 정리되었다.

Cursor가 다시 정상적으로 돌아왔지만, 14일 뒤 프로 버전 지원이 종료되면 어디까지 가능할지에 따라 계속 Cursor를 사용할지, VSCode를 사용할지 결정해야 할 것 같다.