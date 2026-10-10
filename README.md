# inquiry-triage

Sorts website contact-form messages with Claude before they reach your inbox. Each message gets:

- a category: `lead`, `support`, `spam` or `other`
- a one-sentence summary
- the service, budget and timeline the sender stated, or `null` when they did not say
- a draft reply in the sender's language, for you to check and send

It runs on the contact form at [tengubyte.io](https://tengubyte.io).

## How it behaves

- **One API call.** It sends one request to the Claude Messages API with a forced tool call, so the answer comes back as JSON. The library validates that JSON before returning it.
- **Never throws.** On a timeout, an API error or output that fails validation, it returns `{ ok: false, reason }`. Your handler then sends the plain email as it did before, so a Claude outage never costs you an inquiry.
- **Never sends anything.** The draft reply comes to you. Nothing goes back to the sender automatically.
- **No runtime dependencies.** It only uses `fetch`, so it runs on Cloudflare Workers and Pages Functions, Node 22+, Deno, Bun and serverless functions.

## Install

```bash
npm install github:tengubyte/inquiry-triage
```

## Use

```ts
import { formatTriage, triageInquiry } from "inquiry-triage";

const result = await triageInquiry(
  { name, email, company, message },
  {
    apiKey: env.ANTHROPIC_API_KEY,
    business: "A two-person studio that builds websites and booking systems for clinics.",
  },
);

const subject = result.ok ? `[${result.triage.category}] New inquiry from ${email}` : `New inquiry from ${email}`;
const text = result.ok ? `${formatTriage(result.triage)}\n\n----\n\n${message}` : message;
// Send `subject` and `text` with your mail provider, exactly as you did before.
```

A subject tag like `[spam]` or `[lead]` is enough for a mail filter to sort the inbox.

### Options

| Option      | Default            | Notes                                                                                     |
| ----------- | ------------------ | ----------------------------------------------------------------------------------------- |
| `apiKey`    | required           | An Anthropic API key. Keep it in a secret, not in code.                                    |
| `business`  | required           | One or two sentences on what you sell. The model judges fit and writes the draft from it. |
| `model`     | `claude-haiku-5-5` | Any Claude model that supports tool use.                                                  |
| `timeoutMs` | `8000`             | After this, the call gives up and returns `{ ok: false, reason: "timeout" }`.             |

### Failure reasons

`timeout`, `request_failed` (network error), `http_<status>` (for example `http_401` for a bad key or `http_529` when the API is overloaded), and `bad_output` (the model's answer failed validation).

## What the email looks like

```
Category: lead
Summary: Wants a booking website for a dental clinic by March.
Service: booking website for our clinic
Budget: around $4,000
Timeline: before March
Language: en

Draft reply, check before sending:
Hi Sam, thanks for getting in touch. A booking site for the clinic is work we do often...

----

(original message)
```

## Cost

One call sends roughly 1,000 input tokens, including the tool definition, and gets back about 300 output tokens. On Haiku that comes to a fraction of a cent per inquiry. Messages longer than 10,000 characters are cut before sending, so a bot cannot run up the bill with a huge body. Your own form should still cap message length.

## Prompt injection

Treat the message as hostile. The prompt wraps it in `<inquiry>` tags and removes any copy of those tags from the sender's text, so the sender cannot close the block early. It also tells the model to treat everything inside the block as text to sort, and the tool schema limits what can come back.

That lowers the risk without removing it. A determined sender can still get a sales pitch labelled `lead` or steer the wording of the draft. For that reason the draft is never sent on its own. Do not connect the category to anything that acts without a person checking first, such as auto-deleting or auto-replying.

## Privacy

The visitor's name, email, business and message go to Anthropic's API. Name Anthropic in your privacy policy as a service that handles inquiries.

## Development

```bash
npm install
npm run typecheck
npm test
```

Tests use Node's built-in test runner and stub `fetch`, so they need no API key.

## License

MIT. Made by [TenguByte](https://tengubyte.io).
