# Multimodal Product Search Using Image–Text Prompts and Review-Based Ranking

> **Research project:** Multimodal Product Search using Image–Text Prompts and Review-Based Ranking  
> **Authors:** Ojaswin Aggarwal, Harsh Rathore, Milandeep Singh, Abhishek Singhal  
> **Institution:** Department of Computer Science and Engineering, Amity School of Engineering and Technology (ASET), Amity University Uttar Pradesh, Noida, India

## Overview

This project implements a **multimodal product search and ranking system** that combines:

- 🖼️ **Product images**
- 💬 **Natural-language search queries**
- 🧩 **Structured product constraints** such as category, color, size, material, and price
- ⭐ **Review-quality signals**
- 🧠 **Query-intent detection**
- 📊 **Multiple fusion and ranking strategies**

The core idea is to move beyond conventional keyword-only product search. A user can provide an image, text, or both, while specifying constraints such as:

> `black cotton shirt size M under 1500`

The system extracts structured information from the query, optionally extracts visual attributes from the uploaded image using **CLIP**, retrieves candidate products from a SQLite catalog, and ranks the candidates using visual, constraint, and review-quality signals.

The research paper evaluates the approach on **85,241 products and 1,500 queries**, reporting an **NDCG@10 of 0.1691** and **MRR of 0.1548** for the intent-aware dynamic fusion configuration.

The paper describes the motivation, architecture, mathematical formulation, experiments, limitations, and future scope of the system. fileciteturn0file0L14-L23

---

## Key Features

### 1. Multimodal Search

The system accepts:

- Natural-language text
- An optional product image
- A combination of both

The backend exposes a search API that accepts a text query, selected fusion method, and optional image upload.

### 2. CLIP-Based Visual Attribute Extraction

Instead of using CLIP purely as a vector similarity retriever, this project uses CLIP for **zero-shot symbolic attribute extraction**.

The image is classified against predefined product and color vocabularies. The extracted result is converted into structured attributes such as:

```text
category = jacket
color    = black
```

The research paper describes a 120-label product vocabulary and 14 standard colors, with CPU-compatible inference as a design goal. fileciteturn0file0L89-L105

### 3. Rule-Based Constraint Parser

Natural-language queries are converted into structured constraints.

Example:

```text
black cotton shirt size M under 1500
```

can produce a representation similar to:

```json
{
  "category": "shirt",
  "color": "black",
  "size": "M",
  "material": "cotton",
  "price_max": 1500,
  "keywords": []
}
```

The parser handles:

- Category
- Color
- Size
- Material
- Minimum price
- Maximum price
- Price ranges
- Remaining keywords

The research design deliberately uses deterministic rule-based parsing for these structured constraints. fileciteturn0file0L99-L105

### 4. Symbolic Multimodal Fusion

Image-derived attributes and text-derived constraints are merged into one constraint representation.

Text constraints take precedence when both modalities specify the same attribute, because explicit text is treated as the user's direct expression of intent. fileciteturn0file0L106-L111

### 5. Three Search/Reranking Strategies

The repository supports multiple fusion strategies:

| Method | Description |
|---|---|
| **Symbolic Early Fusion** | Applies structured constraints directly during candidate retrieval |
| **Static Late Fusion** | Retrieves candidates and combines visual, constraint, and review scores using fixed weights |
| **Intent-Aware Dynamic Fusion** | Detects query intent and dynamically adjusts the fusion weights |

The research paper compares these approaches experimentally. fileciteturn0file0L112-L145

### 6. Intent-Aware Ranking

Queries are classified into three intent types:

```text
visual
attribute
hybrid
```

The intent classifier examines extracted visual attributes and structured constraints.

The dynamic pipeline then changes the relative contribution of:

- Visual relevance: `α`
- Constraint/text relevance: `β`
- Review quality: `γ`

This allows the ranking strategy to adapt to the query instead of using identical weights for every search.

The research paper defines visual, attribute, and hybrid weighting strategies for this purpose. fileciteturn0file0L136-L145

### 7. Review-Aware Ranking

