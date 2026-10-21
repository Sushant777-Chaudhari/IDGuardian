# IDGuardian

IDGuardian is a machine-learning web application that estimates whether a social-media profile appears **genuine or potentially fake**. It combines writing-style signals and profile-behavior signals to produce an explainable assessment.

> **Important:** IDGuardian is a decision-support and academic project, not proof that an account is fake. Scores can be wrong, especially for private, new, low-activity, or unusual accounts. Do not use the result as the sole basis for reporting, banning, or accusing someone.

## Features

- **Multi-platform model support:** Facebook, Instagram, LinkedIn, and Twitter/X.
- **Layered detection pipeline:**
  - **Lexical analysis:** TF-IDF features with Logistic Regression.
  - **Semantic analysis:** sentence embeddings from a Sentence-BERT model, classified with XGBoost and served through ONNX Runtime.
  - **Behavioral analysis:** platform-specific XGBoost models use profile and engagement features.
  - **Trust fusion:** platform-specific Logistic Regression models combine base-layer scores.
- **Explainability:** highlights influential words and behavioral features where available.
- **Demo profile lookup:** bundled sample profile/post CSV files allow the application to be explored without connecting to social-media accounts.
- **Instagram profile scraping endpoint:** attempts to retrieve a profile and recent posts using the configured scraper dependencies; availability depends on Instagram access and may be restricted.
- **History and saved reports:** optional MongoDB Atlas persistence.
- **Operational features:** request rate limits, security headers, rotating logs, a health endpoint, Docker support, and CI workflow.

## Technology stack

- **Backend:** Python, Flask, Waitress
- **Machine learning:** scikit-learn, XGBoost, Transformers, ONNX Runtime, NumPy, pandas, joblib
- **Data / utilities:** Faker, VADER Sentiment
- **Database:** MongoDB Atlas with PyMongo (optional)
- **Frontend:** HTML, CSS, JavaScript
- **Deployment:** Docker

## Repository structure

```text
IDGuardian/
├── app.py                         # Flask application and API routes
├── common.py                      # Shared constants and feature definitions
├── db.py                          # Optional MongoDB history/saved reports
├── instagram_scraper.py           # Instagram scraping integration
├── models/                        # Pre-trained model artifacts
├── data/                           # Demo datasets and model/QA reports
├── templates/                      # HTML pages
├── static/                         # Frontend JavaScript and CSS
├── tests/                           # API, parsing, and scoring tests
├── requirements.txt                # Runtime dependencies
├── requirements-train.txt          # Model-training dependencies
├── Dockerfile
└── .env.example                    # Example environment configuration
```

## Requirements

- Python 3.12 recommended (matches the Docker image).
- `pip` and `venv`.
- Internet access on first startup if the semantic model needs to be downloaded from Hugging Face.
- A MongoDB Atlas URI only if you want server-side history and saved reports.
- Instagram access may be restricted by login requirements, rate limits, or platform changes.

## Run locally

### 1. Clone or extract the repository

```bash
git clone <your-repository-url>
cd IDGuardian
```

If you downloaded the ZIP, extract it and open a terminal in the `IDGuardian` project directory.

### 2. Create and activate a virtual environment

**Windows PowerShell**
```powershell
py -3.12 -m venv .venv
.\.venv\Scripts\Activate.ps1
```

**Linux / macOS**
```bash
python3.12 -m venv .venv
source .venv/bin/activate
```

### 3. Install dependencies

```bash
python -m pip install --upgrade pip
pip install -r requirements.txt
```

### 4. Configure MongoDB (optional)

The application can start without MongoDB, but server-side history and saved reports will be unavailable.

```bash
cp .env.example .env
```

On Windows, copy `.env.example` to `.env` manually if `cp` is unavailable. Edit `.env` and replace the placeholder with your MongoDB Atlas connection string:

```dotenv
MONGODB_URI=mongodb+srv://<username>:<password>@<cluster-host>/?retryWrites=true&w=majority
```

Keep `.env` private. Do not commit real credentials or API/session cookies to source control.

### 5. Start the application

```bash
python app.py
```

Open **http://127.0.0.1:5000** in your browser.

To change the bind address or port, set `HOST` and `PORT` environment variables. For example, on Linux/macOS:

```bash
HOST=0.0.0.0 PORT=5000 python app.py
```

