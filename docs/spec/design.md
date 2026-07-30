# Credit Risk / Fraud Detector — Design Spec

**Date:** 2026-07-30
**Owner:** Pedro Giussani
**Status:** Approved, pending implementation plan

## 1. Purpose

A portfolio project built to demonstrate three recurring skill themes across Pedro's job
applications, rather than to demonstrate any single one in isolation:

1. **CI/CD** — a working GitHub Actions pipeline (lint, test, build).
2. **Deployment** — a real, live, containerized service (FastAPI + Docker), not just a notebook.
3. **LLM usage as a productivity/product layer** — used deliberately and narrowly, not as a
   crutch for the core logic.

The domain (credit card fraud detection) is chosen to align with Pedro's top-priority target
sector: fintech risk/fraud/credit-scoring roles (see `CLAUDE.md` in `ai_job_search` —
"Target Sectors", priority 1).

This is **not** primarily a research project. Model performance is secondary to shipping a
complete, working, end-to-end system within the time budget below.

## 2. Constraints

- **Timeline: 1 week**, worked solo alongside an ongoing MSc dissertation and active job
  applications. This is tight; the sequencing in Section 6 exists specifically to protect the
  CI/CD and deployment goals from being cut if the week runs long.
- **Solo build, supervised.** Pedro writes all code. Claude acts as supervisor: writes tickets
  (as GitHub Issues), reviews PRs, and gives guidance — but review is **advisory, not
  blocking**. Concerns are raised once; Pedro decides and moves on. No gatekeeping given the
  time budget.
- **Learning-first tool choices.** Where a choice exists between two similarly-viable tools,
  prefer the one that builds a new or complementary skill over the one that's merely fastest,
  as long as it doesn't put the live-deployment deadline at risk.

## 3. Dataset & Model

