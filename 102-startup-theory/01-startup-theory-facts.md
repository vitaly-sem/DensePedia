# Start-up Theory & Facts

---

## Product-Market Fit (PMF)

**Факты:**
- **Product-Market Fit** — продукт решает настоящую проблему рынка
- Marc Andreessen: "The only thing that matters"
- **Superlinear** (PMF reached): пользователи растут органически, churn < 2%/мес

**Признаки PMF:**

| Метрика | Pre-PMF | Post-PMF |
|---|---|---|
| Organic growth | < 5% | > 20% (monthly) |
| NPS | < 20 | > 40 |
| Churn | > 5% | < 2% |
| User feedback | "Nice to have" | "I can't live without it" |
| Referrals | Rare | Viral loop |

**Факт:** Sean Ellis Test: "How would you feel if you could no longer use this product?" — **40%+ "Very disappointed"** = PMF.

---

## Lean Startup Methodology

**Факт:** Build → Measure → Learn (цикл):

```
Build → MVP (minimum viable product)
  ↓
Measure → KPIs, A/B tests, user behavior
  ↓
Learn → Validate or invalidate hypothesis
  ↓
Pivot or Persevere
```

```markdown
### Hypotesis Template
We believe [target user] needs [solution] because [problem].
We'll know we're right when we see [metric] reach [threshold].
```

**Факт:** **MVP — это не минимальный продукт, а минимальный *обучающий* продукт.** Что даст максимальное количество learning за минимальное время.

---

## Key Metrics

### SaaS Metrics (B2B/B2C)

| Metric | Formula | Good | Great |
|---|---|---|---|
| **MRR** (Monthly Recurring Revenue) | Paying users × ARPU | — | Growing |
| **ARR** | MRR × 12 | — | > $1M (seed) |
| **LTV** (Lifetime Value) | ARPU × Avg lifespan | > 3× CAC | > 5× CAC |
| **CAC** (Customer Acquisition Cost) | Sales & marketing / New customers | < $100 (B2C) | < $1000 (B2B) |
| **LTV/CAC** | LTV ÷ CAC | > 3 | > 5 |
| **Churn** | Lost customers / Total | < 5% monthly | < 2% monthly |
| **NRR** (Net Revenue Retention) | (Revenue from existing + upsells) / Prior revenue | > 100% | > 120% |
| **CAC Payback** | CAC / (ARPU × Margin) | < 12 months | < 6 months |

**Факт:** **NRR > 100%** → existing customers expand faster than churn. Лучший признак стабильного роста.

---

## Fundraising Stages

| Stage | ARR (B2B) | Raise | Dilution | Focus |
|---|---|---|---|---|
| **Pre-Seed** | $0 | $50-500K | 5-15% | Idea, team |
| **Seed** | $0-100K | $500K-2M | 15-25% | MVP, early traction |
| **Series A** | $1-3M | $2-15M | 15-25% | PMF, scalable growth |
| **Series B** | $5-20M | $10-50M | 10-20% | Scaling, hirings |
| **Series C+** | $20M+ | $30M+ | 5-15% | Expansion, M&A |

**Факт:** **90%+ стартапов умирают до Series A** (Shikhar Ghosh, Harvard).

**Факт:** **Series A — самая сложная.** 2019 → 2024: время от Seed до Series A выросло с 12 до 25+ месяцев.

---

## Unit Economics

```markdown
### Единица экономики: платящий пользователь / заказ

Revenue:
  ARPU = $50/мес (средний платёж)
  
Costs:
  COGS (инфраструктура)           = $10
  Customer Support                 = $5  
  Gross Margin                     = $35 (70%)

Sales & Marketing per customer:
  CAC = $500 (реклама + sales)

LTV = ARPU × Gross Margin × Avg Months
    = $50 × 0.70 × 36 (3 years)
    = $1,260

LTV/CAC = $1,260 / $500 = 2.5 ✗ (нужно > 3)
```

**Факт:** Если LTV/CAC < 3 — бизнес сжигает деньги. Нужно либо поднять ARPU, либо снизить CAC.

---

## Pivot Types

| Pivot Type | Description | Example |
|---|---|---|
| **Zoom-in** | Feature → Product | Instagram (Burbn → photo sharing) |
| **Zoom-out** | Product → Platform | Slack (game → platform) |
| **Customer Segment** | Change target audience | Twitter (podcasts → microblogging) |
| **Customer Need** | Solve different problem | YouTube (dating → video sharing) |
| **Business Model** | Change revenue model | Dropbox (pay once → subscription) |
| **Technology** | Different tech | Netflix (DVD → streaming) |
| **Channel** | Change distribution | Dollar Shave Club (stores → subscription) |

---

## Metrics for Technical Founders

```markdown
### Technical Metrics Investors Ask About

1. **Infrastructure cost per user** — как растут затраты с ростом?
   - ❌ Linear → проблемы при scale
   - ✅ Sub-linear → экономия на масштабе

2. **Time to deploy** — DORA metrics
   - Deploy frequency
   - Lead time to deploy

3. **Uptime / Reliability**
   - SLA: 99.9% = 8.7h/year, 99.99% = 52min/year

4. **Data privacy / Compliance**
   - SOC 2, GDPR, HIPAA (зависит от рынка)

5. **Scalability architecture**
   - Monolith vs microservices
   - Database bottlenecks
   - Caching strategy
```

---

## Famous Startup Facts

- **Airbnb**: создатели продали Obama O's cereal, чтобы выжить (2008)
- **Slack**: начался как внутренний инструмент для игры Glitch
- **Twitter**: изначально — подкаст-платформа Odeo
- **YouTube**: изначально — dating сайт "Tune In Hook Up"
- **Instagram**: начался как Burbn (Foursquare + Instagram)
- **Netflix**: DVD по почте (2007 → streaming)
- **PayPal**: изначально — криптография (Palm Pilot payments)
- **TikTok**: купил Musical.ly за $1B (2017) → объединил

---

## Чек-лист

- [ ] PMF: Sean Ellis Test (40% "Very disappointed")
- [ ] Lean Startup: Build → Measure → Learn
- [ ] Key metrics: MRR, ARR, LTV, CAC, Churn, NRR
- [ ] Fundraising: Seed → Series A → B → C
- [ ] Unit economics: LTV/CAC > 3
- [ ] Pivot types: zoom-in, segment, need, model
- [ ] Technical metrics: infra per user, DORA, SLA
