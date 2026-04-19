# Trenix — AI-Powered Fitness Planning Platform

> Mobile-first web platform that generates personalized training and nutrition plans using AI, then adapts them weekly based on real user progress.

**Live demo:** [trenix.net](https://trenix.net)

---

## Overview

Trenix is a structured fitness planning system designed for gym users who want the value of a coach without the recurring cost of traditional 1:1 services.

After a one-time onboarding flow, the platform generates a full training and nutrition plan tailored to the user’s goal, level, constraints, and availability. A short weekly check-in then updates the program automatically according to real-world adherence, body metrics, and training progression.

This is not a static workout app. It is an adaptive preparation system.

---

## Key value proposition

- **AI-generated training and nutrition plans**
- **Weekly adaptive adjustments based on real user data**
- **Structured progression logic instead of static templates**
- **Biomechanical exercise substitutions**
- **Premium subscription model with automated billing**
- **Mobile-first UX with PWA support**

---

## Tech stack

### Frontend
| Technology | Purpose |
|---|---|
| **Next.js 16** (App Router) | Main frontend framework, SSR, RSC, routing |
| **TypeScript** | End-to-end type safety |
| **Tailwind CSS v4** | UI styling, design tokens, dark theme |
| **Vercel** | Frontend hosting, CDN, edge delivery |

### Backend
| Technology | Purpose |
|---|---|
| **FastAPI** (Python 3.12) | REST API, async endpoints, typed backend services |
| **SQLAlchemy 2.0** | ORM and query layer |
| **Alembic** | Database migrations |
| **Uvicorn** | ASGI application server |
| **Docker** | Containerized backend deployment |

### Infrastructure & integrations
| Technology | Purpose |
|---|---|
| **PostgreSQL (Neon)** | Primary relational database |
| **Redis** | Reminder queue and background task support |
| **Render** | Backend hosting |
| **Stripe** | Premium subscriptions and webhook processing |
| **Resend** | Transactional email and weekly reminders |
| **Google OAuth 2.0** | Social authentication |

### AI layer
| Technology | Purpose |
|---|---|
| **Anthropic Claude API** | Plan generation engine |
| **Pydantic v2** | Strict schema validation for LLM outputs |

---

## System architecture

```text
┌─────────────────────────────────────────────────────────┐
│                     trenix.net (Vercel)                │
│                                                         │
│  Next.js App Router                                     │
│  ├── Server Components (landing, SEO, metadata)         │
│  ├── Client Components (auth, training, nutrition)      │
│  └── Edge Functions (OG image generation)               │
└──────────────────────┬──────────────────────────────────┘
                       │ HTTPS + JWT
                       │ Refresh token via HttpOnly cookie
┌──────────────────────▼──────────────────────────────────┐
│              FastAPI API (Render + Docker)              │
│                                                         │
│  Modules                                                 │
│  ├── /auth       JWT, Google OAuth, Magic Link          │
│  ├── /users      User profile and state                 │
│  ├── /plans      Plan generation and retrieval          │
│  ├── /training   Session logging and progression        │
│  ├── /checkin    Weekly review and plan adjustment      │
│  ├── /subscribe  Stripe subscriptions and webhooks      │
│  └── /admin      Internal admin operations              │
│                                                         │
│  AI Planning Engine                                      │
│  ├── Prompt construction                                │
│  ├── Claude API call                                    │
│  └── Pydantic schema validation                         │
└──────────┬────────────────────────┬─────────────────────┘
           │                        │
┌──────────▼──────────┐  ┌──────────▼──────────┐
│ PostgreSQL (Neon)    │  │ Redis               │
│                      │  │                     │
│ users                │  │ reminder queue      │
│ plans                │  │ weekly jobs         │
│ workout_sessions     │  │ worker support      │
│ checkins             │  │ push notification   │
│ subscriptions        │  └─────────────────────┘
└──────────────────────┘
```
## Core features

### User onboarding
- Goal, experience level, and weekly availability collection
- Physical metrics and body composition inputs
- Dietary restrictions and nutrition preferences
- Initial user state used to generate both training and nutrition plans

### AI plan generation
- Full training split generation
- Structured nutrition planning with macro targets
- Exercise alternatives based on biomechanical similarity
- Output validated against strict schemas before persistence

### Training workflow
- Weekly training structure with exercises, sets, reps, RPE/RIR, and cardio
- Session logging with real load tracking
- Progressive overload support based on actual performance data

### Nutrition workflow
- Daily macro distribution
- Meal structure with food substitutions
- Adaptation to phase, preferences, and restrictions

### Weekly progression engine
- 2-minute weekly check-in
- Tracks body weight, adherence, recovery, and subjective feedback
- Adjusts training volume, calories, and progression logic automatically

### Authentication & product
- JWT authentication with refresh token rotation
- Google OAuth 2.0
- Magic link email login
- Email verification
- Premium subscriptions via Stripe
- Admin dashboard for users, plans, and generation traces
- PWA support
- Web push notifications and weekly reminder emails

---

## Engineering highlights

### 1. LLM output validation with strict contracts
LLM responses are never trusted as free-form text. Every generated plan is parsed against a strict **Pydantic v2 schema**.

- Invalid fields are rejected
- Missing required fields block generation
- Incorrect types or malformed structures never reach the database

If validation fails, the system returns a controlled blocked state instead of persisting corrupted data.

```python
generation_status = "blocked"
```

This prevents one of the most common failure modes in AI-integrated products: silently accepting structurally invalid outputs.

---

### 2. Secure auth flow with refresh token isolation
The access token is stored **in memory only**, while the refresh token is stored in an **HttpOnly cookie**.

This design reduces exposure to XSS-based token theft and avoids the common anti-pattern of storing long-lived credentials in `localStorage`.

A silent sync against `/auth/me` is performed on app load to restore authenticated state and subscription tier without blocking the UI.

---

### 3. Schema evolution without breaking legacy plans
As the planning engine evolves, generated plan schemas also evolve.

To preserve compatibility with previously stored plans:
- new generation schemas are strict (`extra="forbid"`)
- read-side compatibility models are tolerant where needed (`extra="ignore"`)

This allows the product to iterate on plan structure without breaking historical data already stored in production.

---

## SEO & performance

**Lighthouse audit (mobile):**

| Metric | Result |
|---|---|
| **Performance** | **99 / 100** |
| **SEO** | **100 / 100** |
| **Accessibility** | **97 / 100** |
| **First Contentful Paint** | 1.0 s |
| **Largest Contentful Paint** | 2.0 s |
| **Total Blocking Time** | 10 ms |
| **Cumulative Layout Shift** | 0 |

### SEO implementation
- Dynamic `sitemap.xml`
- `robots.txt` with sitemap declaration
- Structured data via JSON-LD:
  - `SoftwareApplication`
  - `WebSite`
  - `Organization`
  - `FAQPage`
- Dynamic Open Graph image generation with `ImageResponse`
- Open Graph and Twitter metadata
- Server-rendered critical landing content to reduce client bundle cost

---

## Product status

Trenix is an actively developed portfolio project focused on combining:

- applied AI
- typed backend systems
- modern frontend architecture
- real product constraints such as billing, auth, notifications, and SEO

The source code is private. This repository exists as a technical project overview for portfolio and hiring purposes.

---

## Author

Built by [Isaac Román](https://github.com/IsaacRomanSerrano)