Product ranking incorporates a precomputed review-quality signal.

The research methodology uses:

- Sentiment classification
- TF-IDF
- Logistic Regression
- Review filtering
- Wilson-score-based quality estimation

The sentiment model artifacts included in the repository are:

```text
ml/models/
├── sentiment_logistic_regression.joblib
└── sentiment_tfidf_vectorizer.joblib
```

The paper describes review filtering followed by Wilson-score calculation to reduce the influence of products with unreliable or insufficient review evidence. fileciteturn0file0L181-L191

---

# System Architecture

```text
                     ┌─────────────────────────┐
                     │        User Query       │
                     │                         │
                     │  Text + Optional Image  │
                     └────────────┬────────────┘
                                  │
                    ┌─────────────┴─────────────┐
                    │                           │
                    ▼                           ▼
             ┌──────────────┐          ┌────────────────┐
             │ Text Parser  │          │ CLIP Inference │
             │              │          │                │
             │ category     │          │ category       │
             │ color        │          │ color          │
             │ size         │          │ confidence     │
             │ material     │          └───────┬────────┘
             │ price        │                  │
             │ keywords     │                  │
             └──────┬───────┘                  │
                    │                          │
                    └──────────┬───────────────┘
                               ▼
                    ┌──────────────────────┐
                    │ Constraint Fusion    │
                    │                      │
                    │ Unified constraints  │
                    └──────────┬───────────┘
                               │
                ┌──────────────┼───────────────┐
                │              │               │
                ▼              ▼               ▼
        ┌────────────┐ ┌──────────────┐ ┌─────────────────┐
        │ Early      │ │ Static Late  │ │ Intent-Aware    │
        │ Fusion     │ │ Fusion       │ │ Dynamic Fusion  │
        └─────┬──────┘ └──────┬───────┘ └────────┬────────┘
              │               │                  │
              └───────────────┼──────────────────┘
                              ▼
                   ┌─────────────────────┐
                   │ SQLite Product DB   │
                   │                     │
                   │ Products            │
                   │ Review Rankings     │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ Candidate Ranking   │
                   │                     │
                   │ Visual Score        │
                   │ Constraint Score    │
                   │ Review Score        │
                   └──────────┬──────────┘
                              │
                              ▼
                   ┌─────────────────────┐
                   │ Ranked Products     │
                   └─────────────────────┘
```

The paper describes the same overall architecture as a shared preprocessing stage followed by configurable fusion pipelines. fileciteturn0file0L82-L87

---

# Ranking Formulation

The ranking system combines three signals:

```text
Final Score =
    α × Visual Score
  + β × Constraint Score
  + γ × Review Score
```

where:

- `α` controls visual relevance
- `β` controls structured/textual relevance
- `γ` controls review quality

The research paper describes static fusion, where these weights remain fixed, and dynamic fusion, where the weights change according to query intent. fileciteturn0file0L147-L156

## Intent-Aware Weights

The research configuration uses different emphasis for:

### Visual intent

```text
α = 0.80
β = 0.05
γ = 0.15
```

### Attribute intent

```text
α = 0.05
β = 0.75
γ = 0.20
```

### Hybrid intent

```text
α = 0.50
β = 0.35
γ = 0.15
```

These values are part of the paper's experimental formulation. fileciteturn0file0L139-L145

The repository's intent-aware implementation additionally computes and normalizes its dynamic weights at runtime.

---

# Evaluation

## Dataset

The research experiment was constructed from six categories of the **Amazon Reviews 2023** dataset:

- Fashion
- Appliances
- Mobile Accessories
- Clothing
- Electronics
- Sports

After cleaning and filtering, the research database contained:

**85,241 products**

The paper reports removing products without usable images/titles, products with insufficient useful reviews, and products that could not be reliably assigned to the required categories. fileciteturn0file0L192-L199

## Query Set

The evaluation used **1,500 queries**:

| Query type | Number |
|---|---:|
| Visual | 500 |
| Attribute | 500 |
| Hybrid | 500 |
| **Total** | **1,500** |

