# Network Security — Phishing Detection ML Pipeline

This repository contains a small end-to-end machine learning pipeline and a FastAPI service for a phishing detection project. It reads data from a MongoDB collection, runs a standard ML training pipeline (ingest → validate → transform → train), stores artifacts under `Artifacts/`, and exposes a /predict endpoint to score uploaded CSVs.

Table of contents

- Project overview
- Architecture and data flow
- Quickstart — run locally
- Environment variables
- Project structure
- Known issues & recommended fixes
- Deployment notes

Project overview
This project trains classification models using `phisingData.csv` data stored in MongoDB. The pipeline stages are implemented under `networksecurity/components/` and are orchestrated by `networksecurity/pipeline/training_pipeline.py` and simple scripts (`main.py`, `app.py`).

Architecture and data flow

- Data ingestion: read documents from MongoDB collection and write CSV artifact (feature store) and train/test CSVs.
- Data validation: validate shape against `data_schema/schema.yaml` and compute drift report using KS-test.
- Data transformation: impute missing values with `KNNImputer` (configured in constants) and save transformed arrays (`.npy`) and the preprocessor (`.pkl`).
- Model training: evaluate several classifiers with `GridSearchCV`, choose a best model, wrap it with the preprocessor into `NetworkModel`, and save as `final_model/model.pkl`.
- Serving: `app.py` exposes `/train` to retrain and `/predict` to score an uploaded CSV and return results in HTML.

Quickstart — run locally

1. Create and activate a Python 3.10+ virtual environment and install requirements:

```bash
python -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
```

````

2. Prepare environment variables (example `.env`):

```text
MONGO_DB_URL=<your_mongo_connection_string>
# Optional for app.py (alternate name used in code):
MONGODB_URL_KEY=<your_mongo_connection_string>

# AWS (only required if you want artifact sync to S3):
AWS_ACCESS_KEY_ID=...
AWS_SECRET_ACCESS_KEY=...
AWS_REGION=us-east-1

# Optional: DagsHub / MLflow credentials used by model trainer
MLFLOW_TRACKING_USERNAME=...
MLFLOW_TRACKING_PASSWORD=...
```

3. Insert data into MongoDB (optional):

- Use `push_data.py` to convert `Network_Data/phisingData.csv` to JSON and insert into your MongoDB collection.

```bash
python push_data.py
```

4. Run training (one of the following):

- Run the procedural script:

```bash
python main.py
```

- Or run the FastAPI app and use the `/train` endpoint:

```bash
python app.py
# then open http://localhost:8000/docs and call POST /train
```

5. Predict using HTTP (FastAPI `/predict`) or programmatically:

- Start the app and POST a CSV file to `/predict` (form field `file`), the response will be an HTML table with predictions. Predictions are also saved to `prediction_output/output.csv`.

Project structure (important modules)

- `app.py` — FastAPI app and endpoints for /train and /predict.
- `main.py` — simple script to run a one-off training run.
- `push_data.py` — helper to push CSV to MongoDB.
- `networksecurity/components/` — pipeline stages: `data_ingestion.py`, `data_validation.py`, `data_transformation.py`, `model_trainer.py`.
- `networksecurity/pipeline/training_pipeline.py` — orchestrates the pipeline and S3 sync.
- `networksecurity/entity/` — dataclasses for configs and artifact shapes.
- `networksecurity/constant/training_pipeline/` — constants used across the pipeline (target column, filenames, KNN params, DB collection names).
- `networksecurity/utils/` — helper utils for IO, serialization, model wrapper, and metrics.

Known issues & recommended fixes

- Metric mismatch: the model selection routine (`evaluate_models`) uses `r2_score` (a regression metric) to rank classification models. Replace this with a classification metric such as `f1_score` or `roc_auc`.
- MLflow usage: `ModelTrainer.track_mlflow` sets a non-string value as `registered_model_name` and logs models redundantly — this should be adjusted to use a stable model name.
- Env var inconsistency: `MONGO_DB_URL` vs `MONGODB_URL_KEY` are used in different files; unify to a single name to avoid confusion.
- Schema validation is currently only checking column count; prefer validating column names and types.
- Logging directory: the logging module creates nested directories named after the log file; consider simplifying the layout.

Deployment notes

- Docker: the project contains a `Dockerfile`. Build and push to ECR as needed. README originally included EC2 Docker install steps — use them only on fresh instances.
- AWS S3 sync: uploading artifacts uses `aws s3 sync` (OS shell) and expects AWS credentials or IAM role present on the host.
- MongoDB: ensure the MongoDB instance is accessible from where the app runs and credentials are protected (use secrets manager for production).

Next steps (small improvements you can apply now)

1. Fix evaluation metric to use classification metrics and add unit tests for `evaluate_models`.
2. Make MLflow/DagsHub configuration optional and safe (do not hardcode credentials in code). Move config to env.
3. Add stronger schema validation (column names/types) and unit tests for DataValidation.

Contact / author

- Author: Ashwary Gupta


### Network Security Projects For Phising Data

Setup github secrets:
AWS_ACCESS_KEY_ID=

AWS_SECRET_ACCESS_KEY=

AWS_REGION = us-east-1

AWS_ECR_LOGIN_URI = 788614365622.dkr.ecr.us-east-1.amazonaws.com/networkssecurity
ECR_REPOSITORY_NAME = networkssecurity

Docker Setup In EC2 commands to be Executed
#optinal

sudo apt-get update -y

sudo apt-get upgrade

markdown

````
