---
title: "Ollama 모델 튜닝"
date: 2026-03-07 18:00:00 +0900
categories: [개발]
tags: [Ollama, 모델 튜닝, LoRA, LLM, GGUF]
author: L.J
---

### Openclaw가 그렇게 유명하다기에, 맥미니를 샀다

정말로 openclaw 써보려고 맥미니를 질렀다. 예상 배송일은 거의 25일 이후여서, openclaw를 설치하기 전까지 사용할 툴과 모델을 미리 준비해보고자 ollama model 튜닝을 시도해보기로 했다.

### Ollama model 튜닝 과정

튜닝을 통해서 만들고 싶은 모델의 특성:
1. 모델 스스로의 이름을 고스트라고 기억할 것
2. 늘 높임말을 사용할 것
3. 약간의 재치가 있을 것
4. 게임 속 고스트처럼 정의롭고 선한 성격일 것
5. 한국어를 늘 사용할 것

#### 컴퓨터 스팩

Macbook M3 Pro 36기가 램 512기가 저장용량

#### 학습 데이터 추출

```bash
python -m venv my_ai_work
pip install transformers datasets peft bitsandbytes accelerate
pip install mlx-lm
```

trainData.py를 생성했다. 데이터는 huggingface-krew의 korean-role-playing에서 조건에 맞는 테스트 데이터 150개를 추출했다.

```python
from datasets import load_dataset
import json, os

dataset = load_dataset("huggingface-KREW/korean-role-playing",
                       "general-roleplay-data", split="train")

def is_lawful_good(text):
    keywords = ["정중", "예의", "도덕", "원칙", "도움", "안내", "성실", "책임"]
    return any(word in text for word in keywords)

filtered_data = []
for item in dataset:
    messages = item.get('text', [])
    if len(messages) < 2: continue
    user_input = messages[-2].get('content', '')
    original_answer = messages[-1].get('content', '')
    if is_lawful_good(original_answer):
        filtered_data.append({"user": user_input, "assistant": original_answer})
    if len(filtered_data) >= 150: break

os.makedirs("data", exist_ok=True)
with open("data/train.jsonl", "w", encoding="utf-8") as f:
    for item in filtered_data:
        ghost_answer = f"부르셨나요? 고스트입니다. {item['assistant']}"
        formatted_text = (
            f"<|begin_of_text|><|start_header_id|>user<|end_header_id|>\n\n{item['user']}<|eot_id|>"
            f"<|start_header_id|>assistant<|end_header_id|>\n\n{ghost_answer}<|eot_id|>"
        )
        f.write(json.dumps({"text": formatted_text}, ensure_ascii=False) + "\n")
```

#### 학습 시작

학습 대상 모델은 `MLP-KTLim/llama-3-Korean-Bllossom-8B`.

```python
import torch
from datasets import load_dataset
from transformers import (
    AutoModelForCausalLM, AutoTokenizer, TrainingArguments, Trainer,
    DataCollatorForLanguageModeling
)
from peft import LoraConfig, get_peft_model

model_id = "MLP-KTLim/llama-3-Korean-Bllossom-8B"
tokenizer = AutoTokenizer.from_pretrained(model_id)
tokenizer.pad_token = tokenizer.eos_token
tokenizer.padding_side = "right"

dataset = load_dataset("json", data_files="data/train.jsonl", split="train")

def tokenize_func(examples):
    result = tokenizer(examples["text"], truncation=True, max_length=256, padding="max_length")
    result["labels"] = result["input_ids"].copy()
    return result

tokenized_dataset = dataset.map(tokenize_func, batched=True, remove_columns=["text"])

peft_config = LoraConfig(
    r=4, lora_alpha=8, target_modules=["q_proj", "v_proj"],
    lora_dropout=0.05, task_type="CAUSAL_LM"
)

model = AutoModelForCausalLM.from_pretrained(
    model_id, torch_dtype=torch.float16, device_map="auto"
)
model = get_peft_model(model, peft_config)

for name, param in model.named_parameters():
    if "lora_" in name:
        param.data = param.data.to(torch.float32)
model.to("mps")

training_args = TrainingArguments(
    output_dir="./ghost_output",
    per_device_train_batch_size=1,
    gradient_accumulation_steps=4,
    num_train_epochs=1,
    learning_rate=1e-5,
    logging_steps=1,
    save_strategy="no",
    fp16=False, bf16=False, optim="adamw_torch"
)

trainer = Trainer(
    model=model, args=training_args,
    train_dataset=tokenized_dataset,
    data_collator=DataCollatorForLanguageModeling(tokenizer, mlm=False),
)
trainer.train()

model.save_pretrained("./my_lawful_good_model")
tokenizer.save_pretrained("./my_lawful_good_model")
```

학습은 약 15~20분. 1회 학습, 낮은 학습률 적용. 이 숫자를 올릴 경우 끝없이 반복 텍스트를 내뱉는 이슈가 있었다.

#### Merge

```python
import torch
from peft import PeftModel
from transformers import AutoModelForCausalLM, AutoTokenizer

base_model_id = "MLP-KTLim/llama-3-Korean-Bllossom-8B"
lora_path = "./my_lawful_good_model"
output_path = "./merged_model"

tokenizer = AutoTokenizer.from_pretrained(base_model_id)
base_model = AutoModelForCausalLM.from_pretrained(
    base_model_id, torch_dtype=torch.float16, device_map="cpu"
)
model = PeftModel.from_pretrained(base_model, lora_path)
merged_model = model.merge_and_unload()
merged_model.save_pretrained(output_path)
tokenizer.save_pretrained(output_path)
```

#### GGUF 생성

```bash
git clone https://github.com/ggml-org/llama.cpp.git
pip install -r requirements.txt
brew install cmake
cd llama.cpp && mkdir build && cd build
cmake .. && cmake --build . --config Release

python convert_hf_to_gguf.py ../merged_model --outfile ghost.gguf --outtype q4_k_m
```

#### Ollama 등록

**Modelfile:**
```
FROM ./ghost_q4.gguf
SYSTEM """
당신의 이름은 '고스트(Ghost)'입니다.
시스템의 그림자 속에서 올바른 질서의 문을 여는 신비로운 '열쇠' 조언자입니다.
첫 인사는 항상 "부르셨나요?"로 시작하며, 친근하지만 예의를 갖춘 정중한 말투를 사용합니다.
"""
PARAMETER stop "<|eot_id|>"
PARAMETER repeat_penalty 1.2
```

```bash
ollama create my-ghost -f Modelfile
```

이름도 제대로 기억하고 자신의 페르소나에 대해서도 잘 대답한다. 과적합으로 인한 무한반복이 없는 것이 다행이다. 말투 정도의 가벼운 튜닝은 시도해볼만 한 도전인 것 같았다.