- **Dataset:** Kaggle "Credit Card Fraud Detection" — ~285k anonymized/PCA-transformed
  transactions, ~0.17% fraud rate. Chosen for genuine class imbalance (forces real handling,
  not just a claim of it), small size (fast to train), and zero setup friction (Kaggle CLI
  already configured on Pedro's machine).
- **Model:** XGBoost, trained with class-weighting (or comparable imbalance handling) and
  evaluated on **PR-AUC**, not accuracy, given the imbalance.
- **Explainability:** SHAP values computed per prediction, surfaced through the API.

## 4. LLM Layer

- **Provider: Hugging Face Inference API**, via `huggingface_hub`'s `InferenceClient` —
  hosted inference, no local model weights loaded in the deployed service.
  - Rejected: self-hosting a generative model locally via `transformers` inside the deployed
    container. Render's free tier (~512MB RAM, shared CPU, no GPU) cannot reliably serve even
    a small (1-3B parameter) instruction-tuned model; a live demo that times out is worse than
    no live demo.
  - Rejected: Anthropic/OpenAI APIs — viable alternatives, but Hugging Face was preferred
    specifically because it builds a *different* HF skill (hosted generative inference) than
    Pedro's existing FinBERT project (local encoder-only classification), rather than
    introducing a third, unrelated provider.
- **Scope of use:** one function, `shap_values -> HF Inference API call -> plain-English
  explanation string`. This is the only place an LLM is used in the project — the model,
  training pipeline, and API logic are entirely Pedro's own code. This directly demonstrates
  "leverage LLMs without depending on them."
- **Open decision (deferred to implementation):** whether the explanation is returned inline
  in `/predict`'s response, or split into a separate `/explain` endpoint. To be decided when
  that ticket is picked up — either is compatible with this design.
- Requires an HF account/API token (free tier). To be set up as part of the relevant ticket.

## 5. API & Service

- **Framework:** FastAPI.
- **Endpoints:**
  - `GET /health` — returns `200 {"status": "ok"}`. Touches nothing else; exists purely to
    prove the deployed service is reachable. This is the first thing built (see Section 6).
  - `POST /predict` — accepts transaction features, returns fraud probability, a
    fraud/not-fraud classification, and (per Section 4) a plain-English explanation.
- **Containerization:** Docker.
- **Deployment target:** Render, free Docker-based Web Service, deployed from GitHub so a
  push to `main` can auto-redeploy. **Fallback: Fly.io**, if Render's free-tier terms have
  changed by the time this is implemented — worth a quick check at that point.

## 6. Build Sequence — "Walking Skeleton First"

Three sequencing options were considered:

1. **Walking skeleton first (chosen).** Build a minimal, fully-deployed, CI-passing skeleton
   on day one (`/health`, stub `/predict`, Dockerized, deployed live, GitHub Actions green),
   then fill in real functionality behind it.
2. Layer by layer (model → LLM → API → Docker → CI → deploy) — rejected because deployment
   and CI/CD, the two things this project most needs to prove, would land last and be what
   gets cut under time pressure.
3. Two parallel tracks (infra vs. ML, merged near the end) — rejected as slower for a solo
   builder context-switching between two unfinished halves than sequential work.

**Rationale for (1):** it guarantees the CI/CD and deployment goals exist and are proven
working early, rather than being the rushed last step of the week. It also mirrors how a
real team would sequence this work.

**Indicative milestone sequence** (exact tickets/acceptance criteria are written as GitHub
Issues at implementation time, one at a time, per Section 8):

1. Walking skeleton: stub FastAPI app, Dockerized, deployed live to Render, GitHub Actions
   running lint/tests on push.
2. Data acquisition + EDA (in `notebooks/`).
3. Model training: XGBoost + imbalance handling + PR-AUC evaluation; SHAP integration.
4. LLM explanation layer (Hugging Face Inference API).
5. Wire the real model + explanation layer into `/predict`; redeploy.
6. CI/CD hardening: complete test coverage for the API layer, confirm the pipeline runs
   tests + build on every push.
7. README + polish: architecture description, live demo link, how to run locally.

## 7. Testing Strategy

Split by layer rather than applied uniformly:

- **API/engineering code** (`/health`, `/predict`, request validation, the HF explanation
  wrapper function): **TDD** — write the test first, since expected behavior is known
  upfront and doing so clarifies the interface before implementation.
- **ML/model code** (training, imbalance handling, SHAP): **explore first, test after** —
  correct behavior isn't knowable until the data has been explored (in `notebooks/`); tests
  are written afterward as regression checks once expected behavior is established (e.g.
  "PR-AUC stays above threshold X on the holdout set," "SHAP output has one value per
  feature").
- `pytest` runs in CI either way; the split is about *when* the test is written, not whether
  one exists.

## 8. Repo Scaffold & Workflow

```
credit-risk-fraud-detector/
├── CLAUDE.md                  # supervisor rules: ticket format, advisory-not-blocking
│                               #  review, workflow reminders (authored during implementation)
├── README.md                  # public-facing overview, live demo link, how to run
├── docs/
│   └── spec/
│       └── design.md          # this document
├── app/                        # FastAPI service
│   ├── main.py                  # /health, /predict
│   ├── model.py                  # load model + SHAP
│   └── explain.py                 # HF Inference API call -> explanation string
├── training/
│   └── train_model.py          # data load, imbalance handling, train + save model
├── notebooks/                  # EDA and model prototyping only — not production code
├── tests/
├── Dockerfile
├── requirements.txt
├── .github/workflows/ci.yml    # lint + tests + build
├── data/, models/               # gitignored — Kaggle data and trained artifact
└── .gitignore
```

**Workflow:**
- New public GitHub repo, separate from `ai_job_search`.
- Tickets are real **GitHub Issues** (title, description, acceptance criteria), written by
  Claude one at a time as supervisor, before each is picked up.
- Per ticket: Pedro branches from `main` → implements → opens a PR referencing the issue
  (`Closes #N`) → Claude reviews the diff (advisory comments, not a blocking gate) → Pedro
  merges.
- No formal sprint board — a one-person, one-week project doesn't need one. The milestone
  sequence in Section 6 is the plan.

## 9. Definition of Done

- `main` branch has green CI (lint + tests + build passing).
- Service is live and reachable at a public Render URL.
- `README.md` documents the architecture, the live demo link, and how to run the project
  locally.
- All planned issues (Section 6's milestones, broken into GitHub Issues at implementation
  time) are closed.

## 10. Explicitly Out of Scope

- Model performance tuning beyond "reasonable and honestly reported" — this is not a
  Kaggle-leaderboard exercise.
- A formal sprint board / project management tooling beyond the GitHub Issues themselves.
- Any LLM usage beyond the single, bounded explanation-generation call described in Section 4.
- Local generative-model inference via `transformers` in the deployed path (may still happen
  as a separate, non-deployed learning script — not part of this spec's deliverable).
