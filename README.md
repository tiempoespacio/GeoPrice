# Precios Cerca · Prices Near — Neighborhood Price Finder

**Live:** https://geoprice.dapp.cl
**Demo video (2:18):** https://www.youtube.com/watch?v=eqKYygr2MWg

A neighborhood price index built for the **Meta Global AI Developer Hackathon**
(theme: *Build for Your World*), powered by **Muse Spark** via the Meta Model API.

People type what they saw, in their own words — *"handroll for 3000 at Alameda
esquina Cumming"* — and the AI turns it into structured, geolocated, categorized
data. Browsing is free for everyone: no account, no wallet. Publishing requires
MetaMask as an **identity gate** (off-chain, **0 gas**): the wallet is who you
are, not a payment.

> Prices Near is the territorial (6th) dimension of a multidimensional
> marketplace ecosystem: look (free, no wallet) → publish (MetaMask, 0 gas) →
> sell on the marketplace (where gas appears, and sellers receive 99% in DAI:
> 0.5% marketplace + 0.5% future user benefit). A practical blockchain course
> with no tutorials — the need for gas is born later, not at the door.

## What it does

- **AI intake (Muse Spark, server-side):** free-text reports are structured
  into item, price, unit, zone, shop and one of 18 clear categories aligned
  with the marketplace taxonomy (bakery, dairy, meat, produce, groceries,
  drinks, prepared food, household, transport, fuel, pharmacy, lodging,
  leisure, tech, pets, fashion, vehicles, services). No clear category → the
  server rejects the publish (`need_categoria`).
- **Geolocation, orchestrated in stages** (each stage visible in the API
  response): 1) **device GPS** exact point at publish time (with permission,
  accuracy in meters) → 2) **named place** resolved against OpenStreetMap,
  bounded and disambiguated by the Cloudflare edge location → 3)
  **Cloudflare** `request.cf` (city / region / country / approximate coords)
  as anchor and fallback. No coordinates → no publish (`need_geo`).
- **Strict by design:** wallet required (401 without it), item/price/zone
  required, clear category required, resolved coordinates required.
- **Every observation is pinned to IPFS** (Pinata) and gets its own SEO page
  (`/precio/:id`, Open Graph + JSON-LD `Product/Offer`) plus `sitemap.xml`.
- **Shops can claim their place** ("This is my shop" / "I sell this"): a
  verified owner is the only one who gets an exact public point. Observers
  keep their authorship; claiming never transfers it.
- **Map** of the neighborhood with price pins (OpenStreetMap tiles served
  through the worker's own edge cache, ~300 m privacy jitter; exact points
  are never displayed for third parties).
- **11 languages**, PWA, dark/neon minimal UI. No trackers, no cookies,
  no accounts, no KYC.

## Architecture

Single self-contained Cloudflare Worker (`worker.js`) — no build step:

| Piece | Tech |
|---|---|
| Edge app + API | Cloudflare Worker (this repo) |
| AI | Meta Model API — Muse Spark (`muse-spark-1.3`), server-side secret |
| Storage | Cloudflare D1 (table `precios_barrio`) |
| Content addressing | IPFS via Pinata (JSON per observation, wallet never included) |
| Geo | Device GPS → OpenStreetMap Nominatim → Cloudflare `request.cf` |
| Map tiles | OSM via worker proxy `/tiles/{z}/{x}/{y}.png` (edge-cached) |

Key endpoints:

- `POST /api/v1/assist` — free text in, structured draft out (Muse live;
  honest local demo mode when `META_API_KEY` is not configured)
- `POST /api/v1/precios` — publish (wallet + strict validation + geo stages)
- `GET  /api/v1/precios` — public listing (never exposes wallets)
- `POST /api/v1/reclamar` — claim a place as owner/seller
- `GET  /precio/:id` — SEO page per observation · `GET /sitemap.xml`

## Run your own

```bash
npx wrangler d1 create your-d1-database-name
# create the precios_barrio table (schema below)
npx wrangler secret put META_API_KEY    # Meta Model API (optional: demo mode without it)
npx wrangler secret put PINATA_JWT      # IPFS pinning (optional)
npx wrangler deploy
```

Secrets are **only** ever Cloudflare secrets or env references
(`env.META_API_KEY`, `env.PINATA_JWT`). None are committed to this repo;
`wrangler.toml` here is sanitized with placeholders.

### `precios_barrio` schema

```sql
CREATE TABLE precios_barrio (
  id INTEGER PRIMARY KEY AUTOINCREMENT,
  item TEXT, precio REAL, unidad TEXT, zona TEXT,
  observer_wallet TEXT, store_name TEXT, verified INTEGER DEFAULT 0,
  lat REAL, lng REAL,
  geo_city TEXT, geo_region TEXT, geo_country TEXT, lang TEXT,
  categoria TEXT, geo_source TEXT, geo_accuracy REAL,
  cid_ipfs TEXT, promoted_to_product INTEGER DEFAULT 0,
  created_at TEXT DEFAULT CURRENT_TIMESTAMP
);
```

## Privacy

Browsing needs nothing. Publishing stores the minimum: the observation, its
approximate/exact point (exact point stays in the database; IPFS carries a
~100 m rounding), and a pseudonymous wallet address. No names, no emails,
no tracking cookies, no third-party analytics.

## License

MIT
