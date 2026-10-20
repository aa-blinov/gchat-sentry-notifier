# Product

<!-- impeccable:product-schema 1 -->

## Platform

web

The service is a FastAPI application on Python 3.10, pinned to
`python:3.10.17-slim` in both Dockerfiles. It runs as a container or as a
uvicorn process, and it has no interface of its own: it answers a handful of
HTTP routes and writes log lines.

## Users

The owner of the service, and the team whose Google Chat space receives the
alerts. The situation: an error arrives in Sentry, and the person who has to
act on it is in a chat space rather than in Sentry. The service puts the
error where the conversation already is.

Nobody else is confirmed. The repository is public, with one star, one fork,
no repository description and one open pull request. There is no account
system and no onboarding, and no record of who runs it.

## Product Purpose

The service does one thing: it forwards a Sentry error to a Google Chat
space, and only for the environments the owner named.

Success for the user: the error is in the chat, in the same shape every time,
with the project, the level and the culprit in it, and the noise from the
development environment never arrives.

Success for the product: every webhook ends in an answer that says what
happened, including the times when nothing was sent.

## Positioning

Sentry can post to a webhook, and what arrives at a webhook is a JSON
document, while a chat space wants a sentence. The part a plain relay does
not have is the filter: the service keeps the environments on a list and
answers the rest with "Environment not allowed. Skipping notification.",
instead of forwarding noise into the channel.

The second part is the message's shape: one `text` field, eight lines, the
same eight lines every time, so an alert reads the same in every space it
lands in.

The README names no competing product, and the code compares itself with
none, so the comparison stops here.

## Operating Context

The service runs wherever the owner runs it: a container on a small machine,
or a uvicorn process. Three environment variables come from the process
environment: the Google Chat incoming webhook, the Sentry DSN for the
service's own errors, and the list of environments to forward.

The loop that uses it: the Sentry project's settings, Integrations,
Webhooks, and the URL of this service; the service posts to the incoming
webhook of a Google Chat space; the message appears in the space. The space
has to exist first, because the incoming webhook is created there and its
URL is the only thing the service needs from it.

The person who configures the service is the person who operates it. There
is no dashboard, no settings page and no reconfiguration without a restart:
a change to the environment list needs the process to start again.

## Capabilities and Constraints

Confirmed by the code:

- `POST /sentry-webhook` takes a JSON payload and answers with JSON.
- The payload's `event.environment` is compared against
  `ALLOWED_ENVIRONMENTS`, which defaults to `production,prod`.
- The message is one `text` field with the ID, project, environment, level,
  culprit, message, platform and URL, each in Google Chat's `*bold*` markup.
- The forward is an `httpx` POST with `Content-Type: application/json;
  charset=UTF-8`.
- An answer other than 200 comes back as `{"error": "Failed to send message
  to Google Chat: ..."}`, with a 200 status code.
- `GET` and `HEAD` on `/health-check` answer 204 with no body.
- The service serves FastAPI's documentation routes: `/docs`, `/redoc` and
  `/openapi.json`.
- The service sends its own errors to Sentry, with traces and profiles sampled
  at 1.0.
- The application refuses to start when `GOOGLE_CHAT_WEBHOOK_URL` or
  `SENTRY_DSN` is missing.
- The variables come from the process environment. Nothing in the code reads
  a file.

Deliberately not true, and worth knowing before a change:

- The endpoint has no authentication. Anyone who knows the URL can post a
  payload to it, and the service will forward it to the chat space.
- A call to Google Chat that fails ends as an UnboundLocalError and a 500,
  not as the error message that was logged.
- The service image carries the dependencies and no application code, so a
  container from it starts Python and exits at once.
- There is no retry, no queue and no deduplication. One webhook in, one
  message out.
- The default branch has no CI workflow.

## Brand Commitments

**Name:** the repository is `gchat-sentry-notifier`, and the README calls the
product the Sentry to Google Chat webhook notifier. **Tone:** plain and
technical, second person, and no adjective that cannot be counted.

The service has no visual identity of its own: no logo, no colours, no
screenshots. The only text it produces is the Google Chat message and the
log lines, and both are English.

The message format is fixed: eight lines in the same order, each label in
`*bold*`, every time. An alert that arrives in a space the reader has seen
before needs no explanation of its shape.

## Evidence on Hand

- `app/main.py` is the whole implementation: the routes, the filter, the
  message format, the defaults.
- `tests/test_stateless.py` covers the route, the filter, the transform and
  the health check, with the HTTP client mocked.
- `.env.test` holds the three variables with placeholder values.
- `docker-compose.yml` holds the port mapping and the two services.
- The GitHub page shows one star, one fork and one open pull request, and no
  repository description.
- The licence is not in the repository yet. The open pull request that
  adopts the house standard adds an MIT licence, © 2025-2026 Alexander
  Blinov.

Absent, and not to be invented: no usage numbers, no volume of alerts, no
testimonials, no screenshots, no comparison with other alerting tools, and no
record of who runs the service.

## Product Principles

1. **A filtered environment is a normal answer.** The service answers
   Sentry with 200 and a message that says the environment was skipped. A
   non-2xx answer makes Sentry retry, and a retry of an alert nobody wanted
   is noise of a worse kind.
2. **Say what happened, every time.** Every request ends in a JSON body with
   either `message` or `error` in it, and a failed forward is written to the
   log.
3. **Fail at startup, not at the first request.** A missing webhook URL or
   DSN stops the process while it is starting, where the error can be read,
   rather than failing on the first alert of the day.
4. **One text field, eight lines, the same order.** The message is written
   for a chat space rather than for a dashboard, and it does not grow a
   field without a reason.
5. **The configuration is three environment variables.** Anything beyond that
   needs a reason worth the cost of a restart.

## Accessibility & Inclusion

- **The product has no interface of its own.** It answers HTTP and writes log
  lines, and nothing in the code depends on colour, layout or an image.
- **The notification is one text field.** It carries no markup beyond Google
  Chat's `*bold*`, so it needs no layout to be understood.
- **The language has not been decided.** The labels in the message are
  English, and the code writes them in English. Nothing in the repository
  says whether that was a choice.
- **The shape of the message on a phone screen has not been looked at.**
  Eight lines is a decision that was made, and no one has checked how it
  reads in a notification on a small screen.
