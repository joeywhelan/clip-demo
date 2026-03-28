# Presentation Plan: Multi-Modal Search with Jina ClipV2 on EIS

> **Format:** Single-file HTML (reveal.js CDN), 17 slides
> **Audience:** Technical / engineering
> **Color scheme:** Elastic brand — dark navy (`#1B1C3A`), teal (`#00BFB3`), pink (`#F04E98`), yellow (`#FEC514`), white text on dark backgrounds
> **Assets directory:** `assets/` (images referenced relative to presentation file)

---

## Slide Outline

### Slide 1 — Title
- **"Multi-Modal Search with Jina ClipV2 on Elastic Inference Service"**
- Subtitle: Hardware store product search demo
- Use `assets/cover.png` as background or hero image
- Author name, date

### Slide 2 — The Problem
- Customers describe products in words *or* snap a photo
- Traditional search handles one modality, not both
- Goal: match text queries AND image queries to the same product catalog

### Slide 3 — Why Multi-Modal Embeddings?
- CLIP models map text and images into a **shared vector space**
- Cosine similarity works across modalities
- Diagram concept: show "eye bolt" (text) and photo of eye bolt converging to same region in embedding space
- Mention Jina ClipV2 specifically (1024-dim, multilingual)

### Slide 4 — Architecture Overview
- Use `assets/arch.jpg`
- Three components: **Client** ← Ingest/Query → **Elastic Serverless** ← GPU Embedding → **EIS (Jina ClipV2)**
- Call out: no self-hosted GPU infra required — EIS handles it

### Slide 5 — Infrastructure as Code
- Elastic Serverless project provisioned via **Terraform**
- Jina ClipV2 inference endpoint (`.jina-clip-v2`) available by default
- Show simplified Terraform workflow:
  ```
  terraform init → terraform apply → .env with credentials
  ```
- Key point: reproducible, teardown-friendly demo environment

### Slide 6 — The Dataset
- Synthetic hardware store catalog: fasteners (hex bolts, lag screws, wing nuts, eye bolts, etc.)
- Wikimedia Commons images + generated product metadata (name, description, aisle, bin)
- Stored in `metadata.csv`, images in `images/`
- Show a grid of 4-6 sample product images from `images/`

### Slide 7 — Index Schema
- Elasticsearch index with `dense_vector` field (1024 dims, cosine similarity)
- Show the mapping snippet:
  ```json
  {
    "embedding": { "type": "dense_vector", "dims": 1024, "similarity": "cosine" },
    "name": { "type": "text" },
    "description": { "type": "text" },
    "aisle": { "type": "keyword" },
    "bin": { "type": "keyword" },
    "image_path": { "type": "keyword" }
  }
  ```

### Slide 8 — Generating Image Embeddings
- Two methods: **base64-encoded local file** or **URL**
- Show the base64 code path (simplified):
  ```python
  es.inference.inference(
      inference_id=".jina-clip-v2",
      body={"input": [{"content": {"type": "image",
            "format": "base64", "value": data_uri}}]}
  )
  ```
- Highlight: `tenacity` retry with exponential backoff for 429 rate limits

### Slide 9 — Bulk Indexing Pipeline
- For each product: read image → base64 encode → call EIS → store embedding + metadata
- Uses `elasticsearch.helpers.bulk` for efficient indexing
- Call out production considerations: rate limiting, retries, batch sizing

### Slide 10 — Demo: Image Search
- Query: upload `images/wing_nut_1.jpg`
- Process: image → base64 → EIS embedding → kNN search (`k=3`)
- Show `assets/result-1.png` — top 3 results are all wing nuts
- Key point: visual similarity matching works without any text metadata

### Slide 11 — Demo: Text Search (Cross-Modal)
- Query: `"eye bolt"` (plain text)
- Text embedding compared against **image** embeddings — no text-to-text matching
- Show `assets/result-2.png` — correct eye bolt images returned
- Key point: the model bridges the semantic gap between language and vision

### Slide 12 — In-Store Product Finder: A Real-World Application
- **Scenario:** Customer walks into a hardware store looking for a specific fastener
- They can either describe it ("I need a bolt with a loop on top") or snap a photo of the broken one they're replacing
- The store's kiosk or mobile app sends the query through the same CLIP pipeline
- **Flow diagram concept:**
  ```
  Customer → [Photo or Text] → Store App → EIS (Jina ClipV2)
       → kNN Search → Match Product → Return Aisle & Bin Location
  ```
