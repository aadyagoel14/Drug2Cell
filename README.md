# Drug2Cell
An Agentic AI System for In-Silico Drug Response Prediction

Select a drug and cancer cell type — get genome-wide gene expression predictions in seconds, backed by PubMed literature and a RAG-powered chatbot that explains *why*.

> ⚠️ **Code not available due to NIH contract restrictions ** 

---

## What it does

Traditional drug screening takes weeks of wet lab work per candidate. **Drug2Cell** collapses that to seconds:

1. **Predict** — select a drug + cancer cell type; the model returns predicted gene expression changes across the transcriptome
2. **Visualize** — results rendered as UMAPs, violin plots, UMI/gene count distributions, and cells-per-drug charts
3. **Understand** — a PubMed-grounded RAG chatbot explains the biology behind the prediction, cites sources, and suggests alternative compounds to explore

> ⚠️ **Research use only.** Predictions are for exploratory in-silico analysis — not for clinical decision-making.

---

## Demo

> Try it with **Ellagic acid** on **epithelial cell cancer** using the "Try Example" button.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│                    Next.js Frontend                  │
│   Drug + Cell Type selector  │  Results Dashboard   │
│   UMAPs / Violin / Histograms │  RAG Chatbot        │
└──────────────────┬──────────────────────────────────┘
                   │ API routes
       ┌───────────┴───────────┐
       ▼                       ▼
┌─────────────┐       ┌────────────────────┐
│  ML Model   │       │   RAG Pipeline     │
│  (CVAE-GAN) │       │  PubMedBERT +      │
│             │       │  ChromaDB          │
│  Predicts   │       │                   │
│  Δ gene     │       │  Retrieves +       │
│  expression │       │  grounds responses │
└─────────────┘       └────────┬───────────┘
                               │
                       ┌───────▼───────┐
                       │  OpenRouter   │
                       │  LLM gateway  │
                       └───────────────┘
```

### ML Model — CVAE-GAN

A custom generative model built from scratch in PyTorch, trained on the [Sciplex3](https://www.science.org/doi/10.1126/science.aax6234) dataset (677,143 cells, 188 drugs, 3 cancer cell lines).

| Component | Details |
|:---|:---|
| **CVAEEncoder** | `[expression(3000) ‖ condition(64)]` → 512 → 256 → latent(64) |
| **ConditionEncoder** | Cell line embedding + drug fingerprint (Morgan, 1024-dim) + log dose |
| **CVAEDecoder** | Latent → 256 → 512 → NB output heads (μ, r, Δ) |
| **Discriminator** | WGAN-GP with spectral normalization |
| **Total params** | ~8.3M |

### RAG Pipeline

- **Embeddings:** PubMedBERT
- **Vector store:** ChromaDB
- **Corpus:** PubMed (one-time scrape, indexed offline)
- **LLM gateway:** OpenRouter (runtime-configurable model)

---

## Dataset

| Dataset | Description |
|:---|:---|
| **Sciplex3** (primary) | 677,143 cells · 188 drugs · 3 cell lines (MCF7 / A549 / K562) · 5 doses |

**Supported cell lines (MVP):** Epithelial cel · Leukemia · Breast cancer

---

## Baseline Results

The CVAE-GAN is benchmarked against a ladder of baselines on held-out test drugs:

| Model | Pearson R | DEG Jaccard |
|:---|:---:|:---:|
| Zero predictor | 0.000 | 0.017 |
| Global mean | 0.266 | 0.170 |
| Per-cell-line mean | 0.238 | 0.163 |
| Ridge regression | 0.443 | 0.252 |
| Vanilla VAE | 0.579 | 0.307 |
| **CVAE-GAN (target)** | **> 0.65** | **> 0.40** |

---

## Tech Stack

| Layer | Tools |
|:---|:---|
| Frontend | Next.js, React |
| ML model | PyTorch, PyTorch Geometric, RDKit |
| Genomics | Scanpy, Harmony, GSEApy |
| RAG | LangChain, LangGraph, ChromaDB, PubMedBERT |
| LLM gateway | OpenRouter |
| Data | Sciplex3, LINCS L1000 |

---

## Project Structure

```
drug2cell/
├── src/
│   ├── cvae_encoder.py        # CVAE encoder (1.7M params)
│   ├── cvae_decoder.py        # CVAE decoder + NB output heads
│   ├── condition_encoder.py   # Cell line / drug / dose embeddings
│   ├── discriminator.py       # WGAN-GP discriminator
│   ├── losses.py              # NB loss, KL annealing, WGAN-GP
│   ├── train_cvae_gan.py      # Training loop
│   ├── vae.py                 # Vanilla VAE baseline
│   ├── train_vae.py           # VAE training
│   ├── metrics.py             # Pearson R evaluation
│   ├── deg_overlap.py         # DEG Jaccard metric
│   └── gsea_utils.py          # GSEA wrapper (Hallmarks + Reactome)
├── data/
│   ├── sciplex3/              # h5ad files
│   ├── drug_smiles.json       # 187 drug SMILES (PubChem)
│   ├── drug_split.csv         # Train / val / test drug assignment
│   └── genesets/              # Hallmarks (50) + Reactome (1,818)
├── checkpoints/               # Saved model weights
└── README.md
```

---

## Getting Started

### Prerequisites

- Python 3.10+
- PyTorch 2.x (MPS or CUDA recommended)
- Node.js 18+ (for the web app)

### Install

```bash
git clone https://github.com/your-username/biomedical-ai.git
cd drug2cell
pip install -r requirements.txt
```

### Train the model

```bash
python src/train_cvae_gan.py
```

Checkpoints saved to `checkpoints/cvae_gan_best.pt`. Early stopping on validation Pearson R (patience = 10).

### Run the web app

```bash
npm install
npm run dev
```

Set your OpenRouter API key in `.env.local`:

```
OPENROUTER_API_KEY=your_key_here
```

---
