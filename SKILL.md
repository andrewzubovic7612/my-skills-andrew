---
name: booking-best-price
description: >
  Use when the user pastes a holiday-rental or hotel listing link (Airbnb,
  Stayz, Vrbo, Expedia, Booking.com, HomeToGo, or similar) and wants to identify
  the actual property, find it listed on other sites, compare prices to book it
  cheaper, or track down the property manager / owner contact to book direct.
  Requires the Firecrawl MCP server. This skill verifies Firecrawl is installed
  and walks the user through setup first if it is not.
---

# Booking Best Price

## Overview

Booking sites and aggregators (Airbnb, Stayz, Vrbo, Expedia, Booking.com, HomeToGo) deliberately hide a property's real identity and the manager's contact so you cannot book around their fee. The same house is almost always listed on 5 to 15 sites through a single property manager, often including the manager's own website where it is cheapest.

This skill traces one pasted listing link back to the real property, finds every site it appears on, compares prices on the same dates, and surfaces the manager's direct contact. Goal: book the same house cheaper, or deal with the manager direct.

**Core idea:** pull a unique *fingerprint* from the listing (government licence number, exact name, manager/agency, a verbatim sentence of the description, the photos), then search that fingerprint across the web.

## Step 0 — Firecrawl gate (do this FIRST, every time)

OTA pages are JavaScript-heavy and bot-protected. You need the Firecrawl MCP server. Built-in WebFetch is a weak fallback only.

1. Check if it is connected. Run in Bash:
   ```bash
   claude mcp list
   ```
   Look for a `firecrawl` entry showing connected. (Or confirm a `mcp__firecrawl__firecrawl_scrape` tool is available.)

2. **If connected:** say "Firecrawl is connected" and go to Step 1.

3. **If missing:** tell the user they need Firecrawl and walk them through it. Do not run install commands without their go-ahead (it edits their config and needs their key).
   - Get a free API key: open [https://www.firecrawl.dev/app/api-keys](https://www.firecrawl.dev/app/api-keys), sign up, copy the key (starts with `fc-`). Free tier is enough to start.
   - Install (recommended, hosted):
     ```bash
     claude mcp add --transport http firecrawl https://mcp.firecrawl.dev/YOUR-API-KEY/v2/mcp
     ```
   - Or local:
     ```bash
     claude mcp add firecrawl -e FIRECRAWL_API_KEY=YOUR-API-KEY -- npx -y firecrawl-mcp
     ```
   - Replace `YOUR-API-KEY`. Then restart Claude Code and re-run `claude mcp list` to confirm.

## Step 1 — Scrape the pasted link

Scrape the listing as full content. Render the page and beat anti-bot:
- `formats: ["markdown", "links"]`, `onlyMainContent: false`, `waitFor: 8000`.
- If the result is thin or blocked, retry with `proxy: "stealth"` and `waitFor: 10000`, and set `location` to the property's country (e.g. `{country: "ES"}`).

## Step 2 — Pull the fingerprint

Extract every identifier you can. The licence number and a verbatim description sentence are the strongest keys.

| Identifier | Why it matters |
|---|---|
| Government tourist licence / registration no. (e.g. `ETV/6027`, VT-, CTC) | Near-unique, especially in the EU. Best single key. |
| Exact property name / listing title | Direct search hit on other sites. |
| Manager / agency / host name ("Central de Reservas", a company) | Leads to the direct-booking site. |
| Partial / approximate address + postcode | Pins it on a map; OTAs often expose area-level. |
| One verbatim sentence of the description | Quote it in search to surface clones. |
| Distinctive amenity combo (e.g. "ping-pong, BBQ, 7 sunbeds") | Disambiguates near-matches. |
| Photo URLs | Reverse-image search if text fails. |
| Per-platform internal IDs (Booking `hotel_id`, Vrbo/Stayz `p`-number, Airbnb room id) | Cross-references the same group's sites. |

## Step 3 — Find every channel

Run several searches in parallel (one message, multiple calls). Use `firecrawl_search`:
- The licence number in quotes.
- The exact property name + town.
- A quoted unique sentence from the description.
- The manager/agency name + town + "holiday rental".

Collect the listing on every site it appears: Airbnb, Vrbo/Stayz, Expedia/Hotels.com, Booking.com, plus regional portals and the manager's own domain.

## Step 4 — Identify the manager and their direct site

The manager named on the listing usually runs their own booking site (and sometimes sister brands). That direct channel carries no OTA margin and is typically the cheapest, and it gives you a real human to negotiate with. Find it (`site:` search the manager name) and note their booking reference for the property.

## Step 5 — Price-compare cleanly

Get a real, dated quote from 2 to 3 channels plus the direct site. Avoid false comparisons:
- **Same dates, same guests, same currency.** Append date params so the page prices the actual stay. Booking.com example:
  `?checkin=2026-07-01&checkout=2026-07-19&group_adults=6&no_rooms=1&group_children=0&selected_currency=EUR`
- Compare **all-in** totals (cleaning + tourist tax + fees), not headline nightly rates.
- **Ignore aggregator teaser prices.** The price in a HomeToGo/aggregator URL often differs from the checkout price. Only trust the figure on the live property page at the right dates.

## Step 6 — Get the contact

- The **manager** is the reachable contact: company name, phone, email, address, and their own booking page. This is your real counterparty and where you negotiate.
- Be honest that the **individual private owner** is almost never published. Every channel routes through the manager by design. Do not imply you can get a private owner's personal number when you cannot.

## Step 7 — Deliver

Give the user, tightly:
1. **What it is** — name, location, key features, licence number.
2. **Where it is listed** — table of channels with links.
3. **Price comparison** — all-in totals for the dates, cheapest flagged, teaser prices called out as unreliable.
4. **Direct contact** — manager name, phone, email, direct booking link + property reference.
5. **Recommended move** — usually email/call the manager with the dates and reference and ask for their best direct all-in price.

## Platform notes

| Link | Notes |
|---|---|
| **Booking.com** | Exposes manager ("company info"), partial address, licence number, `hotel_id`. Add `&selected_currency=EUR` and date params for an all-in total. |
| **Stayz** | Australian brand of Vrbo. Its `p`-number is the same property on Vrbo worldwide and often on Expedia/Hotels.com (all Expedia Group). |
| **Expedia / Hotels.com** | Same group as Vrbo/Stayz. Shows property name + manager. Cross-check the other group sites. |
| **Airbnb** | Strong anti-bot: use `proxy: "stealth"`, `waitFor: 10000`. Hides exact address and host surname, but shows title, host first name, "managed by", neighbourhood, review text. The same manager usually also lists on Vrbo/Booking. |
| **HomeToGo / aggregators** | Not a host, a meta-search. Exposes the underlying provider/manager, licence, and full description, which is gold for the fingerprint. Distrust its quoted price until you reach the provider's checkout. |

## Common mistakes

- Comparing different currencies or different dates and declaring a "winner".
- Quoting the aggregator URL/teaser price as the real price.
- Comparing headline nightly rates instead of all-in totals.
- Claiming you found the private owner's contact when it is really the manager.
- Skipping Step 0 and then failing on a blocked Airbnb/Booking page.

## Scope

This is for booking the same property cheaper or direct, and for legitimate price research. It is not for harassing owners, evading genuine taxes or deposits, or scraping personal data. Surface the public business contact of the manager, nothing more.
