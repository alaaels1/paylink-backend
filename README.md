# PayLink

Run: `docker compose up --build`
Check: `curl http://localhost:8000/health`  ->  `{"status":"ok"}`

Dev: `pip install -r requirements-dev.txt && ruff check . && pytest`
