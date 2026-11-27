# 💳 PayLink: Digital Wallet & Bill-Payment Backend

[![Python](https://img.shields.io/badge/Python-3.11+-blue.svg)](https://www.python.org/)
[![FastAPI](https://img.shields.io/badge/FastAPI-0.110+-009688.svg)](https://fastapi.tiangolo.com/)
[![PostgreSQL](https://img.shields.io/badge/PostgreSQL-16-336791.svg)](https://www.postgresql.org/)
[![Docker](https://img.shields.io/badge/Docker-Compose-2496ED.svg)](https://www.docker.com/)
[![CI Status](https://img.shields.io/badge/CI-GitHub%20Actions-2088FF.svg)](#)

**PayLink** is a resilient, auditable digital wallet and utility bill-payment backend engine built for the Egyptian market (comparable to Fawry or Vodafone Cash). It enables customers and retail agents to pay electricity, water, mobile top-up, and internet bills safely and reliably.

---

## ✨ Key Architectural Highlights

* **Double-Entry Immutable Ledger:** Wallet balances (`AVAILABLE` and `HELD`) are never stored as mutable columns—they are dynamically derived from append-only ledger entries using exact `DECIMAL` arithmetic (`ROUND_HALF_UP`).
* **Strict Payment State Machine:** Enforces a deterministic lifecycle (`Initiated` → `FundsHeld` → `SentToBiller` → `Completed` / `Failed` / `Unknown` → `Refunded`) preventing invalid transitions or premature refunds.
* **Idempotency & Concurrency Safety:** Guarantees zero double charges and zero negative balances under simultaneous requests via per-customer idempotency keys and database-level locking.
* **Plug-and-Play Biller Adapters:** Uses the **Adapter** and **Factory** patterns to integrate diverse biller protocols without modifying core business logic:
  * **ElectroCo** (Synchronous REST JSON)
  * **AquaCo** (SOAP-style XML over HTTP)
  * **TopUpNet** (Asynchronous REST JSON with webhook callbacks)
* **Resilience & Reconciliation:** Automatic background re-queries (`+1m`, `+5m`, `+15m`) for timed-out (`Unknown`) payments, independent notification retries, and daily settlement file reconciliation.

---

## 🛠️ Tech Stack

* **Backend Framework:** Python, FastAPI
* **Database:** PostgreSQL
* **Containerization:** Docker & Docker Compose
* **Quality & CI:** Pytest, Ruff (Linter), GitHub Actions

---

## 🚀 Quick Start (Docker)

### 1. Build and Run the Stack
Start the FastAPI backend and PostgreSQL database with a single command:

```bash
docker compose up --build
```

### 2. Verify Service Health
Once the containers are healthy, test the `/health` endpoint:

```bash
curl http://localhost:8000/health
```

**Expected Response:**

```json
{"status": "ok"}
```

---

## 💻 Local Development & Testing

To run linting and automated tests locally without Docker:

```bash
# Install development dependencies
pip install -r requirements-dev.txt

# Run code linter
ruff check .

# Run test suite
pytest
```

---

## 📂 Repository Structure

```text
paylink-backend/
├── app/
│   └── main.py                 # FastAPI entry point & health check
├── tests/                      # Automated unit & integration tests
├── docs/
│   ├── design/                 # System design diagrams (Mermaid .mmd files)
│   │   ├── use-case.mmd
│   │   ├── class-diagram.mmd
│   │   ├── state-diagram.mmd
│   │   ├── er-diagram.mmd
│   │   ├── seq-payment.mmd
│   │   └── seq-refund.mmd
│   └── prompt-log.md           # Phase 1 AI engineering & review log
├── .github/workflows/
│   └── ci.yml                  # Automated CI pipeline (Lint, Test, Docker Build)
├── Dockerfile
├── docker-compose.yml
└── README.md
```

---

## 👥 Team Members (Advanced Software Engineering)

| Name | Role |
| :--- | :--- |
| **Alaa Ayman Ibrahim Elsaidy** | Requirements, Architecture Design & DevOps Lead |
| **Omar Ahmad Abdelaziz Ahmad** | Quality Assurance & Review Lead |
