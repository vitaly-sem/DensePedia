# Основы машинного обучения

---

## Что такое ML

**Machine Learning** — подмножество AI, где система учится на данных без явного программирования правил.

```
Traditional Programming:
    Rules + Data → Answers

Machine Learning:
    Data + Answers → Rules (model)
```

---

## Типы обучения

### Supervised Learning (обучение с учителем)

Модель учится на размеченных данных (X → y).

| Задача | Пример | Алгоритмы |
|--------|--------|-----------|
| **Classification** | Спам/не спам | Logistic Regression, SVM, Random Forest, XGBoost, Neural Networks |
| **Regression** | Цена дома | Linear Regression, Decision Trees, Neural Networks |

```python
# Классификация на Python (scikit-learn)
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split

X_train, X_test, y_train, y_test = train_test_split(X, y, test_size=0.2)
model = RandomForestClassifier(n_estimators=100)
model.fit(X_train, y_train)
accuracy = model.score(X_test, y_test)
```

### Unsupervised Learning (обучение без учителя)

Модель ищет структуру в данных без меток.

| Задача | Пример | Алгоритмы |
|--------|--------|-----------|
| **Clustering** | Сегментация клиентов | K-Means, DBSCAN, Hierarchical |
| **Dimensionality Reduction** | Сжатие признаков | PCA, t-SNE, UMAP |
| **Anomaly Detection** | Выявление мошенничества | Isolation Forest, Autoencoder |

### Reinforcement Learning (обучение с подкреплением)

Агент учится через взаимодействие со средой, получая награду за действия.

```
State → Agent → Action → Environment → Reward + Next State
```

**Примеры:** AlphaGo, Dota 2 bots, робототехника, рекомендательные системы.

---

## Нейронные сети

### Перцептрон

```
    x1 ─── w1 ─┐
                │
    x2 ─── w2 ─── Σ ── activation ── output
                │
    x3 ─── w3 ─┘
```

### MLP (Multilayer Perceptron)

```
Input Layer → Hidden Layer(s) → Output Layer
    [4]          [128, 64]         [2]
```

### Функции активации

| Функция | Формула | Когда |
|---------|---------|-------|
| **ReLU** | `max(0, x)` | Hidden layers (стандарт) |
| **Sigmoid** | `1 / (1 + e⁻ˣ)` | Binary classification output |
| **Softmax** | `eˣⁱ / Σeˣʲ` | Multi-class classification |
| **Tanh** | `(eˣ - e⁻ˣ) / (eˣ + e⁻ˣ)` | RNN/LSTM |

### Как обучается нейросеть

```
Forward pass  →  Compute loss  →  Backpropagation  →  Update weights
   (predict)        (error)         (gradients)         (optimizer)
```

**Факт:** Backpropagation — просто chain rule из математики (производная композиции функций).

---

## Оценка моделей

### Метрики классификации

| Метрика | Формула | Когда важна |
|---------|---------|-------------|
| **Accuracy** | `(TP+TN)/(TP+TN+FP+FN)` | Сбалансированные классы |
| **Precision** | `TP/(TP+FP)` | Ложные срабатывания дороги (spam) |
| **Recall** | `TP/(TP+FN)` | Пропуск цели дорог (рак) |
| **F1** | `2·P·R/(P+R)` | Дисбаланс классов |

```
               Predicted
               Positive Negative
Actual Positive   TP       FN
       Negative   FP       TN
```

### Метрики регрессии

| Метрика | Описание |
|---------|----------|
| **MSE** | Mean Squared Error — штрафует большие ошибки |
| **MAE** | Mean Absolute Error — все ошибки равны |
| **R²** | Коэффициент детерминации — доля объяснённой дисперсии |

### Overfitting vs Underfitting

```
Underfitting       Good Fit          Overfitting
│╲    ╱│          │╲  ╱│          │╲        ╱│
│ ╲  ╱ │          │ ╲╱ │          │ ╲  ╲╱╱  │
│  ╲╱  │          │  ╲╱ │          │  ╲╱  ╲╱ │
└──────┘          └──────┘          └──────────┘
High bias         Balanced          High variance
```

**Борьба с overfitting:** Regularization (L1/L2), Dropout, Early Stopping, More data, Data augmentation.

---

## Feature Engineering

### Типы признаков

| Тип | Пример | Кодирование |
|-----|--------|-------------|
| **Numerical** | Возраст, цена | Scaler (Standard, MinMax) |
| **Categorical** | Страна, пол | One-hot, Label, Target encoding |
| **Text** | Описание | TF-IDF, Embeddings |
| **Temporal** | Дата | Разбить на час/день/месяц/день_недели |

### Feature Importance

```python
# Random Forest — importance
importances = model.feature_importances_
# SHAP — model-agnostic
import shap
explainer = shap.TreeExplainer(model)
shap_values = explainer.shap_values(X)
```

---

## Data Pipeline

```
Raw Data → Clean → Transform → Feature Engineering → Train/Test Split → Model
               ↑                                         ↓
         Missing values                            Cross-validation
         Outliers                                  Hyperparameter tuning
         Duplicates
```

### Train / Validation / Test

```
┌─────────────────────────────────────────────┐
│              All Data (100%)                 │
├─────────────────┬───────────────┬────────────┤
│   Train (60%)   │ Val (20%)    │ Test (20%) │
│   ← учим модель  │ ← выбираем    │ ← финальная │
│                  │   гиперпарам. │   оценка    │
└─────────────────┴───────────────┴────────────┘
```

---

## Чек-лист

- [ ] Данные размечены и проверены на качество
- [ ] Baseline (простая модель) определён до сложной
- [ ] Train/Val/Test разделены (не подсматривать в test!)
- [ ] Гиперпараметры подобраны (GridSearch, Optuna)
- [ ] Модель проверена на overfitting (train vs val loss)
- [ ] Feature importance проанализирована
- [ ] Метрики выбраны под бизнес-задачу, не generic accuracy
