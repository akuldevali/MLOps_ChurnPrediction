# MLOps Churn Prediction

An end-to-end MLOps project that predicts customer churn for a telecom business. It covers the full ML lifecycle: synthetic data generation, model training, local inference API, automated CI/CD with GitHub Actions, artifact storage on AWS S3 with DVC, and production model serving on Kubernetes via KServe.

---

## Project Overview

**Goal:** Predict the probability that a customer will cancel their subscription based on account features such as tenure, monthly charges, and support call frequency.

**Model:** A Random Forest Classifier (scikit-learn) trained on synthetic telecom data with 1,000 samples.

**Input features:**

| Feature | Type | Description |
|---|---|---|
| `age` | int | Customer age (18–70) |
| `tenure_months` | int | Months as a customer (1–72) |
| `monthly_charges` | float | Monthly subscription cost ($20–$120) |
| `total_charges` | float | Cumulative amount spent ($100–$8,000) |
| `num_support_calls` | int | Support contacts this month (0–10) |

**Output:** `churn` (0 = stays, 1 = leaves) and `churn_probability` (0.0–1.0).

**Example:** A customer paying $95/month who has called support 5 times and has only been a customer for 6 months will likely have a high churn probability. The business can then proactively offer a discount or reach out before the customer leaves.

---

## Architecture

```
generate_data.py  ──►  data/churn_data.csv
                                │
                         train.py
                                │
                         models/churn_model.pkl
                                │
              ┌─────────────────┼───────────────────────┐
              │                 │                       │
           api.py          DVC push               GitHub Actions
        (local REST)      (S3 remote)           (auto-train + push)
                                                        │
                                              manifests/inference.yaml
                                                        │
                                              KServe InferenceService
                                             (Kubernetes production)
```

**Tech stack:**

- **ML:** scikit-learn, pandas, numpy
- **API:** FastAPI + uvicorn
- **Artifact versioning:** DVC + AWS S3
- **CI/CD:** GitHub Actions
- **Production serving:** KServe on Kubernetes

---

## Repository Structure

```
.
├── generate_data.py          # Generates synthetic churn dataset
├── train.py                  # Trains and saves the Random Forest model
├── api.py                    # FastAPI local inference server
├── requirements.txt          # Python dependencies
├── data/
│   ├── churn_data.csv        # Training dataset (tracked by DVC)
│   └── churn_data.csv.dvc    # DVC metadata pointer
├── models/
│   └── churn_model.pkl       # Trained model (tracked by Git LFS)
├── manifests/
│   └── inference.yaml        # KServe InferenceService manifest
├── s3-secret.yaml            # Kubernetes Secret for S3 credentials
├── svcaccount.yaml           # Kubernetes ServiceAccount
└── .github/workflows/
    └── mlops-pipeline.yaml   # GitHub Actions CI/CD pipeline
```

---

## Setup

### Prerequisites

- Python 3.11
- `git` and `git-lfs`
- AWS credentials with S3 access (for DVC remote)
- `kubectl` + a running Kubernetes cluster with KServe installed (for production serving only)

### 1. Clone the repository

```bash
git clone https://github.com/akuldevali/MLOps_ChurnPrediction.git
cd MLOps_ChurnPrediction
```

### 2. Create a virtual environment and install dependencies

```bash
python -m venv venv
source venv/bin/activate        # Windows: venv\Scripts\activate
pip install -r requirements.txt
```

### 3. Configure AWS credentials

Export them as environment variables, or create a `.env` file (never commit this):

```bash
export AWS_ACCESS_KEY_ID=<your-access-key>
export AWS_SECRET_ACCESS_KEY=<your-secret-key>
```

---

## Local Usage

### Generate data

```bash
python generate_data.py
# Creates data/churn_data.csv with 1,000 synthetic customer samples
```

### Train the model

```bash
python train.py
# Trains a Random Forest, prints accuracy + AUC-ROC, saves models/churn_model.pkl
```

### Run the inference API

```bash
python api.py
# Starts server at http://127.0.0.1:8000
# Interactive docs at http://127.0.0.1:8000/docs
```

**Health check:**

```bash
curl http://localhost:8000/health
```

**Prediction:**

```bash
curl -X POST http://localhost:8000/predict \
  -H "Content-Type: application/json" \
  -d '{
    "age": 45,
    "tenure_months": 6,
    "monthly_charges": 95.00,
    "total_charges": 570.00,
    "num_support_calls": 5
  }'
```

Response:

```json
{
  "churn": 1,
  "churn_probability": 0.73
}
```

---

## DVC — Data & Model Versioning

DVC versions large files (dataset, model) outside of Git, with AWS S3 as the remote backend.

```bash
# Push local data/models to S3
dvc push

# Pull data/models from S3
dvc pull
```

The remote is configured in `.dvc/config` pointing to `s3://mlops-churn-akuldevali`.

---

## CI/CD Pipeline

The GitHub Actions workflow (`.github/workflows/mlops-pipeline.yaml`) runs automatically on every push to `main`:

1. Checks out the code
2. Installs Python 3.11 dependencies
3. Generates fresh training data via `generate_data.py`
4. Trains the model via `train.py`
5. Uploads `models/churn_model.pkl` to S3
6. Updates the S3 model URI in `manifests/inference.yaml`
7. Commits and pushes the updated manifest back to the repo

**Required GitHub Secrets:**

| Secret | Description |
|---|---|
| `AWS_ACCESS_KEY_ID` | AWS access key for S3 writes |
| `AWS_SECRET_ACCESS_KEY` | AWS secret key for S3 writes |

---

## Kubernetes Deployment (KServe)

### Prerequisites

- A Kubernetes cluster with KServe installed
- The cluster must have network access to your S3 bucket

### 1. Create the namespace

```bash
kubectl create namespace svc-account
```

### 2. Apply the S3 credentials secret

Edit `s3-secret.yaml` with your base64-encoded AWS credentials, then apply:

```bash
kubectl apply -f s3-secret.yaml
```

### 3. Apply the ServiceAccount

```bash
kubectl apply -f svcaccount.yaml
```

### 4. Deploy the InferenceService

```bash
kubectl apply -f manifests/inference.yaml
```

### 5. Check status

```bash
kubectl get inferenceservice -n svc-account
kubectl get pods -n svc-account
```

Once the `InferenceService` shows `READY`, KServe exposes a REST endpoint that accepts the same five input features and returns churn predictions.

---

## Security Notes

- Never commit AWS credentials to version control. Use GitHub Secrets for CI/CD and environment variables or IAM roles locally.
- The `.env` file is listed in `.gitignore` — keep it that way.
- Rotate any credentials that have previously been exposed in commits.
