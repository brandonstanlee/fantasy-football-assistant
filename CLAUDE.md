# Fantasy Football Assistant

A FastAPI service that serves ML-powered weekly fantasy football projections and start/sit recommendations. Users manage leagues, rosters, and watchlists through a CRUD API, and per-position models trained on nflverse data project weekly player points.

## Tech stack

- **Backend**: FastAPI, SQLModel (or SQLAlchemy + Alembic for migrations), Pydantic for DTOs
- **Database**: SQLite in development, PostgreSQL in production
- **Data source**: nflverse. `nfl_data_py` has been the standard package, but nflverse has been moving to `nflreadpy`. Verify which is currently maintained before committing to one.
- **ML**: scikit-learn and/or XGBoost/LightGBM (tabular data like this usually beats deep learning)
- **Testing**: pytest
- **CI/CD**: GitHub Actions running tests on every push
- **Deploy target**: Render, Fly.io, or Railway
- **Frontend**: Streamlit first, React optional

## Architecture

```
nflverse data ──► ingestion job ──► PostgreSQL ◄── FastAPI ◄── frontend
                                        ▲             ▲
                                        │             │
                              training pipeline ──► model registry (versioned files)
```

## Project structure

```
fantasy-football-assistant/
├── app/
│   ├── main.py
│   ├── models.py      # database tables
│   ├── schemas.py     # DTOs (Pydantic)
│   ├── routers/
│   └── db.py
├── ml/
│   ├── features.py
│   └── train.py
├── tests/
└── pyproject.toml
```

## Data model

| Table | Key fields |
|---|---|
| `users` | id, email, password_hash |
| `leagues` | id, user_id, name, scoring_format (PPR/half/standard), roster_slots |
| `roster_entries` | id, league_id, player_id, slot (starter/bench/IR), added_at |
| `watchlist` | id, user_id, player_id, note |
| `players` | id, name, position, team (loaded from nflverse) |
| `player_weeks` | player_id, season, week, stats columns (loaded from nflverse) |
| `projections` | player_id, season, week, predicted_points, model_version, created_at |

## API endpoints

**CRUD core**
- `POST /auth/register`, `POST /auth/login`
- `POST/GET/PATCH/DELETE /leagues`
- `POST/GET/PATCH/DELETE /leagues/{id}/roster`
- `POST/GET/DELETE /watchlist`

**Model-powered**
- `GET /players/{id}/projection?week=N`
- `POST /leagues/{id}/startsit`: takes candidate player IDs, returns them ranked by projected points
- `GET /model/info`: version, training date, evaluation metrics

## DTO conventions

- Never return database models directly from the API. Always use separate Pydantic schemas.
- Separate request and response schemas per resource, e.g. `LeagueCreate` (name, scoring_format) in, `LeagueRead` (id, name, scoring_format, roster_count) out.
- Incoming schemas must not allow clients to set server-controlled fields (ids, `user_id`, timestamps, etc.).
- Response schemas must never expose `password_hash`, raw feature columns, or internal IDs that aren't needed.
- Validate at the boundary (Pydantic constraints, enums for scoring format and roster slot).
- Third-party data (nflverse) is mapped into our own schemas at ingestion; external column names don't leak through the API.

## Auth

- Hashed passwords (bcrypt/argon2 via a maintained library), JWT access tokens.
- Every league, roster, and watchlist query is scoped to the authenticated user. Tests verify that user A cannot read or modify user B's data.

## ML guidelines

- **Baseline first**: predict a player's points as their rolling 3-week average. Every model must beat it.
- Features to explore: rolling averages, opponent defense strength, home/away, snap share, Vegas implied team totals (if obtainable).
- **Time-based splits only.** Train on earlier seasons (e.g. 2018 to 2024), validate on a later one (e.g. 2025). Never shuffle randomly. Features for week N may only use data available before week N.
- Train per-position models (QB, RB, WR, TE behave very differently).
- Evaluate with MAE and rank correlation (e.g. Spearman) per position, compared against the baseline.
- Save each model with a version tag and a metrics file. The API loads a specific version and reports it via `/model/info`.
- Log weekly predictions against actual results so live performance can be reported.

## Roadmap

1. **CRUD foundation**: project skeleton, `/health`, models and migrations, auth, leagues/rosters/watchlist endpoints with DTOs and tests, nflverse ingestion script
2. **Model**: baseline, feature engineering, time-based split, per-position models, evaluation, versioned artifacts
3. **Integration and deployment**: serve projections from the API, weekly data and projection refresh job, deploy with CI, track predictions vs. actuals
4. **Polish**: simple frontend, README with architecture diagram and results table

## Development conventions

- Work in small steps. Write tests alongside features (pytest), and don't consider something done if tests fail.
- Use type hints throughout. Keep route handlers thin and put logic in separate modules.
- Use Alembic migrations for schema changes rather than editing tables ad hoc.
- Keep secrets (JWT secret, DB URL) in environment variables and a gitignored `.env` file. Provide a `.env.example`.
- Make small, clear git commits.
- Keep dependencies minimal, and note what each one is for.
- When a design choice has real tradeoffs (e.g. SQLModel vs. SQLAlchemy, nfl_data_py vs. nflreadpy), present the options and recommend one rather than silently picking.
