# Sentry to Google Chat webhook notifier

Point Sentry at this service and an error arrives in a Google Chat space as a
message: the project, the environment, the level and the culprit. The service
takes the Sentry payload, keeps the ones from the environments you named, and
forwards the rest to a Google Chat incoming webhook.

[![License](https://img.shields.io/badge/license-MIT-green)](LICENSE)

---

## Quick start

Python 3.10, and the six dependencies in `requirements.txt`.

```bash
pip install -r requirements.txt
export GOOGLE_CHAT_WEBHOOK_URL="https://chat.googleapis.com/v1/spaces/SPACE/messages?key=KEY&token=TOKEN"
export SENTRY_DSN="https://KEY@sentry.io/12345"
uvicorn app.main:app --port 8000
```

The first two variables are required. `app/main.py` reads them when it is
imported and raises `ValueError` when either one is missing, so the service
stops at startup rather than at the first alert of the day.

Check that it is up:

```bash
curl -i http://localhost:8000/health-check
```

The answer is `204 No Content` with an empty body.

The tests run in a container, and take their variables from `.env.test`:

```bash
docker compose run --rm tests
```

The compose file has a `webhook-service` as well. Its image holds the
dependencies and no application code, so the container starts Python, reads
nothing and exits. Run the service with uvicorn.

## What it does

- **Takes the Sentry webhook.** `POST /sentry-webhook` reads the JSON body
  Sentry sends and answers `200` with a JSON message saying what it did with
  it.
- **Keeps the environments you named.** A payload whose `event.environment` is
  on the list is forwarded. Anything else is answered with "Environment not
  allowed. Skipping notification." and never reaches Google Chat.
- **Formats the message for a chat space.** The payload becomes one `text`
  field with the ID, project, environment, level, culprit, message, platform
  and URL, each wrapped in the `*bold*` markup Google Chat renders.
- **Posts to an incoming webhook.** The message goes to
  `GOOGLE_CHAT_WEBHOOK_URL` as `application/json; charset=UTF-8`. An answer
  other than `200` comes back as an `error` key holding the text Google Chat
  replied with.
- **Answers a health check.** `GET` and `HEAD` on `/health-check` return `204`
  with no body.
- **Reports its own failures to Sentry.** The service initialises the Sentry
  SDK with its own DSN, with traces and profiles sampled at `1.0`.

## How it works

```
Sentry -> POST /sentry-webhook -> environment filter -> message text -> httpx POST -> Google Chat
```

The service is one module, `app/main.py`, 107 lines. It reads its three
environment variables at import time, initialises the Sentry SDK, and
registers two routes.

The filter is a list membership test. The payload's `event.environment` is
lowercased and stripped, and the result has to be in `ALLOWED_ENVIRONMENTS`,
which is the comma-separated value split on commas. A payload with no `event`
key gives an empty environment, which is on no list, and is skipped.

The message is built with `dict.get`, so a payload that lacks a field shows
`None` in the text. The platform is the one field with a fallback, `unknown`.

The route answers with one of three bodies:

```json
{"message": "Webhook received and forwarded to Google Chat successfully"}
{"message": "Environment not allowed. Skipping notification."}
{"error": "Failed to send message to Google Chat: ..."}
```

The forward uses `httpx.AsyncClient` without a timeout, so the client's
default of 5 seconds applies.

`Dockerfile` installs the dependencies on `python:3.10.17-slim` and stops
there. It copies no application code, so `/app` in the image holds
`requirements.txt` and nothing else, and it sets no command of its own, so
the image keeps the base image's `CMD ["python3"]`. A container from it
starts Python, reads nothing and exits; `docker compose up webhook-service`
comes up and is straight back to `Exited (0)`. `Dockerfile.test` builds the
same base, installs both requirement files, copies `app/` and `tests/`, and
runs `pytest -v /tests`. `docker-compose.yml` defines two services:
`webhook-service`, which publishes host port 8007 on container port 8000, and
`tests`, which mounts the working tree at `/workspace` and runs
`pytest -v tests`. Both take their variables from `.env.test`.

## Configuration

| Variable | Default | What it does |
| --- | --- | --- |
| `GOOGLE_CHAT_WEBHOOK_URL` | none | Where the message is posted. The service refuses to start without it. |
| `SENTRY_DSN` | none | Where the service sends its own errors. The service refuses to start without it. |
| `ALLOWED_ENVIRONMENTS` | `production,prod` | Comma-separated environments that are forwarded. Every other environment is skipped. |

The port is the one uvicorn is started with, 8000 in the command above. The
compose file publishes 8007 on the host for the container's 8000.

`.env.test` holds the three variables with placeholder values, and Docker
Compose reads it for both services. Nothing in the application reads a file:
the variables come from the process environment.

## Development

```bash
pip install -r requirements.txt          # the service
pip install -r requirements-test.txt     # pytest, pytest-asyncio, pytest-mock
pytest -v tests/                        # the five tests
docker compose run --rm tests           # the same tests, in the test image
ruff check .                            # the linter; ruff is not in requirements-test.txt
```

`pytest` needs `GOOGLE_CHAT_WEBHOOK_URL` and `SENTRY_DSN` in the environment,
because the test module imports the application and the application refuses
to import without them. The compose file supplies `.env.test`; a local run
needs the two variables exported.

The five tests in `tests/test_stateless.py` cover the route, the filter, the
transform and the health check. None of them reaches the network: the HTTP
client is replaced with a mock.

| Path | What it holds |
| --- | --- |
| `app/main.py` | The whole service: the app, the two routes, the filter, the message format. |
| `tests/test_stateless.py` | The five tests. |
| `requirements.txt` | The six runtime dependencies, unpinned. |
| `requirements-test.txt` | pytest, pytest-asyncio, pytest-mock. |
| `pytest.ini` | `asyncio_mode = strict`, and a function-scoped fixture loop. |
| `ruff.toml` | Line length 120, and the E, F, W, B and C rule sets. |
| `Dockerfile` | The service image. |
| `Dockerfile.test` | The test image, with `pytest -v /tests` as its command. |
| `docker-compose.yml` | The `webhook-service` and `tests` services. |
| `.env.test` | The three variables, with placeholder values. |

## License

MIT, © Alexander Blinov. See [LICENSE](LICENSE).
