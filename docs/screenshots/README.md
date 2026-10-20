# Screenshots

There are no screenshots in this repository, and there is no interface to
take them of.

The service draws nothing. It answers two HTTP routes, forwards a text field
to a Google Chat incoming webhook, and writes log lines. A screenshot would
have to be of Google Chat, taken by the owner of the space rather than by
this project, and it would show Google Chat's own rendering of a message
rather than anything the code decides.

What the code does decide is worth writing down instead, because it is what
a reader of the chat space sees.

| What | Where it comes from |
| --- | --- |
| The message | The `text` field built in `transform_sentry_webhook_to_google_chat` in `app/main.py`: eight lines, joined by `\n`, each label wrapped in `*bold*`. |
| The three answers | The bodies returned by the `POST /sentry-webhook` route: the success message, the skipped message and the error message. |
| The health check | `GET` and `HEAD` on `/health-check`, which answer `204` with an empty body. |

To see the message for yourself, run the service and post a payload to it:

```bash
uvicorn app.main:app --port 8000

curl -s http://localhost:8000/sentry-webhook \
  -H 'Content-Type: application/json' \
  -d '{"id":"12345","project_name":"demo","level":"error","culprit":"app.views",
       "message":"Test error","url":"http://example.com",
       "event":{"environment":"production","platform":"python"}}'
```

The command needs `GOOGLE_CHAT_WEBHOOK_URL` and `SENTRY_DSN` in the
environment, because the module refuses to import without them. It also
posts a real message into whatever space the webhook URL points at, so
point it at a test space first.

If images are added later, they go in this folder and are referenced from
the README by relative path, the way every other repository in the
collection does it.