On PowerShell:

```powershell
$env:HOST="0.0.0.0"
$env:PORT="5000"
python app.py
```

The app loads model artifacts from `models/` and demo profile data from `data/` at startup. If the semantic layer cannot load, the app may continue in degraded mode with the other available layers; check the startup output and `/healthz`.

## Run with Docker

Build the image from the project root:

```bash
docker build -t idguardian .
```

Run it:

```bash
docker run --rm -p 5000:5000 --env-file .env idguardian
```

If you are not using MongoDB, omit `--env-file .env` (or provide an environment file without `MONGODB_URI`). Then open **http://localhost:5000**.

The image uses Python 3.12, runs the application as a non-root user, and exposes port `5000`.

## API endpoints

| Method | Endpoint | Purpose |
|---|---|---|
| `GET` | `/` | Landing page |
| `GET` | `/dashboard` | Main application dashboard |
| `GET` | `/login` | Login page |
| `GET` | `/api/lookup?query=<username-or-profile-url>` | Analyze a matching bundled demo profile |
| `POST` | `/api/scrape` | Attempt Instagram profile retrieval |
| `GET` | `/api/history?client_id=<id>` | Retrieve analysis history (MongoDB required) |
| `POST` | `/api/history` | Save an analysis to history (MongoDB required) |
| `DELETE` | `/api/history/<entry_id>?client_id=<id>` | Delete one history entry |
| `DELETE` | `/api/history?client_id=<id>` | Clear history |
| `GET` | `/api/saved?client_id=<id>` | Retrieve saved reports (MongoDB required) |
| `POST` | `/api/saved` | Save a report (MongoDB required) |
| `DELETE` | `/api/saved/<entry_id>?client_id=<id>` | Delete one saved report |
| `DELETE` | `/api/saved?client_id=<id>` | Clear saved reports |
| `GET` | `/healthz` | Application/model/database health status |

`/api/lookup` uses the included demo dataset; it does **not** automatically analyze every public social-media URL. The Instagram scraping endpoint is separate and can fail when the platform blocks or limits requests.

## Model artifacts and training

Pre-trained model files are stored in `models/`. The application expects these files to be present. The repository also includes training and dataset-generation scripts, including:

- `generate_dataset.py` and `generate_demo_dataset.py`
- `train_models.py`
- `qa_report.py` and `error_analysis.py`
- `compute_model_curves.py`, `compute_reference_stats.py`, and `compute_lexical_word_stats.py`

Training dependencies are separated from serving dependencies. Review `requirements-train.txt` before running training. Training can require additional memory, disk space, and time compared with running the web application.

For dataset assumptions and implementation decisions, see `DATASET_GENERATION_SPEC.md`, `NOTES.md`, and the reports in `data/`.

## Tests

Install the test dependencies if needed:

```bash
pip install -r requirements-qa.txt
```

Run the test suite with:

```bash
python -m pytest
```

If `pytest` is not installed in your environment, install the development/test dependencies listed in `requirements-dev.txt` as appropriate.

## Security and privacy notes

- Do not commit `.env`, database credentials, cookies, or session data.
- The application includes rate limiting and common security response headers, but a production deployment still needs HTTPS, secret management, monitoring, and a review of its authentication and authorization requirements.
- History and saved reports use a browser-generated `client_id`; this is not equivalent to authenticated user identity. Do not treat it as strong access control for sensitive data.
- Only analyze data you are authorized to access and comply with the relevant platform terms and privacy laws.
- Treat model outputs as probabilistic indicators, not definitive judgments.

## Limitations

- Performance depends on the training data and may not generalize to real-world accounts or platform changes.
- Missing or incomplete profile information can reduce reliability.
- Semantic scoring depends on successful model download/loading; check `/healthz` and application logs.
- Demo lookup and live Instagram scraping are different workflows.
- MongoDB-backed history/saved reports require a reachable Atlas cluster and a valid `MONGODB_URI`.

## Future improvements

- Evaluate on a larger, independently collected and ethically sourced real-world dataset.
- Add calibration and platform-specific threshold selection.
- Expand authorized data connectors and improve graceful handling of unavailable fields.
- Strengthen authentication and per-user authorization before storing sensitive reports.
- Add deployment monitoring and reproducible model/version tracking.


