# document-retrieval-system

Offline ingestion pipeline that parses PDF documents, extracts structured metadata, embeds the text with Sentence Transformers, and persists the vectors in a local ChromaDB store. The resulting database is consumed by the [Strengthiva backend](../Strengthiva) RAG endpoint.

## What it does

1. Loads two PDFs from `data/`:
   - 582 pages of diet charts
   - product-to-disease mapping
2. Chunks each PDF and extracts metadata (disease condition, BMI range, gender, meal slot, dosha target, product statuses, etc.)
3. Embeds all chunks with `all-MiniLM-L6-v2`
4. Persists embeddings to `chroma_db/` in a collection named `ayurveda_knowledge_base`

## Prerequisites

- Python 3.14 (pinned in `.python-version`)
- [uv](https://github.com/astral-sh/uv) (recommended) **or** pip

## Setup

### 1. Clone the repository

```bash
git clone <repo-url>
cd document-retrieval-system
```

### 2. Install dependencies

**With uv (recommended):**

```bash
uv sync
```

**With pip:**

```bash
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Verify data files are present

```
data/
├── DietCharts_Ayurveda_cleaned.pdf
└── Product_combination.pdf
```

Both PDFs must be present before running the notebook. They are not committed to the repository.

### 4. Run the ingestion notebook

Open and run all cells in `notebook/document.ipynb`:

```bash
# with uv
uv run jupyter notebook notebook/document.ipynb

# or with an activated venv
jupyter notebook notebook/document.ipynb
```

The notebook will:
- Parse and chunk both PDFs
- Embed all chunks in batches of 64
- Write the persisted ChromaDB store to `chroma_db/` at the project root

The final cell prints the total number of chunks stored (expect ~several hundred).

## Project Structure

```
document-retrieval-system/
├── data/
│   ├── DietCharts_Ayurveda_cleaned.pdf   # Ayurvedic diet charts (input)
│   └── Product_combination.pdf            # Product-disease mapping (input)
├── notebook/
│   └── document.ipynb                     # Ingestion pipeline (run this)
├── chroma_db/                             # Generated — persisted vector store
├── main.py                                # Placeholder entry point
├── pyproject.toml
├── requirements.txt
└── uv.lock
```

## ChromaDB Output

The collection is created at `chroma_db/` (relative to the project root) with:

- **Collection name:** `ayurveda_knowledge_base`
- **Distance metric:** cosine
- **Embedding model:** `all-MiniLM-L6-v2` (384-dimensional)

### Metadata schema

Each chunk carries a subset of these fields depending on document type:

| Field | Values | Source |
|---|---|---|
| `doc_type` | `diet_chart`, `product_recommendation` | All chunks |
| `disease_condition` | e.g. `Diabetes`, `Obesity` | All chunks |
| `bmi_range` | e.g. `25-30`, `>30` | Diet chart chunks |
| `gender` | `Male`, `Female` | Diet chart chunks |
| `meal_slot` | `Breakfast`, `Lunch`, `Dinner`, … | Diet chart chunks |
| `diet_chart_id` | e.g. `CHART_12` | Diet chart chunks |
| `dosha_target` | `Kapha`, `Pitta`, `Vata` | Diet chart chunks |
| `ayurvedic_actions` | e.g. `Medohara, Lekhana` | Diet chart chunks |
| `products` | comma-separated product names | Product chunks |
| `classical_medications` | comma-separated classical formulas | Product chunks |
| `product_statuses` | `Ready`, `Yet to manufacture` | Product chunks |
| `page_number` | integer | All chunks |
| `source` | file path of source PDF | All chunks |

## Connecting to Strengthiva

After ingestion, set `CHROMA_DB_PATH` in the Strengthiva backend's `.env` to the absolute path of the `chroma_db/` directory:

```env
CHROMA_DB_PATH=/absolute/path/to/document-retrieval-system/chroma_db
```

The RAG service will connect to this path and query the `ayurveda_knowledge_base` collection on startup.
