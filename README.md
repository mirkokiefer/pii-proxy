# pii-proxy

Privacy proxy for AI agents. Mask PII before sending to LLMs, unmask responses to write back to real systems.

## Why

Your AI agent processes emails, spreadsheets, CRM data. You don't want to send real names, emails, and tracking numbers to Claude or GPT. But token-based masking (`PERSON_1`, `EMAIL_2`) degrades model quality — LLMs reason poorly over meaningless tokens.

**pii-proxy** replaces PII with plausible fake values — the LLM sees realistic data and reasons correctly. A bijective map lets you reverse everything when writing back to your database.

## Install

```bash
npm install @daslab/pii-proxy
```

## Quick start

```typescript
import { PrivacyProxy } from '@daslab/pii-proxy';

const proxy = new PrivacyProxy();

// Mask PII with plausible fakes
const masked = proxy.mask(
  "Ship order to mirko@kiefer.com, tracking AETH0000345323DY"
);
// → "Ship order to alex@johnson.net, tracking BFUI0000482918EZ"

// Send masked.text to your LLM...

// Reverse all fakes back to real values
const real = proxy.unmask(llmResponse);
// "I'll notify alex@johnson.net" → "I'll notify mirko@kiefer.com"
```

## How it works

1. **Detect** — regex-based detectors find emails, tracking numbers, IPs, UUIDs, credit cards, phone numbers, and URLs with tokens.
2. **Replace** — each entity is replaced with a plausible fake of the same type (an email becomes another email, a tracking number keeps the same format).
3. **Map** — a bijective map ensures the same real value always maps to the same fake, and vice versa. Consistent within a session, reversible at any time.

```
Real:   "Contact mirko@kiefer.com about AETH0000345323DY"
         ↓ mask()
Fake:   "Contact alex@johnson.net about BFUI0000482918EZ"
         ↓ send to LLM → get response
LLM:    "I've emailed alex@johnson.net about the shipment"
         ↓ unmask()
Real:   "I've emailed mirko@kiefer.com about the shipment"
```

## Entity types

| Type | Detection | Fake replacement |
|---|---|---|
| Email | `user@domain.com` | Realistic fake email |
| Phone | `+1-234-567-8901` | Format-preserving fake |
| Credit card | Luhn-validated numbers | Valid fake card number |
| IP address | IPv4 addresses | Random valid IP |
| UUID | Standard UUID format | Random UUID |
| URL | URLs with query params/tokens | Sanitized URL |
| Tracking number | UPS, USPS, DHL, AliExpress, etc. | Format-preserving fake |

## Structured data

Mask entire objects (e.g., tool call inputs):

```typescript
const { masked } = proxy.maskObject({
  to: "mirko@kiefer.com",
  subject: "Order update",
  body: "Tracking: AETH0000345323DY",
  metadata: { ip: "10.0.0.1" }
});

// masked.to → "alex@johnson.net"
// masked.subject → "Order update" (no PII, unchanged)
// masked.body → "Tracking: BFUI0000482918EZ"
// masked.metadata.ip → "172.45.123.89"

// Reverse everything
const original = proxy.unmaskObject(masked);
```

## Persistence

Save and restore the map across sessions:

```typescript
// Save
const data = proxy.getMap().serialize();
await redis.set('pii-session:123', data);

// Restore in a new process
const proxy2 = new PrivacyProxy();
proxy2.loadMap(await redis.get('pii-session:123'));
proxy2.unmask(text); // works with the same mappings
```

## Roadmap

- [x] **v0.1** — Regex detection, faker replacement, bijective round-trip
- [ ] **v0.2** — NER-based name/location detection (optional Presidio backend)
- [ ] **v0.3** — Tool-aware selective masking (keep location real for hotel search, mask for email)
- [ ] **v0.4** — Persistent map backends (Redis, SQLite)
- [ ] **v0.5** — Anthropic/OpenAI SDK middleware (drop-in agent integration)

## License

MIT