Examples described in the paper include:

```text
Visual:
blue jacket

Attribute:
cotton medium jacket

Hybrid:
blue cotton jacket
```

The evaluation used NDCG, Precision, and MRR metrics. fileciteturn0file0L201-L207

---

# Research Results

The following numbers are **reported results from the accompanying research paper**, not a claim that they will necessarily be reproduced on every local installation.

| Metric | Symbolic Early | Static Late | Intent-Aware |
|---|---:|---:|---:|
| NDCG@5 | 0.1392 | 0.1598 | 0.1606 |
| NDCG@10 | 0.1490 | 0.1682 | **0.1691** |
| NDCG@20 | 0.1541 | 0.1731 | **0.1740** |
| Precision@5 | 0.0633 | 0.0680 | 0.0680 |
| Precision@10 | 0.0569 | 0.0589 | 0.0589 |
| MRR | 0.1346 | 0.1536 | **0.1548** |

The paper reports an NDCG@10 of **0.1691** and MRR of **0.1548** for intent-aware dynamic fusion. fileciteturn0file0L211-L230

The paper also reports that the hybrid-query NDCG@10 increased from **0.0536** with symbolic early fusion to **0.0801** with intent-aware fusion. fileciteturn0file0L251-L270

## Statistical Testing

A bootstrap significance test was used in the paper to examine the reported improvements. The paper reports statistically significant improvements for the late-fusion approaches over symbolic early fusion at `p < 0.05`, while the difference between static late fusion and intent-aware fusion was reported as minor and not significant. fileciteturn0file0L242-L250

---

# Project Structure

```text
Product-Recommender/
│
├── backendapi/
│   ├── main.py
│   └── requirements.txt
│
├── frontend/
│   ├── index.html
│   ├── package.json
│   ├── vite.config.ts
│   └── src/
│       ├── App.tsx
│       ├── components/
│       │   ├── DiagnosticsPanel.tsx
│       │   ├── ImageUploader.tsx
│       │   ├── MethodSelector.tsx
│       │   ├── ProductCard.tsx
│       │   ├── ResultsGrid.tsx
│       │   ├── SearchButton.tsx
│       │   └── SearchInput.tsx
│       ├── services/
│       │   └── api.ts
│       └── types/
│           └── search.ts
│
├── ml/
│   ├── data/
│   │   └── reviews-output/
│   │       ├── product_ranking.sqlite
│   │       └── eval_results.json
│   │
│   ├── models/
│   │   ├── sentiment_logistic_regression.joblib
│   │   └── sentiment_tfidf_vectorizer.joblib
│   │
│   ├── src/
│   │   ├── pipeline/
│   │   │   ├── constraint_parser.py
│   │   │   ├── early_fusion_pipeline.py
│   │   │   ├── late_fusion.py
│   │   │   ├── query_builder.py
│   │   │   ├── run_early_fusion.py
│   │   │   └── run_late_fusion.py
│   │   │
│   │   ├── sentiment/
│   │   │   ├── sentiment_artifacts.py
│   │   │   └── sentiment_inference.py
│   │   │
│   │   └── vision/
│   │       ├── clip_artifacts.py
│   │       ├── clip_inference.py
│   │       ├── clip_vocab.py
│   │       ├── early_fusion_clip_inference.py
│   │       └── vocab/
│   │
│   └── training/
│       ├── comprehensive_evaluation.py
│       ├── evaluate_final.py
│       ├── evaluate_retrieval.py
│       ├── intent_aware_fusion.py
│       ├── intent_classifier.py
│       ├── product_ranker.ipynb
│       └── sentiment_tfidf_logreg_training.ipynb
│
├── tmp_uploads/
├── pyproject.toml
├── .gitignore
└── README.md
```

---

# Technology Stack

## Machine Learning / NLP

- Python
- PyTorch
- Hugging Face Transformers
- CLIP
- scikit-learn
- TF-IDF
- Logistic Regression

## Backend

- FastAPI
- Python
- SQLite
- Uvicorn

## Frontend