- Result card shows: product image, name, price, and **exact store location** (Aisle 7, Bin 3B)

### Slide 13 — Store App: Architecture & Data Model
- Extends the demo schema with store-specific fields:
  ```json
  {
    "embedding":    { "type": "dense_vector", "dims": 1024, "similarity": "cosine" },
    "name":         { "type": "text" },
    "description":  { "type": "text" },
    "sku":          { "type": "keyword" },
    "price":        { "type": "float" },
    "aisle":        { "type": "keyword" },
    "bin":          { "type": "keyword" },
    "in_stock":     { "type": "boolean" },
    "quantity":     { "type": "integer" },
    "image_url":    { "type": "keyword" }
  }
  ```
- **Filtered kNN** in action: restrict results to in-stock items only
  ```json
  {
    "knn": {
      "field": "embedding",
      "query_vector": "<customer_query_embedding>",
      "k": 5,
      "num_candidates": 50,
      "filter": { "term": { "in_stock": true } }
    }
  }
  ```

### Slide 14 — Store App: Customer Experience Walkthrough
- **Step 1 — Capture:** Customer photographs a rusted wing nut at the in-store kiosk
- **Step 2 — Embed:** Image is base64-encoded and sent to EIS for embedding
- **Step 3 — Search:** Filtered kNN query finds the closest matching in-stock products
- **Step 4 — Navigate:** App displays a result card:
  ```
  ┌──────────────────────────────────┐
  │  🔩  Wing Nut, 1/4"-20 Zinc     │
  │  SKU: HW-FAS-4421               │
  │  Price: $0.38                    │
  │  ────────────────────────────    │
  │  📍 Aisle 4 · Bin 2C            │
  │  ✅ In Stock (47 remaining)     │
  └──────────────────────────────────┘
  ```
- Optional: integrate with store map for turn-by-turn wayfinding

### Slide 15 — Store App: Why This Beats Traditional Search
- **Keyword search fails:** customer doesn't know the product is called a "wing nut"
- **Category browsing fails:** too many fastener sub-categories to navigate
- **Visual search wins:** photo-to-product match is instant and requires zero domain knowledge
- Cross-modal fallback: if the photo is blurry, customer can describe it ("butterfly-shaped nut") and still get accurate results
- Same embedding index serves both modalities — no separate search infrastructure

### Slide 16 — What's Next
- **Hybrid search**: combine BM25 lexical + vector similarity via Elasticsearch RRF
- **Filtered kNN**: restrict vector search by aisle, category, or price range
- **Multi-field embeddings**: embed product descriptions alongside images for richer recall
- **Production hardening**: async ingestion, monitoring, A/B evaluation of search quality
- **Store app extensions**: multi-store inventory federation, personalized recommendations, purchase history re-ordering

### Slide 17 — Resources & Q&A
- GitHub repo: `github.com/joeywhelan/clip-demo`
- Elastic Inference Service docs link
- Jina ClipV2 model card link
- **Q&A**

---

## Technical Notes

### Framework
- **reveal.js** via CDN (`https://cdn.jsdelivr.net/npm/reveal.js@5/`)
- Single `presentation.html` file, no build step
- Code highlighting via reveal.js built-in highlight plugin

### Elastic Brand Styling
```css
:root {
  --elastic-navy:  #1B1C3A;
  --elastic-teal:  #00BFB3;
  --elastic-pink:  #F04E98;
  --elastic-yellow: #FEC514;
  --elastic-blue:  #0077CC;
  --elastic-white: #FFFFFF;
  --elastic-gray:  #D3DAE6;
}
```
- Slide backgrounds: navy (`#1B1C3A`)
- Headings: teal (`#00BFB3`) or white
- Accent / highlights: pink for callouts, yellow for emphasis
- Code blocks: dark background with light text, teal for keywords
- Font: system sans-serif stack (Inter or similar clean engineering font)

### Image Assets (all in `assets/`)
| File | Used on |
|------|---------|
| `cover.png` | Slide 1 (title) |
| `arch.jpg` | Slide 4 (architecture) |
| `result-1.png` | Slide 10 (image search demo) |
| `result-2.png` | Slide 11 (text search demo) |
| Sample product images from `images/` | Slide 6 (dataset grid) |

### Output
- File: `presentation.html` in project root
- All images referenced via relative paths (`assets/`, `images/`)
- Self-contained — open in any browser, present with arrow keys
