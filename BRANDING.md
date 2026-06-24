# BRANDING.md — Public branding & store-listing identity

**Standing rule for every public-facing listing (Chrome Web Store, App Store, Product Hunt, landing pages, etc.).**

Mosh's identity has two halves — use BOTH, in different slots, never either/or:

- **Mosh Bari** = the visible personal brand. Goes in the *keyword/name* slot. It builds Mosh's personal brand, surfaces him in Google when people search his name, ties his whole portfolio of apps together, and a real named human reads as trustworthy for small tools.
- **ZPresso LLC** = the legal/publisher entity. Goes in the *publisher/"By ___"* slot. It signals an established software business and carries legal legitimacy. (Registered company; Apple Developer team `G6J9BUHSNH`.)

## Where each goes

| Slot | Use | Why |
|---|---|---|
| Product / extension **name** | include **Mosh Bari** | highest search weight + Google visibility for his name |
| **Publisher** / developer display name | **ZPresso LLC** | the "By ___" trust line; verify the domain for the blue **verified badge** (biggest single trust boost) |
| **Description / summary** | mention **both** — "Built by Mosh Bari, ZPresso LLC" | gets the name indexed in every field; covers Google too |

## Notes
- Search engines tokenize on spaces. A single unbroken token (e.g. `ZPressoLLC`) exact-matches that exact search; `ZPresso LLC` indexes as two common-ish words. Pick the form per goal — for *personal* SEO, "Mosh Bari" (his real name) is the right call over a company token nobody searches.
- Chrome Web Store: manifest `description` is capped at **132 characters** — keep the branded summary under that.
- The product name still leads with what the app *does* ("YT Transcript Scraper by Mosh Bari"), not the brand alone — keyword relevance first, brand second.

First applied: YT Transcript Scraper extension v2.7 (Jun 2026) — name "YT Transcript Scraper by Mosh Bari", publisher ZPresso LLC.
