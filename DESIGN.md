---
name: gchat-sentry-notifier
description: The service has no visual identity of its own; the only surface it renders is one Google Chat message.
colors:
  # No palette of its own. Google Chat paints the message.
  primary: null
  neutral-bg: null
typography:
  body:
    # No typeface of its own. Google Chat picks the font.
    fontFamily: null
    fontSize: null
    fontWeight: null
    lineHeight: null
  title:
    fontFamily: null
    fontSize: null
    fontWeight: null
    lineHeight: null
    letterSpacing: null
rounded:
  md: null
spacing:
  sm: null
  md: null
---

# The design record of gchat-sentry-notifier

This service has no interface. It answers two HTTP routes, forwards a
message to a Google Chat incoming webhook, and writes log lines. Nothing in
the repository draws a pixel, and no colour, typeface, radius or spacing
value appears anywhere in the code.

The frontmatter above is therefore empty on purpose. Every token is `null`,
because the service holds no token to record. A filled-in palette here would
be a fiction: it would describe a screen the project does not have.

What the project does own is one message. Google Chat renders that message,
and the repository decides what is in it, in what order, and which parts are
bold. That is the entire design surface, and it is recorded below.

## Overview

**Creative North Star: "the plain text alarm"**

The product is a relay, and a relay that decorates its output is a relay
that has to be redesigned every time the receiver changes. The service sends
one text field with eight lines, the same eight lines every time, each label
wrapped in Google Chat's `*bold*` markup. The reader in the chat space sees
an alert that looks like every other alert they have read there.

The confirmed rejections are worth recording, because the code shows them
being made:

- No logo, no mark, no colour.
- No card, no title, no severity colour. The level is a word on a line.
- No link with a friendly label. The URL is a line of its own.
- No grouping, no sections, no emoji in the message.

The key characteristics are: one text field, eight lines, a fixed order, a
bold label on every line, and nothing else.

## Colors

The service names no colour. There is no CSS, no theme, no colour constant
in the code, and the one consumer of the output, Google Chat, applies its own
palette to whatever arrives.

- **Google Chat's own surface**: the colours of the space the message lands
  in. The service has no say in them, and the repository records none.

The one thing the code decides about the appearance of the message is which
parts of it are bold, and bold is a property of the markup rather than of a
colour.

### Named rules

**The message is text or it is nothing.** A line that arrives without a
label cannot be read, so every line carries one. A line that arrives with
two labels has stopped being a line.

## Typography

**Display Font:** none. **Body Font:** none.

The font is the receiver's. What the repository sets is the emphasis: the
label at the start of each line is wrapped in `*`, which covers the whole of the
markup the service uses.

The hierarchy is therefore flat and it is deliberate:

| Level | What it is | Where it appears |
| --- | --- | --- |
| Label | The field name, in `*bold*` | The start of all eight lines |
| Value | The field's contents, plain | The rest of each line |
| Line break | `\n` | Between every line, always |

There is no size, no weight beyond bold, and no alignment in the repository.
Google Chat's own settings decide how the text looks on a phone and on a
desktop.

## Components

### The chat message

- **Shape:** one object with one key, `text`. The value is eight lines
  joined by `\n`, with no trailing newline.
- **Content:** the ID, the project, the environment, the level, the culprit,
  the message, the platform and the URL, in that order.
- **Emphasis:** each line opens with `*Label*: ` and the label is closed
  with the same `*` before the colon.
- **Fallback:** the platform is the only field with one, and it is
  `unknown`. Every other field prints `None` when the payload does not carry
  it, because the message is built with `dict.get` without a default.
- **States:** there is one. The message is either sent or it is not, and
  neither state changes the message itself.

### The reply to Sentry

The route always answers with a JSON body, and the body always has one of
two keys:

```json
{"message": "Webhook received and forwarded to Google Chat successfully"}
{"message": "Environment not allowed. Skipping notification."}
{"error": "Failed to send message to Google Chat: ..."}
```

The status code is 200 in all three cases, and it stays 200 when the
forward failed. Sentry retries a non-2xx answer, and a retried alert nobody
wanted is worse than a lost one.

### The log lines

Loguru writes the failures and the received payload to standard output. There
is no format of its own, no level of its own, and no place where the
severity of an alert is decided.

## Do's and Don'ts

### Do:

- **Do** keep the eight lines in the same order. The order is the reading
  order, and a reader who has seen one alert knows where to look in the
  next.
- **Do** label every line. A value without a label cannot be read.
- **Do** keep the message to one `text` field. A second field would be
  rendered by Google Chat as a second card, and the space would show two
  things instead of one alert.
- **Do** keep the reply a 200. Sentry retries anything else.

### Don't:

- **Don't** add a colour, a level indicator or an emoji to the message. The
  service has no palette, and a glyph in a chat message reads as decoration
  on every device and as noise on the rest.
- **Don't** add a field without a reason written down in `PRODUCT.md`. The
  message grows by one line, and every reader of every future alert reads
  it.
- **Don't** introduce a second message shape. The forward sends one object
  with one key, and the reply is one of three bodies.
- **Don't** let the message carry a second language. The labels are English
  and the payload's own fields are passed through as they arrive.
