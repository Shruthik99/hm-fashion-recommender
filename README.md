# 🛍️ Fashion Recommender: Balancing Relevance and Diversity

A recommender system built on **31.8 million real H&M purchases**. It compares four ways of deciding what to show a customer, and measures something most projects skip: how much accuracy you give up when you make recommendations more varied.


---

## 🎯 What problem is this solving?

Imagine a shop assistant who has to suggest 12 items to each customer walking in.

**Assistant A** suggests the same 12 bestsellers to everybody. Safe, but it ignores the person — someone who only buys dresses gets shown socks.

**Assistant B** looks at what you bought before. If you bought three black sweaters, you get twelve more black sweaters. Personal, but boring.

**Assistant C** does what B does, then deliberately swaps a few suggestions for something different — a different category, a different colour.

The catch: every time Assistant C adds variety, it risks dropping an item you *would* have bought. **This project measures that trade-off.**

---

## 🔍 How it works

Each model takes a customer's purchase history and returns **12 article recommendations**. Four models, built in increasing order of sophistication:

| # | Model | What it does |
|---|---|---|
| 1️⃣ | **Popularity** | Recommends the most-purchased articles to everyone. The baseline everything else must beat. |
| 2️⃣ | **ALS** | Learns a "taste vector" for each customer and article from purchase patterns alone. Finds customers similar to you and recommends what they bought. |
| 3️⃣ | **Hybrid** | Takes ALS's top 100 candidates and boosts those matching your usual product types and colours. |
| 4️⃣ | **Hybrid + MMR** | Re-picks the final 12 so they aren't all near-identical. Each pick balances *relevance* against *difference from what's already chosen*. |

### 📏 How performance is measured

Hide the last two weeks of purchases. Let each model recommend using only earlier data. Then check how many of its 12 suggestions the customer actually bought in those hidden weeks.

- **Precision@12** — of the 12 recommended, what share were bought?
- **Recall@12** — of what the customer bought, what share did we find?
- **NDCG@12** — same, but rewards putting correct items near the top
- **Intra-list diversity (ILD)** — how different the 12 items are from each other

---

## 📊 Results

Tested on **83,578 customers** over a held-out 14-day period.

| Model | Precision@12 | Recall@12 | NDCG@12 | Diversity (attr) | Diversity (cosine) |
|---|---|---|---|---|---|
| Popularity (whole window) | 0.0020 | 0.0062 | 0.0043 | 0.8178 | 0.4072 |
| Popularity (last day) | 0.0043 | 0.0140 | 0.0091 | 0.8007 | 0.7845 |
| ALS | 0.0044 | 0.0151 | 0.0101 | 0.8101 | 0.4593 |
| **Hybrid** 🏆 | **0.0048** | **0.0162** | **0.0109** | 0.7027 | 0.4013 |
| Hybrid + MMR (λ=0.8) | 0.0046 | 0.0159 | 0.0107 | 0.7879 | 0.4248 |

<details>
<summary><b>❓ Why are these numbers so small?</b></summary>

<br>

They look low, but that's the nature of the problem, not a broken model:

- There are **35,314 candidate articles** to choose 12 from
- The average customer buys only **3.7 articles** in a two-week window
- **29.7%** of test purchases are articles that never appeared in training — unreachable by any model here

In practice, the hybrid puts at least one correct item in about **5% of customers' lists**. Compare models *against each other*, not against an intuition of what a "good" percentage looks like.

</details>

---

## 💡 What was learned

### ⏱️ Recency mattered more than model sophistication

Weighting recent purchases more heavily (a 7-day half-life) raised ALS validation Precision@12 from **0.0028 → 0.0045**. No change to the model architecture came close — not more latent factors, not different regularization.

**Why:** fashion is seasonal. Trousers fell from 15.8% of purchases in March to 10.1% in June, while dresses rose from 9.5% to 15.2%. A model weighting six-month-old purchases equally is learning last season's taste.

### ⚖️ The baseline you pick decides your conclusion

> ALS beats popularity by **2.2×** — if popularity is counted over the whole six months.
>
> ALS is **statistically tied** with popularity — if popularity is counted over just the last day.

Both statements are true. Most papers report the first. This one reports both, because the second is the honest comparison.

### 🎨 Product attributes were the cheapest improvement

Adding product type, department and colour to ALS improved Precision@12 by **+0.00036 (±0.00010)** — a real effect, not noise — for only **11% more compute**. It also tripled how many distinct articles get recommended across all customers: 359 → 1,026.

### 🔀 Diversity is cheap, but only up to a point