- React
- TypeScript
- Vite
- Tailwind CSS
- Framer Motion

## Data / Storage

- Amazon Reviews 2023
- SQLite
- Precomputed review-ranking data
- Serialized ML artifacts using Joblib

---

# API

The project includes a FastAPI backend.

## Start the API

From the project root:

```bash
uvicorn backendapi.main:app --reload
```

The API provides:

### Health Check

```http
GET /health
```

Example response:

```json
{
  "status": "ok"
}
```

### Root

```http
GET /
```

### Product Search

```http
POST /api/search
```

Form fields:

```text
text   = natural-language query
method = early_fusion | late_fusion
image  = optional image upload
```

Example using `curl`:

```bash
curl -X POST http://127.0.0.1:8000/api/search \
  -F "text=black cotton shirt size M under 1500" \
  -F "method=late_fusion"
```

With an image:

```bash
curl -X POST http://127.0.0.1:8000/api/search \
  -F "text=black cotton shirt under 1500" \
  -F "method=late_fusion" \
  -F "image=@path/to/product.jpg"
```

The API returns ranked products together with fields such as:

```json
{
  "results": [
    {
      "id": "...",
      "title": "...",
      "price": 1299,
      "rating": 4.3,
      "image_url": "...",
      "category": "...",
      "final_score": 0.82,
      "clip_score": 0.76,
      "text_score": 0.91
    }
  ],
  "diagnostics": {
    "query": "...",
    "method": "late_fusion"
  },
  "total": 20
}
```

---

# Frontend

The repository contains a React + TypeScript frontend.

Install dependencies:

```bash
cd frontend
npm install
```

Start the development server:

```bash
npm run dev
```

Build for production:

```bash
npm run build
```

Type-check the project:

```bash
npm run type-check
```

The frontend provides components for:

- Text search
- Image upload
- Fusion-method selection
- Search execution
- Product result cards
- Result grids
- Diagnostics

---

# Python Environment

Create a virtual environment:

### Windows

```powershell
python -m venv .venv
.venv\Scripts\activate
```

### Linux / macOS

```bash
python3 -m venv .venv
source .venv/bin/activate
```

Install backend dependencies:

```bash
pip install -r backendapi/requirements.txt
```

Install ML dependencies:

```bash
pip install -r ml/requirements.txt
```

Depending on the installed PyTorch/Transformers versions and operating system, the first CLIP inference may download model assets.

---

# Running the ML Pipelines

The repository contains command-line entry points for the fusion pipelines.

## Early Fusion

```bash
python -m src.pipeline.run_early_fusion \
  --image path/to/image.jpg \
  --text "blue cotton shirt size M"
```

## Static Late Fusion

```bash
python -m src.pipeline.run_late_fusion \
  --image path/to/image.jpg \
  --text "blue cotton shirt under 1500"
```

## Intent-Aware Dynamic Fusion

The intent-aware implementation is located in:

```text
ml/training/intent_aware_fusion.py
```

It can operate in dynamic mode or static baseline mode.

Conceptually:

```text
Query
  ↓
Constraint Parsing
  ↓
Intent Classification
  ↓
Dynamic Weight Selection
  ↓
Candidate Retrieval
  ↓
Multisignal Ranking
  ↓
Top-K Products
```

---

# Example Queries

### Visual Query

```text
blue jacket
```

Expected intent:

```text
visual
```

### Attribute Query

```text
cotton medium jacket under 2000
```

Expected intent:

```text
attribute
```

### Hybrid Query

```text
blue cotton jacket size M
```

Expected intent:

```text
hybrid
```

### More Complex Query

```text
black leather sneakers size 9 under 3000
```

The parser attempts to extract:

```text
color       → black
material    → leather
category    → sneakers
size        → 9
price_max   → 3000
```

The remaining terms are retained as search keywords where applicable.

---

# How the Ranking Pipeline Works

