# Documentation

The documents for the Sentry to Google Chat notifier. The service is one
module of 107 lines, so the documentation is short. What it does cover is
where the edges are: the filter, the message format, and the three answers
the route can give.

| Document | What it covers |
| --- | --- |
| [architecture.md](architecture.md) | The one module, the two routes, the filter and the message. |
| [screenshots/README.md](screenshots/README.md) | Why there are no images, and what stands in for them. |
| [../README.md](../README.md) | The front door: what it does and how to run it. |
| [../AGENTS.md](../AGENTS.md) | The commands, the paths, and the traps. |
| [../PRODUCT.md](../PRODUCT.md) | What the service is for, and why the message looks the way it does. |
| [../DESIGN.md](../DESIGN.md) | The one surface the service renders, and the fact that it has no interface of its own. |

## Reading order

New here: [../README.md](../README.md), then [architecture.md](architecture.md).

Changing the code: [architecture.md](architecture.md), then
[../AGENTS.md](../AGENTS.md) for the traps.

Changing what the service sends: [../PRODUCT.md](../PRODUCT.md) first. It
holds the message format and the reason the format has the shape it has.

## What is not here

There is no guide to deploying the service. The README says how it runs
under uvicorn, and the two Dockerfiles hold the image. There is no API
reference either: FastAPI serves one at `/docs` from the running service,
and FastAPI produces it, so a copy in this folder would go out of
date.