MMR made lists **6% more varied** (on a measure it doesn't optimize) for a **2% precision cost**. 

But push the diversity dial too far — below λ=0.7 — and it stops working: the *independent* diversity measure starts falling while precision keeps dropping. You pay for variety that only exists in the number being maximized.

---

## 🔬 Why you can trust these numbers

Recommender evaluation is easy to get wrong. The steps taken to avoid that:

<details>
<summary><b>📅 Split by time, not at random</b></summary>

<br>

Test data is strictly *later* than anything the model saw. Random splits let a model learn from the future, which makes offline results unreliable ([Ji et al., 2020](https://arxiv.org/abs/2010.11060)).

</details>

<details>
<summary><b>🚫 Filtering can't peek at the future either</b></summary>

<br>

Customers need ≥5 purchases to be included — but that's counted on *training* purchases only. Counting all purchases would let a customer's future shopping decide whether they're in the dataset at all.

</details>

<details>
<summary><b>🔒 The test set was used exactly once</b></summary>

<br>

8 popularity windows, 12 ALS configs, 11 blending weights and 8 diversity settings were all compared on a **separate validation period**. Only then was the test set scored, with every setting frozen. No peeking, no retroactive tuning.

</details>

<details>
<summary><b>📏 The scoring code was verified three ways</b></summary>

<br>

1. Random recommendations score at chance level (0.00009 vs 0.00010 expected)
2. A by-hand calculation reproduces the vectorized result exactly
3. A perfect list scores NDCG = 1.0000

</details>

<details>
<summary><b>📐 Every comparison has an error bar</b></summary>

<br>

Differences are computed **per customer** and reported with ±2 standard errors, so you can see which gaps are real and which are noise. Several turned out to be noise — and are reported as such.

</details>

## 🏗️ The pipeline

```text
31.8M purchases (Sept 2018 – Sept 2020)
        │
        ├─ Keep last 6 months before the evaluation period
        ├─ Drop customers & articles with <5 purchases (repeat until stable)
        ▼
451,552 customers × 35,314 articles  (99.96% of the matrix is empty)
        │
        ├─ Train: Feb 26 – Aug 25, 2020
        ├─ Validation: Aug 26 – Sep 8    ← all tuning happens here
        └─ Test: Sep 9 – Sep 22          ← scored once
        │
        ▼
┌───────────────────────────────────────────────┐
│ Popularity   most-bought articles, last day   │
│ ALS          32 factors, 7-day half-life      │
│ Hybrid       0.7 × ALS + 0.3 × content        │
│ MMR          greedy re-rank, λ = 0.8          │
└───────────────────────────────────────────────┘
        ▼
   12 recommendations per customer
```


---

## 🚀 Run it yourself

**Prerequisite:** a free Kaggle account, with the competition rules accepted → [accept here](https://www.kaggle.com/competitions/h-and-m-personalized-fashion-recommendations)

```bash
# 1. Clone the repo
git clone https://github.com/YOUR-USERNAME/REPO-NAME.git
cd REPO-NAME

# 2. Install dependencies
pip install -r requirements.txt

# 3. Set your Kaggle API token
#    (Settings → API → Generate New Token on kaggle.com)
setx KAGGLE_API_TOKEN "your_token_here"     # Windows
export KAGGLE_API_TOKEN="your_token_here"   # macOS / Linux

# 4. Open the notebook
jupyter notebook
```

Then run all cells top to bottom.

> ⚠️ **First run downloads ~30 GB** of competition data. It's cached afterwards, so later runs skip straight to the analysis. Expect ~30 minutes for a full run.

---

## 📁 What's in here




The notebook runs cleanly from a fresh kernel — execution counts 1 through 30, no errors.

**Built with:** `pandas` · `numpy` · `scipy` · `implicit` · `matplotlib` · `kagglehub`

---

## ⚠️ What this doesn't prove

Stated plainly, because results mean less without their boundaries:

- 🗓️ **One 14-day test window, one random seed.** No evidence about robustness across seasons or promotions.
- 👥 **Only customers with ≥5 purchases** are evaluated — that's 65.1% of test buyers. The other 35% can't be personalized at all (though a popularity fallback serves them at ~91% of its usual accuracy).
- 🆕 **29.7% of test purchases** involve articles absent from training. No model here can reach them.
- 🔄 **Attribute diversity is partly circular** for MMR, since it's the exact quantity MMR maximizes. That's why an independent cosine measure is reported alongside it.

---

## 🔭 Where this could go next

- 🖼️ **Use the product images.** Two dresses with identical text attributes can look nothing alike. Visual embeddings could give a far better similarity measure — and could reach the cold-start articles that text attributes can't.
- 🆕 **Build a proper cold-start path** for the 35% of customers with too little history, e.g. popularity within customer segments.
- 📈 **Evaluate over rolling windows** with confidence intervals instead of a single fortnight.
- 🤖 **Replace the hand-tuned blend weight** with a learned ranker over all available signals.

---

## 📄 License

MIT
---


