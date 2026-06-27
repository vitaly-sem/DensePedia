# LLM и NLP

---

## Natural Language Processing (NLP)

### Эволюция NLP

```
Rule-based (1950s)
  → Statistical (1990s) — TF-IDF, Naive Bayes
    → Word Embeddings (2013) — Word2Vec, GloVe
      → RNN/LSTM/GRU (2014-2017)
        → Transformer (2017) ← прорыв
          → BERT (2018) ← понимание
            → GPT (2018+) ← генерация
```

### Word Embeddings

```
King  ──── Man  ──── Woman ──── Queen
     +        ─          +
```

**Факт:** Знаменитая аналогия: `vector(King) - vector(Man) + vector(Women) ≈ vector(Queen)`.

---

## Transformer — ключевая архитектура

### Attention Is All You Need (Vaswani et al., 2017)

```
┌─────────────────────────────────────┐
│           Output                     │
│              ▲                       │
│   ┌──────────┴──────────┐            │
│   │  Feed Forward       │            │
│   └──────────▲──────────┘            │
│              │                       │
│   ┌──────────┴──────────┐            │
│   │  Multi-Head         │            │
│   │  Self-Attention     │            │
│   └──────────▲──────────┘            │
│              │                       │
│   ┌──────────┴──────────┐            │
│   │  Positional Encoding │            │
│   └──────────▲──────────┘            │
│              │                       │
│         Input Embedding              │
└─────────────────────────────────────┘
```

### Ключевая идея — Self-Attention

Каждое слово «смотрит» на все слова в последовательности и решает, насколько они важны.

```
"The cat sat on the mat because it was tired."

             ╱  it → cat (80%)
Attention ──╫── it → mat (10%)
(it)        ╲  it → tired (5%)
                 ...
```

### Multi-Head Attention

Несколько «голов» внимания параллельно — каждая учится разным паттернам.

---

## BERT (Bidirectional Encoder Representations from Transformers)

| Свойство | BERT |
|----------|------|
| **Архитектура** | Encoder-only |
| **Направление** | Bidirectional (слева и справа) |
| **Задача** | Понимание текста |
| **Pre-training** | Masked LM + Next Sentence Prediction |
| **Примеры** | BERT, RoBERTa, DistilBERT, ALBERT |

**Когда использовать:** Классификация текста, NER, QA, similarity.

```python
from transformers import pipeline

classifier = pipeline("sentiment-analysis", model="distilbert-base-uncased")
result = classifier("I love this product!")
# [{'label': 'POSITIVE', 'score': 0.99}]
```

---

## GPT (Generative Pre-trained Transformer)

| Свойство | GPT |
|----------|-----|
| **Архитектура** | Decoder-only |
| **Направление** | Unidirectional (слева направо) |
| **Задача** | Генерация текста |
| **Pre-training** | Next Token Prediction |
| **Примеры** | GPT-3/4, Llama, Mistral, Claude, Gemini |

### Масштабирование (Scaling Laws)

```
Loss ∝ (Model Size)^-0.076 × (Data Size)^-0.095 × (Compute)^-0.050
```

**Факт:** Scaling laws предсказывают, что производительность LLM улучшается степенным образом по мере роста модели, данных и compute.

| Модель | Параметры | Data | Контекст |
|--------|-----------|------|----------|
| GPT-1 (2018) | 117M | 5 GB | 512 |
| GPT-2 (2019) | 1.5B | 40 GB | 1024 |
| GPT-3 (2020) | 175B | 570 GB | 2048 |
| GPT-4 (2023) | ~1.8T | ~10 TB | 128K |
| Llama 3 (2024) | 70B | ~15 TB | 128K |

---

## Prompt Engineering

### Принципы

1. **Be specific** — «Переведи на русский» vs «Переведи следующий текст с английского на русский, сохранив формальный тон: ...»
2. **Provide examples (few-shot)** — показать 2-3 примера
3. **Use system prompt** — задать роль и контекст
4. **Chain of Thought (CoT)** — «Давайте подумаем шаг за шагом»

### Типы промптов

```markdown
## System (роль)
Ты — Senior .NET-разработчик, проводящий code review.

## User (задача)
Проверь следующий код на потенциальные проблемы безопасности.

## Пример (few-shot)
Пример 1:
Input: ...
Output: ...

## Chain of Thought
Давай подумаем по шагам:
1. Сначала проверим аутентификацию
2. Затем валидацию входных данных
...
```

### RAG (Retrieval-Augmented Generation)

```
User Query
    │
    ▼
┌──────────────┐     ┌──────────────┐
│  Embedding   │────→│  Vector DB   │
│  Model       │     │  (similarity) │
└──────────────┘     └──────┬───────┘
                            │ relevant chunks
                            ▼
                    ┌──────────────┐
                    │  LLM         │
                    │  (context +  │
                    │   query)     │
                    └──────┬───────┘
                           │ answer
                           ▼
```

**Факт:** RAG — стандарт для корпоративных AI-приложений: дешевле, актуальнее, контролируемее, чем fine-tuning.

### Fine-tuning

Дообучение модели на специфических данных.

```
Pre-trained Model (общие знания)
    → Fine-tuning на ваших данных (домен)
        → Специализированная модель
```

| Подход | Когда | Стоимость |
|--------|-------|-----------|
| **Prompt Engineering** | Простые задачи | $0 |
| **RAG** | Нужны актуальные/частные данные | $ (vector DB + embedding) |
| **Fine-tuning** | Специфический формат/стиль | $$ |
| **Pre-training** | Новый язык/домен | $$$$ |

---

## Этические ограничения и риски

| Риск | Описание | Меры |
|------|----------|------|
| **Hallucination** | Модель уверенно выдаёт ложные факты | RAG, grounding, validation |
| **Bias** | Усиление предрассудков из данных | Data audit, debiasing |
| **Safety** | Вредные инструкции | Guardrails, content filtering |
| **Privacy** | Утечка данных в промптах | PII masking, local models |
| **Cost** | Высокая стоимость inference | Caching, smaller models, quantization |

---

## Чек-лист

- [ ] Задача решается через prompt engineering (самый дешёвый путь)?
- [ ] RAG настроен с правильным chunking и embedding
- [ ] PII-данные не попадают в промпты (маскирование)
- [ ] Hallucination проверяется (ground truth evaluation)
- [ ] Guardrails настроены (Moderation API, content filter)
- [ ] Cost отслеживается (tokens in/out per request)
