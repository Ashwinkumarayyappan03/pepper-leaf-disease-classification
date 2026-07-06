# PepperLeaf — Production Backend

AI-powered pepper disease detection API with JWT authentication,
layered architecture, real trained CNN model, and Grad-CAM explainability.

See `ARCHITECTURE.md` for the full layer-by-layer breakdown.

## Quick start

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Configure environment
cp .env.example .env
# fill in ANTHROPIC_API_KEY and SECRET_KEY at minimum

# 3. Get the dataset and trained model in place
#    (dataset images go in data/real/{train,val,test}/{healthy,bacterial_spot}/
#     model goes in ml/saved_models/pepper_disease_model.h5 — already included)

# 4. Seed the database with a demo user + real scan history
python scripts/seed_db.py
# Creates: demo@pepperleaf.app / demopassword123

# 5. Run the server
uvicorn app.main:app --reload

# 6. Open the interactive API docs
# http://localhost:8000/docs
```

## Or with Docker

```bash
docker compose up --build
```

## Verified test results

```
Test Accuracy:  99.73%
Test Precision: 99.73%
Test Recall:    99.73%
Test F1:        99.73%
(374 real held-out PlantVillage photos)
```

## API endpoints (all except register/login require a Bearer token)

| Method | Path                          | Layer touched                  |
|--------|-------------------------------|---------------------------------|
| POST   | /api/v1/auth/register         | Services → Persistence          |
| POST   | /api/v1/auth/login             | Services → Security             |
| POST   | /api/v1/auth/refresh            | Security                        |
| GET    | /api/v1/auth/me                  | Security → Persistence          |
| POST   | /api/v1/farms/                    | Persistence                     |
| GET    | /api/v1/farms/                    | Persistence                     |
| GET    | /api/v1/farms/{id}                | Persistence                     |
| PATCH  | /api/v1/farms/{id}                | Persistence                     |
| DELETE | /api/v1/farms/{id}                | Persistence                     |
| POST   | /api/v1/detect/                    | ML → Persistence (full stack)  |
| GET    | /api/v1/history/                   | Persistence                     |
| GET    | /api/v1/weather/{lat}/{lon}        | External service                |
| POST   | /api/v1/assistant/                  | External service (streaming)   |

## Running tests

```bash
pytest tests/ -v
```

## Training your own model

```bash
cd ml_training
# 1. Download PlantVillage from Kaggle, unzip to ./archive
python split_dataset.py     # builds data/real/{train,val,test}/
python train.py              # trains, saves to ml/saved_models/
```

## Verified end-to-end (this was actually run, not just written)

```
1. POST /api/v1/auth/register  -> 201, user created
2. POST /api/v1/auth/login      -> 200, real JWT issued
3. GET  /api/v1/auth/me (no token)     -> 401 rejected correctly
4. GET  /api/v1/auth/me (with token)   -> 200, returns user
5. POST /api/v1/farms/ (with token)     -> 201, farm created
6. POST /api/v1/farms/ (no token)       -> 401 rejected correctly
7. POST /api/v1/detect/ (real photo)    -> 200, real CNN prediction
                                            + real Grad-CAM heatmap
                                            + saved to database
8. GET  /api/v1/history/                 -> 200, shows the scan from step 7
```
