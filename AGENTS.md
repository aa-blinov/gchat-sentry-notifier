# AGENTS.md — agent context for `gchat-sentry-notifier`

One FastAPI service that turns a Sentry webhook into a Google Chat message.
The whole implementation is `app/main.py`, and the whole configuration is
three environment variables.

This file is what you need to work on the project without reading all of it.

## Run / build / test

```bash
pip install -r requirements.txt          # the service
pip install -r requirements-test.txt     # pytest, pytest-asyncio, pytest-mock
export GOOGLE_CHAT_WEBHOOK_URL=... SENTRY_DSN=...   # the application will not import without these two
pytest -v tests/                        # the five tests, locally
docker compose run --rm tests           # the same tests, in the test image
uvicorn app.main:app --port 8000        # run the service
```

There is no build step. The two Dockerfiles pin `python:3.10.17-slim`, and
`ruff.toml` holds the linter's rules, but ruff is not in either requirement
file, so install it yourself before running `ruff check .`.

## Where things live

| Path | What it holds |
| --- | --- |
| `app/main.py` | The whole service: the FastAPI app, the two routes, the environment filter, the message format, the Sentry SDK setup. 107 lines. |
| `tests/test_stateless.py` | The five tests. The HTTP client is a mock, so nothing in the suite reaches the network. |
| `requirements.txt` | The six runtime dependencies: fastapi, uvicorn[standard], httpx, loguru, sentry-sdk[fastapi], python-dotenv. None of them is pinned. |
| `requirements-test.txt` | pytest, pytest-asyncio, pytest-mock. |
| `pytest.ini` | `asyncio_mode = strict` and `asyncio_default_fixture_loop_scope = function`. |
| `ruff.toml` | Line length 120, and the E, F, W, B and C rule sets. |
| `Dockerfile` | The service image: the base, the runtime dependencies, and no application code. |
| `Dockerfile.test` | The test image: the same base, both requirement files, `app/` and `tests/` in `/app`, and a `CMD` that points at `/tests`. |
| `docker-compose.yml` | The `webhook-service` and `tests` services. |
| `.env.test` | The three variables with placeholder values, read by Compose for both services. |
| `.dockerignore` | `.env`, `*.pyc`, `__pycache__`, `.DS_Store`, `.venv`, `venv`. |

## Traps

- **The module refuses to import without two variables.** `app/main.py`
  raises `ValueError` at import time when `GOOGLE_CHAT_WEBHOOK_URL` or
  `SENTRY_DSN` is missing. `pytest` then fails during collection, and
  uvicorn exits with a traceback. The message names `service.env`, a file
  this repository does not have; the file Compose reads is `.env.test`.
- **The payload's environment is normalised, the list is not.** The
  environment is lowercased and stripped before the comparison, but
  `ALLOWED_ENVIRONMENTS` is only split on commas. `ALLOWED_ENVIRONMENTS=Production`
  never matches anything, and neither does `production, prod` with the space
  after the comma.
- **A failed call to Google Chat ends as an UnboundLocalError.** The
  `except httpx.RequestError` branch logs the error and then falls through
  to `response.status_code`, and `response` was never assigned. The client
  gets httpx's default 5-second timeout, because no timeout is passed, so a
  slow Google Chat produces a 500 with an UnboundLocalError in it, not the
  message that was logged.
- **A refused message is still HTTP 200.** When Google Chat answers with any
  status other than 200, the route returns `{"error": ...}` with a 200 status
  code. Sentry sees a success, and the error is only in the body and in the
  log.
- **The service image holds the dependencies and no application code.**
  `Dockerfile` copies `requirements.txt` and stops, so `/app` in the image
  has no `app/` directory. There is no `CMD` either, so the image keeps the
  base image's `CMD ["python3"]`: a container from it starts Python, reads
  nothing from its input and exits with status 0, and `docker compose up
  webhook-service` comes up and is straight back to `Exited (0)`. The compose
  service is there for the port mapping and the environment file; the
  service itself runs under uvicorn.
- **The fixture that sets the environment changes nothing.**
  `mock_environment` is autouse and calls `monkeypatch.setenv`, but
  `app/main.py` reads the environment when it is imported, and the import
  happens while the test module is collected, before any fixture runs. The
  values in force during the tests are the ones in the shell that started
  pytest.
- **Every trace and every profile goes to Sentry.** `traces_sample_rate=1.0`
  and `profiles_sample_rate=1.0` send 100% of the service's own telemetry to
  the DSN in `SENTRY_DSN`. Lower the two rates before pointing this at a
  busy project.
- **The test image's own command points at a directory that is not in it.**
  `Dockerfile.test` copies `tests/` into `/app/tests` and sets `CMD
  ["pytest", "-v", "/tests"]`, so the image exits with `file or directory not
  found: /tests`. The compose service replaces the command with `pytest -v
  tests`, mounts the tree at `/workspace` and runs from there, which is why
  `docker compose run --rm tests` works and the image on its own does not.

## Style / safety rails

- **The configuration is read at import time, and only at import time.** A
  new setting is a new `getenv` line at the top of `app/main.py`, next to the
  other three, and the module has to decide what happens when it is missing.
- **A filtered environment is an answer, not an error.** The route returns
  200 with "Environment not allowed. Skipping notification." Sentry retries a
  non-2xx answer, so a webhook that was filtered out has to stay a 200.
- **Every request ends in a message body.** The route always returns JSON
  with either `message` or `error` in it, including when the forward to
  Google Chat failed.
- **Do not add a dependency without a reason next to it.** `requirements.txt`
  holds six entries and no pins. `python-dotenv` is in the list, and nothing
  in the code imports it.
- **Comments explain why, never what.**

## Where the product truth lives

`PRODUCT.md` holds it. Read it before changing what the service sends, and
write the change there when the message changes.
