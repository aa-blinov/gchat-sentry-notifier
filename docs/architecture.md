# Architecture

How the Sentry to Google Chat notifier is put together.

## The shape

The service is a single module. `app/main.py` holds the FastAPI application,
the two routes, the environment filter and the message transform, and there
is no second module to import.

| Module | What it is responsible for |
| --- | --- |
| `app/main.py` | The app object, the environment filter, the message transform, the two routes, the Sentry SDK setup. |
| `tests/test_stateless.py` | The five tests. The HTTP client is patched, so nothing in the suite reaches the network. |

Around the module sit the things that are not code: `requirements.txt` with
six unpinned runtime dependencies, `requirements-test.txt` with pytest,
pytest-asyncio and pytest-mock, `pytest.ini` with `asyncio_mode = strict`,
and `ruff.toml` with a line length of 120 and the E, F, W, B and C rule
sets.

Two Dockerfiles describe two images. `Dockerfile` installs the runtime
dependencies on `python:3.10.17-slim` and copies no application code, so
the image holds `requirements.txt` and nothing else, and it keeps the base
image's own `CMD ["python3"]`. `Dockerfile.test` builds the same base,
installs both requirement files, copies `app/` and `tests/` into `/app`, and
sets a command that points at `/tests`, a directory the image does not have.
`docker-compose.yml` defines the two services and replaces the test
command with `pytest -v tests` from a mounted working tree, which is why
`docker compose run --rm tests` works and the test image on its own does
not.

## The data flow

```
Sentry -> POST /sentry-webhook -> JSON body -> environment filter -> message text -> httpx POST -> Google Chat
```

The route reads the body with `await request.json()` and hands the parsed
object to the transform. The transform reads `event.environment`, lowercases
it and strips it, and compares it with the list in
`ALLOWED_ENVIRONMENTS`. A payload whose environment is not on the list ends
the flow there: the route answers with a message and no request is made to
Google Chat.

A payload that passes the filter is turned into one object with one key,
`text`, holding eight lines joined by `\n`. The route then posts that object
to `GOOGLE_CHAT_WEBHOOK_URL` with `httpx.AsyncClient` and a
`Content-Type: application/json; charset=UTF-8` header. A status of 200
from Google Chat produces the success message; any other status produces the
error message, with the body Google Chat replied with inside it.

A separate path goes the other way. `init_sentry()` runs at import time and
configures the Sentry SDK with `SENTRY_DSN`, `traces_sample_rate=1.0` and
`profiles_sample_rate=1.0`, so the service's own errors and traces go back
to Sentry.

## Decisions worth knowing about

### The configuration is read once, at import time

The three variables are read with `getenv` in the module body, and the
module raises `ValueError` when `GOOGLE_CHAT_WEBHOOK_URL` or `SENTRY_DSN`
is missing. A service that is missing its configuration stops while it is
starting, where the error can be read, rather than failing on the first
alert of the day.

The cost is that the configuration cannot change without a restart, and that
anything importing the module needs the two variables in the environment
first. The test module works around this by patching `sentry_sdk.init`
before the import, and the autouse fixture that sets the environment does
nothing, because the import happens during collection and before any
fixture runs.

### The payload's environment is normalised, the list is not

The environment is lowercased and stripped before the comparison.
`ALLOWED_ENVIRONMENTS` is only split on commas. So
`ALLOWED_ENVIRONMENTS=Production` matches nothing, and so does
`production, prod` with a space after the comma. The list is written by
hand and read by hand, and the two halves do not agree on what a space is.

### A filtered environment is a 200

A payload from an environment that is not on the list is answered with
`{"message": "Environment not allowed. Skipping notification."}` and a 200
status code. Sentry retries a non-2xx answer, and a retry of an alert the
owner did not want is noise of a worse kind than the original skip.

### The message is one text field with eight lines

The transform builds a single string rather than a card, a title and
fields. Google Chat renders a `text` field as the message text and renders
anything else as something else, and the second one is a different shape in
a chat space. Eight lines, in one order, every time, is a thing a reader
learns.

The cost is that a field with no value prints `None`, because the transform
uses `dict.get` without a default for everything except the platform, which
falls back to `unknown`.

### The forward passes no timeout

The `httpx.AsyncClient` is created without a timeout argument, so httpx's
default of 5 seconds applies. The client is also created per request, so
each webhook opens a new connection and closes it again.

### The message is posted once, and a failure is not a failure status

There is no retry, no queue and no deduplication: one webhook in, one
message out. When Google Chat answers with a status other than 200 the
route still returns 200, with an `error` key in the body and a log line
beside it. Sentry sees a successful webhook and the operator sees the error
in the log.

## Boundaries

- **`app/` imports nothing from the repository.** The module stands alone,
  and a second module under `app/` has to earn its place before it is
  added. The filter and the transform are the two pieces most likely to be
  pulled out, and both are currently under twenty lines.
- **`app/main.py` does not import `tests/`.** The dependency runs the other
  way: the test module imports the application.
- **The environment is set before `app.main` is imported, always.** Any
  new entry point, script or test that imports the module needs
  `GOOGLE_CHAT_WEBHOOK_URL` and `SENTRY_DSN` in the environment first.
- **Nothing else reaches Google Chat.** The webhook URL is read in one
  place and used in one place, and there is no second client in the
  repository.
