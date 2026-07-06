# PepperLeaf — Layered Architecture

This backend follows a strict layered architecture. Each layer only talks
to the layer directly below it. This keeps business logic testable without
a database or HTTP server, and keeps the ML model swappable without
touching API code.

```
┌─────────────────────────────────────────────────────────────┐
│  LAYER 1 — PRESENTATION (app/api/v1/)                        │
│  HTTP routes. Parses requests, calls services, returns        │
│  responses. Contains NO business logic.                       │
│  Files: auth.py, detect.py, farms.py, history.py,             │
│         weather.py, assistant.py                              │
└───────────────────────────┬─────────────────────────────────┘
                             │ calls
┌────────────────────────────▼─────────────────────────────────┐
│  LAYER 2 — VALIDATION (app/schemas/)                          │
│  Pydantic models. Defines exact shape of every request/        │
│  response. Validates input before it reaches business logic.   │
│  Files: auth.py, farm.py, scan.py, weather.py                  │
└───────────────────────────┬─────────────────────────────────┘
                             │ used by
┌────────────────────────────▼─────────────────────────────────┐
│  LAYER 3 — SECURITY (app/core/security.py, app/core/deps.py)  │
│  Password hashing, JWT issue/verify, the get_current_user      │
│  dependency that protects routes.                              │
└───────────────────────────┬─────────────────────────────────┘
                             │ used by
┌────────────────────────────▼─────────────────────────────────┐
│  LAYER 4 — BUSINESS LOGIC (app/services/)                      │
│  Pure functions/classes with the actual rules: register a      │
│  user, calculate outbreak risk, issue tokens. Testable with     │
│  zero HTTP or DB mocking beyond a session object.               │
│  Files: auth_service.py, weather_service.py                    │
└───────────────────────────┬─────────────────────────────────┘
                             │ calls
┌────────────────────────────▼─────────────────────────────────┐
│  LAYER 5 — ML / INFERENCE (app/ml/)                             │
│  Model loading, prediction, Grad-CAM. Knows nothing about       │
│  HTTP, JWTs, or the database — takes a file path, returns a     │
│  dict.                                                          │
│  Files: inference_service.py, gradcam_service.py                │
└───────────────────────────┬─────────────────────────────────┘
                             │ persists via
┌────────────────────────────▼─────────────────────────────────┐
│  LAYER 6 — PERSISTENCE (app/models/, app/db/)                  │
│  SQLAlchemy ORM models + session factory. The only layer        │
│  that knows SQL exists.                                         │
│  Files: user.py, farm.py, scan.py, model_version.py, session.py │
└───────────────────────────┬─────────────────────────────────┘
                             │ configured by
┌────────────────────────────▼─────────────────────────────────┐
│  LAYER 7 — CONFIGURATION (app/core/config.py)                  │
│  Single source of truth for every environment variable.         │
│  Everything above imports `settings` from here — nothing        │
│  reads os.environ directly outside this file.                   │
└─────────────────────────────────────────────────────────────┘

CROSS-CUTTING (touches every layer, but isolated):
  app/middleware/   — request logging, rate limiting
  app/core/exceptions.py — global error handlers
  app/utils/        — small stateless helpers (file validation, etc.)
```

## The dependency rule

Code in a lower-numbered layer NEVER imports from a higher-numbered layer.
`app/ml/inference_service.py` (Layer 5) has no idea FastAPI exists.
`app/services/auth_service.py` (Layer 4) has no idea what a JWT looks like
on the wire — it just calls `create_access_token()` from Layer 3.

This is what makes `tests/test_inference.py` able to test the model with
zero HTTP server running, and `tests/test_auth.py` able to test password
hashing with zero database.

## Folder map

```
pepperleaf_pro/
├── app/
│   ├── main.py                    # Wires all layers together, app factory
│   ├── api/v1/                    # LAYER 1 — Presentation
│   │   ├── auth.py                #   /api/v1/auth/*
│   │   ├── detect.py              #   /api/v1/detect/*
│   │   ├── farms.py               #   /api/v1/farms/*
│   │   ├── history.py             #   /api/v1/history/*
│   │   ├── weather.py             #   /api/v1/weather/*
│   │   └── assistant.py           #   /api/v1/assistant/*
│   ├── schemas/                   # LAYER 2 — Validation
│   │   ├── auth.py
│   │   ├── farm.py
│   │   ├── scan.py
│   │   └── weather.py
│   ├── core/                      # LAYER 3 — Security + LAYER 7 — Config
│   │   ├── config.py              #   Settings (env vars)
│   │   ├── security.py            #   Password hashing, JWT
│   │   ├── deps.py                #   get_current_user dependency
│   │   └── exceptions.py          #   Global error handlers
│   ├── services/                  # LAYER 4 — Business logic
│   │   ├── auth_service.py
│   │   └── weather_service.py
│   ├── ml/                        # LAYER 5 — ML / Inference
│   │   ├── inference_service.py
│   │   └── gradcam_service.py
│   ├── models/                    # LAYER 6 — Persistence (ORM)
│   │   ├── user.py
│   │   ├── farm.py
│   │   ├── scan.py
│   │   └── model_version.py
│   ├── db/
│   │   └── session.py             # Engine, SessionLocal, get_db, init_db
│   ├── middleware/                 # Cross-cutting
│   │   ├── logging.py
│   │   └── rate_limit.py
│   └── utils/                      # Cross-cutting
│       └── file_validation.py
├── tests/
│   ├── conftest.py                 # Shared fixtures (test DB, test client)
│   ├── test_auth.py                # Layer 3 unit tests
│   └── test_inference.py           # Layer 5 unit tests
├── scripts/
│   └── seed_db.py                  # Seeds demo user/farms/real scan history
├── ml/
│   └── saved_models/
│       ├── pepper_disease_model.h5
│       └── training_metrics.json
├── ml_training/
│   ├── train.py                    # Standalone training script
│   └── split_dataset.py            # Standalone dataset splitter
├── data/
│   └── real/                       # train/val/test split (gitignored, large)
├── requirements.txt                 # Grouped by layer with comments
├── Dockerfile
├── docker-compose.yml
├── .env.example
└── README.md
```

## Request flow example: uploading a leaf photo

```
1. POST /api/v1/detect/ hits app/api/v1/detect.py        (Layer 1)
2. FastAPI validates the multipart form against the route
   signature, response shaped by app/schemas/scan.py      (Layer 2)
3. get_current_user dependency decodes the JWT from the
   Authorization header                                    (Layer 3)
4. Route checks the farm belongs to that user via a query
   against app/models/farm.py                               (Layer 6)
5. Route calls predict() and generate_gradcam_base64()
   from app/ml/                                              (Layer 5)
6. Route builds a Scan ORM object and commits it             (Layer 6)
7. Response serialized through DetectionResult schema
   and returned                                               (Layer 2 → 1)
```

Every step in that list is one function call into the layer below — no
layer skips ahead or reaches backward.