```text
1. Receive query
        │
        ▼
2. Parse text constraints
        │
        ├── category
        ├── color
        ├── size
        ├── material
        ├── price
        └── keywords
        │
        ▼
3. Process image with CLIP
        │
        ├── category
        └── color
        │
        ▼
4. Merge symbolic constraints
        │
        ▼
5. Retrieve candidate products
        │
        ▼
6. Calculate ranking signals
        │
        ├── visual score
        ├── constraint score
        └── review score
        │
        ▼
7. Apply fusion strategy
        │
        ├── Early fusion
        ├── Static late fusion
        └── Intent-aware fusion
        │
        ▼
8. Sort candidates
        │
        ▼
9. Return Top-K products
```

---

# What Makes the Approach Different

The research design combines several elements that are usually handled separately:

```text
Image
  +
Natural Language
  +
Hard/Structured Constraints
  +
Review Quality
  +
Query Intent
```

Rather than using a single dense multimodal representation for every query, the system converts important attributes into interpretable symbolic fields and then performs configurable ranking.

The paper specifically presents the combination of image attributes, structured constraints, review quality, and intent-aware fusion as the central design point. fileciteturn0file0L319-L324

---

# Limitations

The research evaluation has important limitations that should be considered when interpreting the reported metrics.

### 1. Synthetic / Product-Derived Queries

The 1,500 evaluation queries were generated from product information, meaning the queries are inherently connected to a known target product.

This provides a controlled evaluation setting but does not fully represent arbitrary real-world user queries. fileciteturn0file0L272-L275

### 2. Visual Evaluation

The paper notes that the evaluation's visual score is based on product-title matching rather than complete image-to-image matching.

Consequently, the reported benchmark does not fully measure the potential of CLIP-based image search. fileciteturn0file0L276-L278

### 3. Dataset Scope

The research database was constructed from selected categories of the Amazon Reviews 2023 dataset, rather than representing the entire e-commerce ecosystem.

### 4. Review Dependence

Review quality depends on the availability and quality of product reviews. Products with insufficient usable reviews may be excluded from the ranking database according to the research protocol.

---

# Future Work

The paper proposes several directions for extending the system:

- True image-to-image visual similarity
- Real user-query evaluation
- Personalized ranking based on user behavior
- Voice-based product search
- Video-based product search
- AR-assisted product visualization
- More advanced sentiment and trust analysis
- Fake-review detection
- Conversational multimodal shopping assistants
- Generative AI for product customization
- Edge and vector-indexing optimizations

These directions are discussed in the paper's future-scope section. fileciteturn0file0L340-L352

---

# Research Paper

**Title:** Multimodal Product Search Using Image–Text Prompts and Review-Based Ranking

**Authors:**

- Ojaswin Aggarwal
- Harsh Rathore
- Milandeep Singh
- Abhishek Singhal

**Department:** Computer Science and Engineering, Amity School of Engineering and Technology (ASET), Amity University Uttar Pradesh, Noida, India. fileciteturn0file0L2-L13

The accompanying paper contains the detailed methodology, mathematical formulation, experiments, comparisons, limitations, and future scope.

---

# Authors

| Author | Role |
|---|---|
| Ojaswin Aggarwal | Research / Development |
| Harsh Rathore | Research / Development |
| Milandeep Singh | Research / Development |
| Abhishek Singhal | Research / Development |

---

# License

No license is specified in the supplied project repository.

If this repository is intended for public distribution, add an appropriate license file such as `MIT`, `Apache-2.0`, or another license chosen by the project authors.

---

# Citation

If you use this project in academic work, cite the associated research paper:

```text
Ojaswin Aggarwal, Harsh Rathore, Milandeep Singh,
Abhishek Singhal,
"Multimodal Product Search Using Image–Text Prompts
and Review-Based Ranking."
```

---

## Project Status

This repository contains the implementation of a research-oriented multimodal product search system, including:

- CLIP-based visual attribute extraction
- Rule-based constraint parsing
- SQLite candidate retrieval
- Symbolic early fusion
- Static late fusion
- Intent-aware dynamic fusion
- TF-IDF + Logistic Regression sentiment inference
- Review-aware ranking artifacts
- FastAPI backend
- React/TypeScript frontend
- Evaluation and training scripts
