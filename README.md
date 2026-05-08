# SwapnYatra

> **India's budget-first, AI-powered travel planning super app.**
> Plan, book, and explore — without expensive travel agents.

[![Status](https://img.shields.io/badge/status-MVP-yellow)]()
[![Stack](https://img.shields.io/badge/stack-TypeScript%20%7C%20Next.js%20%7C%20Express-0F766E)]()
[![Scope](https://img.shields.io/badge/scope-Domestic%20India-F59E0B)]()
[![License](https://img.shields.io/badge/license-Private-lightgrey)]()

> The source code lives in a **private repository**. This page is a public showcase of the architecture, technology, and product thinking behind it.

---

## The Problem

India has ~500 million domestic travelers a year. Most face the same broken planning loop:

1. Search "places to visit in [destination]" — get 10 generic blogs.
2. Cross-check IRCTC, RedBus, Skyscanner, MakeMyTrip, Booking.com — five tabs deep.
3. Hand-build a Google Sheet with timings, costs, hotels.
4. Give up halfway and pay a travel agent 20–40% markup.

Existing platforms sell **tickets**. Nobody sells the **plan**.

## The Solution

**SwapnYatra is the missing layer between inspiration and booking.** Type your source, destination, dates, budget, and interests — get a complete day-by-day itinerary with realistic transport, stays, sights, and a transparent budget breakdown.

Built specifically for **middle-class travelers, students, families, solo travelers, and budget-first explorers**.

---

## Key Features

- **AI-powered itinerary generation** with interpretable scoring (interest + season + budget + travel style)
- **Anywhere → anywhere in India** via OpenStreetMap geocoding + routing — no hardcoded city list
- **Multi-modal transport** with realistic timings: train, bus, flight, cab
- **Budget-first design** — every recommendation knows its price; full breakdown of transport / stay / food / activities / buffer
- **Hidden-gem boost** — de-ranks over-touristed spots in favor of low-traffic alternatives
- **Live booking deep-links** to RedBus, ConfirmTkt, Skyscanner, Uber with from/to/date prefilled
- **15+ activity filters** (trekking, beach, heritage, spiritual, wildlife, photography, food, snow, etc.)
- **Accessibility-aware** — wheelchair and low-walking filters at planner and place level
- **Group + family mode** with travel-style-aware budget calibration
- **Three-tier resolution** for arbitrary inputs: curated seed → live OSM → graceful empty state

---

## Tech Stack

### Frontend
- **Next.js 14** (App Router) + **TypeScript**
- **React 18** with Hooks-based components
- **Tailwind CSS** for styling
- **Axios** for API calls
- Mobile-responsive, dark-mode-ready

### Backend
- **Node.js 20** + **Express** + **TypeScript** (strict mode)
- **Zod** for runtime input validation
- **Modular monolith** architecture — clean module boundaries ready for microservice extraction
- **uuid v4** for trip identifiers
- **In-memory cache** for external API calls

### Data
- **PostgreSQL 15** schema (designed; in-memory seed for MVP)
- Full-text search via `pg_trgm` and tsvector
- Production: AWS RDS / Supabase
- Phase 3 plan: Redis cache, Elasticsearch, ClickHouse

### Free External APIs (no key required)
- **OpenStreetMap Nominatim** — geocoding any place name in India
- **OSRM** — driving distance + duration between coordinates
- **Overpass API** — live tourist attraction discovery
- **Wikipedia REST API** — place summaries

### Infra (designed for)
- **AWS ECS Fargate** + **API Gateway**
- **Cloudflare** CDN
- **GitHub Actions** CI/CD with blue-green deploys
- **Sentry** + **Grafana Cloud** observability

---

## Architecture

```
                ┌──────────────────────────────┐
                │   Mobile (RN)  +  Web (Next) │
                └────────────┬─────────────────┘
                             │ HTTPS
                ┌────────────▼─────────────────┐
                │      API Gateway / Nginx     │
                └────────────┬─────────────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
   ┌────▼─────┐        ┌─────▼─────┐        ┌─────▼─────┐
   │ Trip     │        │ Booking   │        │  AI /     │
   │ Service  │        │ Service   │        │  Reco     │
   └────┬─────┘        └────┬──────┘        └─────┬─────┘
        │                   │                     │
   ┌────▼─────┐        ┌────▼──────┐        ┌─────▼─────┐
   │ Places   │        │ Transport │        │ Budget    │
   └────┬─────┘        └────┬──────┘        └───────────┘
        │                   │
        └─────┬─────────────┘
              │
   ┌──────────▼───────────┐    ┌──────────────┐
   │  PostgreSQL (RDS)    │    │  Redis cache │
   └──────────────────────┘    └──────────────┘
              │
   ┌──────────▼───────────┐
   │  S3 (images, KMLs)   │
   └──────────────────────┘
```

External: Nominatim, OSRM, Overpass, Wikipedia (free) · IRCTC, RedBus,
MakeMyTrip, Razorpay, Twilio, FCM (Phase 2+)

### Service modules

| Module | Responsibility |
|---|---|
| **Trip Planner** | Orchestrates AI ranking → places → transport → hotels → budget |
| **AI Engine** | Interpretable rules + heuristic scoring (Phase 3: ML/LLM refinement) |
| **Transport** | Multi-modal leg builder with OSRM-backed distances and partner deep-links |
| **Hotels** | Aggregator-shaped service; Booking, Agoda, OYO, Zostel adapters |
| **Budget** | Single source of truth for all cost math + cost-cut suggestions + group splitter |
| **Discovery** | Live POI fetch from Overpass when seed has no coverage |
| **Booking Engine** | Affiliate intent + webhook reconciliation |
| **Community** | Reviews, photos, shared itineraries with auto-moderation |
| **Admin** | Place CRUD, review moderation, audit trail |

### Data flow: trip generation

```
User → POST /api/trip/generate { source, dest, days, budget, interests }
  ↓
  Trip Service (orchestrator)
    ├─→ AI Engine: rank places (interests × season × budget × style)
    ├─→ Discovery: live fallback via Overpass when seed is empty
    ├─→ Transport: OSRM distances → bus/train/flight selection
    ├─→ Hotel: per-cluster stay picks under budget ceiling
    └─→ Budget: rollup with 7% buffer + cost-cut tips
  ↓
Returns full itinerary JSON with day-by-day timeline + booking deep-links
```

---

## Product Decisions Worth Highlighting

1. **Budget-first, not luxury-first.** Every UI element shows price. No "from ₹X" upsells.
2. **Plan, don't sell.** Booking is downstream of planning, not the other way around.
3. **Interpretable AI by default.** The planner explains *why* every place was picked. Black-box LLMs come in Phase 3 only as a refinement layer over the rules.
4. **No fake live data.** We don't pretend to have IRCTC schedules. We show realistic typical times and deep-link to authoritative sources for verification.
5. **India-native, not localized.** Built for IRCTC quirks, regional cuisines, monsoon timing, INR pricing — not a translated Booking.com.
6. **Hidden-gem boost.** Low-popularity places with high interest match get a +0.1 boost in ranking. Goa is fine; Gokarna is the gem.
7. **18-section spec layout.** Every domain (vision, personas, architecture, monetization, security, roadmap, case study, etc.) is its own folder with a single source of truth.

---

## Roadmap

| Phase | Focus | Target |
|---|---|---|
| **Phase 1 — MVP** ✅ | Web planner, budget engine, OSM fallback, deep-link booking | 100 beta users |
| **Phase 2** | Native apps, IRCTC + RedBus partnerships, multi-language UI, premium tier | 50K MAU |
| **Phase 3** | ML personalization, LLM itinerary refinement, OR-Tools route optimization | 500K MAU |
| **Phase 4** | International outbound (Southeast Asia), B2B2C white-label, voice UI | 5M MAU |

---

## What I Learned

- **Multi-tier fallback design.** Free APIs aren't always available — the planner gracefully degrades from curated seed → live Overpass → empty-with-helpful-message.
- **API rate-limit etiquette.** Nominatim's 1-req/sec limit means in-memory caching and User-Agent honesty matter more than fancy retry logic.
- **Budget transparency as a feature, not a chore.** Showing the math (`₹2,500 = ₹800 train + ₹1,200 stay + ₹500 food`) builds trust faster than any marketing copy.
- **TypeScript strict mode forces honest interfaces.** `noUncheckedIndexedAccess` caught half a dozen sloppy array accesses I'd have shipped otherwise.
- **DI keeps services testable.** Every service takes its dependencies as a parameter — swapping the mock for a real adapter is a one-line change.
- **Modular monolith first, microservices second.** Folder boundaries (`05_Trip_Planner`, `06_Booking_Engine`, …) let me extract services later without rewrites.
- **Domain-modeling India.** Train + bus is cheaper than flight for most middle-distance trips. Twin-shared rooms are the family default. Heritage sites are usually free or under ₹50. These aren't corner cases — they're the product.

---

## Status & Demo

- **Repository:** Private (this page is the public-facing summary)
- **Live demo:** _(planned)_
- **Screenshots:** see `/screenshots` folder _(planned)_

If you'd like a code walkthrough or architecture deep-dive, [reach out](mailto:biradar.swapnil01@gmail.com).

---

## Author

Built by **Swapnil Biradar** — solo product + engineering.

[LinkedIn](#) · [Email](mailto:biradar.swapnil01@gmail.com)

> SwapnYatra means "dream journey" in Sanskrit/Hindi. The product exists so anyone in India, regardless of city or budget, can plan one.
