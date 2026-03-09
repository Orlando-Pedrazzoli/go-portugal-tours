# FollowerScan — Instagram Follower Analysis Tool with SaaS Subscription

**Live:** [followerscan.com](https://www.followerscan.com)

A web application that analyses Instagram data exports to identify who doesn't follow you back, detect possible blocks, and track follower changes over time. All data processing happens in the browser — no user data is sent to any server. The project has a freemium model with Clerk authentication and Stripe subscriptions.

---

## Context

This started as a personal project — I wanted to understand who had unfollowed me on Instagram without using shady third-party apps that ask for your password. Instagram lets you download your data as JSON files, but the raw data is hard to read. I built a tool that parses those files and presents the analysis in a usable way.

What makes this different from similar tools: the data never leaves the browser. The JSON parsing, relationship analysis, and block detection all run client-side. The server-side code only handles authentication (Clerk) and subscription management (Stripe) — it never sees the Instagram data.

I later added a SaaS layer with Clerk auth and Stripe subscriptions to experiment with monetisation. Free users get limited analysis; paid users unlock the full dashboard.

---

## Tech Stack

**Framework:** Next.js 16, React 19, TypeScript

**UI:** Tailwind CSS 3, Framer Motion, Radix UI (Dialog, Progress, Tabs, Toast, Slot), class-variance-authority, tailwind-merge, Recharts

**Authentication:** Clerk (sign-in, sign-up, session management, localizations)

**Payments:** Stripe (checkout sessions, webhooks, subscription management)

**Data Processing (client-side):** JSZip (ZIP extraction), jsPDF + jspdf-autotable (PDF report generation), React Dropzone (file upload)

**Infrastructure:** Vercel

---

## Architecture

The app has two distinct layers: the client-side analysis engine (no server involved) and the server-side SaaS infrastructure (Clerk + Stripe).

```
src/
├── app/
│   ├── page.tsx                        # Landing page
│   ├── layout.tsx                      # Root layout with providers
│   ├── providers.tsx                   # Clerk + theme providers
│   ├── not-found.tsx
│   ├── robots.ts                       # Dynamic robots.txt
│   ├── sitemap.ts                      # Dynamic sitemap
│   │
│   ├── upload/                         # File upload page
│   ├── analyze/                        # Free analysis page
│   ├── tutorial/                       # Step-by-step guide
│   ├── sample/                         # Demo with sample data
│   │
│   ├── dashboard/
│   │   ├── layout.tsx                  # Protected layout (requires auth)
│   │   └── analyze/                    # Full analysis (requires subscription)
│   │
│   ├── pricing/                        # Subscription plans
│   │
│   ├── auth/
│   │   ├── sign-in/[[...sign-in]]/     # Clerk sign-in (catch-all route)
│   │   └── sign-up/[[...sign-up]]/     # Clerk sign-up (catch-all route)
│   │
│   ├── api/
│   │   ├── subscription/
│   │   │   ├── create-checkout/route.ts    # Creates Stripe checkout session
│   │   │   └── increment-usage/route.ts    # Tracks usage for metered billing
│   │   └── webhooks/stripe/route.ts        # Stripe webhook handler
│   │
│   ├── privacy/                        # Privacy policy
│   ├── terms/                          # Terms of service
│   ├── cookies/                        # Cookie policy
│   ├── gdpr/                           # GDPR information
│   └── faq/                            # FAQ
│
├── components/
│   ├── home/
│   │   └── HomeClient.tsx              # Landing page client component
│   ├── subscription/
│   │   └── UpgradePrompt.tsx           # Paywall / upgrade CTA
│   ├── ui/
│   │   ├── RevolutionaryDashboard.tsx  # Main analysis dashboard
│   │   ├── button.tsx                  # CVA button variants
│   │   ├── card.tsx
│   │   ├── badge.tsx
│   │   ├── input.tsx
│   │   ├── progress.tsx
│   │   ├── tabs.tsx
│   │   ├── header.tsx
│   │   ├── footer.tsx
│   │   └── cookie-consent.tsx
│   └── theme-provider.tsx              # Dark/light mode
│
├── lib/
│   ├── instagram-parser.ts             # Core: JSON parsing + analysis engine
│   ├── subscription.ts                 # Stripe subscription helpers
│   └── utils.ts                        # cn() helper, utilities
│
├── hooks/
│   └── useSubscription.ts              # Subscription status hook
│
├── types/
│   ├── instagram.ts                    # Instagram data interfaces
│   └── jspdf-autotable.d.ts           # Type declarations for PDF lib
│
└── proxy.ts
```

---

## Key Features

### Client-side Instagram data analysis

The core of the application is `instagram-parser.ts` — a TypeScript module that processes Instagram's data export files entirely in the browser. It handles:

- **ZIP extraction** — Users upload the ZIP file they downloaded from Instagram. JSZip extracts it in memory, no server upload needed.
- **JSON parsing** — The parser recognises 12+ file types from Instagram's export: followers, following, close friends, blocked profiles, recently unfollowed, follow requests, pending requests, hidden story list, followed hashtags, restricted profiles, and removed suggestions.
- **Relationship analysis** — Cross-references followers vs. following to identify: people who don't follow back, mutual followers, possible blocks (based on behavioural patterns), recent unfollows, and categorised relationships (VIPs, red flags, etc.).
- **Block detection** — An algorithm that estimates possible blocks by analysing patterns across multiple data points: unfollow history, pending requests, and temporal changes. The accuracy is estimated at 60-90% depending on available data.

### Authentication with Clerk

Clerk handles the entire auth flow through Next.js catch-all routes (`[[...sign-in]]` and `[[...sign-up]]`). The `providers.tsx` wraps the app with Clerk's provider. The dashboard routes are protected — unauthenticated users are redirected to sign-in.

Clerk was chosen over a custom JWT solution because it provides a complete auth UI out of the box (sign-in, sign-up, user management, session handling) with no backend code. For a side project where auth isn't the product, this trade-off makes sense.

### Stripe subscriptions

The SaaS model works like this:

1. Free users can access basic analysis at `/analyze`
2. The full dashboard at `/dashboard/analyze` requires authentication and an active subscription
3. `UpgradePrompt.tsx` acts as a paywall when free users try to access premium features
4. The `/pricing` page shows available plans
5. When a user subscribes, the flow is:

```
User clicks "Subscribe"
  → Frontend calls POST /api/subscription/create-checkout
    → Server creates a Stripe Checkout Session
      → User completes payment on Stripe's hosted page
        → Stripe sends webhook to /api/webhooks/stripe
          → Server updates user's subscription status
```

Usage tracking happens through `/api/subscription/increment-usage`, and the `useSubscription` hook provides subscription status to any component that needs it.

### PDF report generation

Users can export their analysis as a PDF report using jsPDF with the autotable plugin. The report includes follower statistics, relationship breakdowns, and the list of accounts in each category — all generated client-side.

### Legal compliance

The app has dedicated pages for Privacy Policy, Terms of Service, Cookie Policy, GDPR information, and FAQ. The cookie consent component manages consent state. Dynamic `robots.ts` and `sitemap.ts` files are generated for SEO.

---

## Privacy model

This is worth explaining because it's the main selling point. The Instagram data (JSON files with usernames, follower lists, etc.) never touches the server. Here's what happens where:

| What                    | Where                         | Why                              |
| ----------------------- | ----------------------------- | -------------------------------- |
| ZIP upload + extraction | Browser (JSZip)               | Privacy — no user data on server |
| JSON parsing + analysis | Browser (instagram-parser.ts) | Privacy — no user data on server |
| PDF report generation   | Browser (jsPDF)               | Privacy — no user data on server |
| Analysis history        | Browser (localStorage)        | Privacy — no user data on server |
| Authentication          | Server (Clerk)                | Needed for subscription gating   |
| Payment processing      | Server (Stripe)               | Needed for SaaS model            |
| Usage tracking          | Server (API route)            | Needed for metered billing       |

The server knows who you are (via Clerk) and whether you've paid (via Stripe), but it never sees your Instagram data.

---

## Running locally

Prerequisites: Node.js 20+, Clerk account, Stripe account (for subscription features).

```bash
git clone https://github.com/Orlando-Pedrazzoli/followerscan.git

npm install
cp .env.example .env.local
# Fill in:
#   NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY
#   CLERK_SECRET_KEY
#   STRIPE_SECRET_KEY
#   STRIPE_WEBHOOK_SECRET
#   NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY

npm run dev    # Port 3000
```

The app works without Stripe credentials — you just won't be able to test the subscription flow. The analysis features work regardless.

---

## What I'd improve

- **Server-side subscription validation.** Currently the subscription check relies partly on client-side logic via `useSubscription`. A middleware-level check on the `/dashboard` routes would be more secure and harder to bypass.
- **Stripe Customer Portal.** Users can subscribe but managing their subscription (cancel, update payment method) isn't fully wired up. Stripe's Customer Portal would handle this with minimal code.
- **Rate limiting on API routes.** The checkout and usage-tracking endpoints don't have rate limiting. For a production SaaS, this is a gap.
- **Testing.** No automated tests. The Instagram parser in particular would benefit from unit tests — it handles 12+ file formats with different structures, and a regression in parsing could silently break the analysis.
- **Comparison feature.** The block detection algorithm compares current data with previous analyses stored in localStorage. This works but localStorage has size limits and isn't synced across devices. A lightweight database (even just a key-value store) would make the comparison feature more robust for paying users.

---

## License

Private project. Code shared for portfolio purposes. All rights reserved.

---

Built by [Orlando Pedrazzoli](https://www.orlandopedrazzoli.com) — Full Stack Developer, Lisbon.
