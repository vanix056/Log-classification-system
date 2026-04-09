# Log Classification System

## Overview

The Log Classification System is a hybrid ML-powered pipeline that automatically categorizes log messages from multiple enterprise sources into structured labels. It addresses the challenge of processing heterogeneous log data at scale by combining three complementary approaches: rule-based regex matching, sentence-embedding classification, and LLM inference. The result is a robust, source-aware classification engine exposed as a REST API, enabling downstream teams to consume labeled log data directly.

## Key Features

- **Hybrid classification pipeline** — routes each log through the optimal strategy based on source and message structure
- **Regex engine** — fast, deterministic matching for well-structured log patterns (e.g., backups, reboots, file uploads)
- **BERT-based classifier** — uses `all-MiniLM-L6-v2` sentence embeddings with a Logistic Regression model for semantically complex logs
- **LLM fallback (Groq/DeepSeek)** — delegates ambiguous `LegacyCRM` logs to a large language model for contextual reasoning
- **Source-aware routing** — applies the appropriate strategy per log source (LegacyCRM vs. all others)
- **REST API** — FastAPI endpoint accepts CSV uploads and returns a labeled CSV for batch processing
- **9 classification labels** — HTTP Status, Critical Error, Error, Security Alert, System Notification, Resource Usage, User Action, Workflow Error, Deprecation Warning

## Tech Stack

| Category | Technology |
|---|---|
| Language | Python 3.x |
| API Framework | FastAPI |
| ML / Embeddings | `sentence-transformers` (`all-MiniLM-L6-v2`) |
| ML Classifier | scikit-learn (Logistic Regression) |
| LLM Provider | Groq API (`deepseek-r1-distill-llama-70b`) |
| Data Processing | pandas |
| Model Serialization | joblib |
| Environment Config | python-dotenv |
| Training / EDA | Jupyter Notebook |

## Installation

### Prerequisites

- Python 3.9+
- A [Groq API key](https://console.groq.com/) for LLM-based classification

### Steps

```bash
# Clone the repository
git clone https://github.com/vanix056/Log-classification-system.git
cd Log-classification-system

# Create and activate a virtual environment
python -m venv venv
source venv/bin/activate   # On Windows: venv\Scripts\activate

# Install dependencies
pip install fastapi uvicorn pandas scikit-learn sentence-transformers groq python-dotenv joblib
```

### Environment Setup

Create a `.env` file in the project root:

```env
GROQ_API_KEY=your_groq_api_key_here
```

## Usage

### Run the API Server

```bash
uvicorn server:app --reload
```

The API will be available at `http://127.0.0.1:8000`.

### Classify Logs via API

Send a POST request with a CSV file containing `source` and `log_message` columns:

```bash
curl -X POST "http://127.0.0.1:8000/classify/" \
  -F "file=@test.csv" \
  --output classified_output.csv
```

The response is a CSV file with an additional `target_label` column.

### Run Classification Directly (CLI)

```bash
python classify.py
```

This reads `test.csv`, classifies each log entry, and writes `output.csv`.

## Project Structure

```
Log-classification-system/
├── server.py              # FastAPI application — /classify/ endpoint
├── classify.py            # Main classification router (regex → BERT → LLM)
├── log_regex.py           # Rule-based regex classifier
├── bert.py                # Sentence-transformer + Logistic Regression classifier
├── llm.py                 # Groq LLM classifier (for LegacyCRM source)
├── models/
│   └── log_classifier.joblib   # Trained Logistic Regression model
├── training/
│   ├── training.ipynb          # EDA, clustering, and model training notebook
│   └── dataset/
│       └── synthetic_logs.csv  # Labelled training dataset
├── resources/
│   └── output.csv              # API output (generated at runtime)
└── test.csv                    # Sample input for testing
```

## Configuration

| Variable | Description |
|---|---|
| `GROQ_API_KEY` | API key for the Groq inference service. Required for LLM classification of `LegacyCRM` log messages. Set in `.env`. |

The `resources/output.csv` path is the default output location used by the API server. This can be changed in `server.py`.

## Model Architecture

The classification pipeline uses a **three-tier routing strategy**:

1. **Regex Matcher** (`log_regex.py`) — Applies pattern-matching rules to identify simple, structured events such as logins, backups, and system reboots. Returns immediately if a match is found.

2. **BERT + Logistic Regression** (`bert.py`) — For non-LegacyCRM logs not matched by regex, the `all-MiniLM-L6-v2` model generates 384-dimensional sentence embeddings. A pre-trained Logistic Regression classifier (`models/log_classifier.joblib`) predicts the label. Predictions below 50% confidence are returned as `Unclassified`.

3. **LLM Classifier** (`llm.py`) — For all `LegacyCRM`-sourced logs, a prompt is sent to the `deepseek-r1-distill-llama-70b` model via the Groq API to classify into `Workflow Error` or `Deprecation Warning`.

## Dataset

- **File**: `training/dataset/synthetic_logs.csv`
- **Size**: ~2,500+ labelled log entries
- **Sources**: `ModernCRM`, `AnalyticsEngine`, `ModernHR`, `BillingSystem`, `ThirdPartyAPI`, `LegacyCRM`
- **Labels**: HTTP Status, Critical Error, Error, Security Alert, System Notification, Resource Usage, User Action, Workflow Error, Deprecation Warning
- **Complexity tiers**: `regex` (rule-matchable), `bert` (embedding-based), `llm` (LegacyCRM-only)

## Training

The training workflow is documented in `training/training.ipynb`:

1. Load and explore `synthetic_logs.csv`
2. Use DBSCAN clustering on sentence embeddings to identify log patterns
3. Apply regex rules to label simple patterns
4. Separate remaining non-LegacyCRM logs for supervised classification
5. Encode log messages with `all-MiniLM-L6-v2`
6. Train a Logistic Regression classifier on the embeddings
7. Evaluate and serialize the model to `models/log_classifier.joblib`

```bash
# Open the training notebook
jupyter notebook training/training.ipynb
```

## Evaluation Metrics

Evaluated on a 30% held-out test split of non-regex, non-LegacyCRM logs:

| Class | Precision | Recall | F1-Score | Support |
|---|---|---|---|---|
| Critical Error | 0.91 | 1.00 | 0.95 | 48 |
| Error | 0.98 | 0.89 | 0.93 | 47 |
| HTTP Status | 1.00 | 1.00 | 1.00 | 304 |
| Resource Usage | 1.00 | 1.00 | 1.00 | 49 |
| Security Alert | 1.00 | 0.99 | 1.00 | 123 |
| **Overall Accuracy** | | | **0.99** | **571** |

## API Endpoints

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/classify/` | Upload a CSV with `source` and `log_message` columns; returns a labeled CSV |

**Request**: Multipart form upload with a `.csv` file.

**Required CSV columns**:
- `source` — the system that generated the log (e.g., `ModernCRM`, `LegacyCRM`)
- `log_message` — the raw log text

**Response**: A CSV file with an additional `target_label` column containing the predicted classification.

**Error responses**:
- `400` — File is not a CSV, or required columns are missing
- `500` — Internal classification error

## Contributing

1. Fork the repository and create a feature branch from `main`.
2. Follow the existing code structure — add new classifiers as separate modules.
3. Ensure new classification strategies integrate with the routing logic in `classify.py`.
4. Test changes against `test.csv` before submitting a pull request.
5. Keep pull request descriptions concise and focused on a single concern.

## License

This project is licensed under the [MIT License](LICENSE).

## Author

Developed and maintained by **vanix056**.  
For questions or contributions, open an issue or pull request on [GitHub](https://github.com/vanix056/Log-classification-system).
