---
title: "Ollama LLM 사용기"
date: 2025-06-29 18:00:00 +0900
categories: [개발]
tags: [Ollama, LLM, llama3, 음성인식, Django]
author: L.J
---

### 나만의 개인 LLM을 가지고 싶었다.

누구나 아이언맨의 자비스 같은 AI 비서를 꿈꾸지 않을까? 그리고 무엇보다 '무료로' 사용할 수 있는 AI.

M3 칩 512G 저장공간, 36 메모리를 사용중이다. 여유롭게 사용하고 싶어 llama3를 사용했다.

```bash
ollama run llama3
```

생각보다 속도가 빠른 것에 놀라웠다. 하지만 질문할 때 해당 질문에 대한 것만 기억하고, 이전에 했던 질문을 기억하는 기능은 없다.

### 계획 짜기

넣고 싶은 기능:
1. 웹 검색 기능
2. 나와의 대화를 기억하기
3. 말투나 성격 제어
4. 음성으로도 소통 가능하게 만들기

#### 음성 제어 코드

Python을 통해 llama3가 음성으로 답변하는 코드를 작성했다.

```python
import requests
import speech_recognition as sr
import asyncio
import edge_tts
from langdetect import detect

OLLAMA_URL = "http://localhost:11434/api/generate"
OLLAMA_MODEL = "llama3"
NEO_PROMPT = "너의 이름은 Neo야."

def ask_ollama(prompt):
    full_prompt = f"{NEO_PROMPT}\n\n사용자: {prompt}\nNeo:"
    data = {"model": OLLAMA_MODEL, "prompt": full_prompt, "stream": False}
    response = requests.post(OLLAMA_URL, json=data)
    return response.json().get("response", "답변을 가져오지 못했습니다.")

def listen_microphone():
    recognizer = sr.Recognizer()
    with sr.Microphone() as source:
        recognizer.adjust_for_ambient_noise(source, duration=1)
        recognizer.pause_threshold = 2.0
        audio = recognizer.listen(source)
        question = recognizer.recognize_google(audio, language="ko-KR")
        return question

def speak(text):
    lang = detect(text)
    voice = "en-US-GuyNeural" if lang == 'en' else "ko-KR-InJoonNeural"
    # edge-tts로 음성 출력
```

대화 종료를 말하면 종료하도록 커스터마이즈 했고, 프롬프트를 직접 추가하여 성격을 지정할 수 있다. 음성도 한국어/영어에 적합한 목소리로 변경된다.

GitHub: [https://github.com/Lajancia/LLM-voice-control](https://github.com/Lajancia/LLM-voice-control)

### 웹 검색은 어떻게 붙일까?

LangChain 사용을 추천받았다. 특히 도구! 데이터베이스와 연결이 가능하다면 이전 대화를 기억하게 할 수도 있을 것 같다.

이번 토이프로젝트에서는 **llama3 + LangChain + Django** 조합으로 개인 AI 비서를 만들어보도록 하자.