# BudgetFLOW — Project Structure & Stack Reference

Internal package name: `paisa`. An AI-powered personal finance app for the Indian market (all amounts in ₹ INR).

## 1. Tech Stack

| Layer | Choice |
|---|---|
| Framework | Next.js 16 (App Router, React Server Components) |
| UI library | React 19 |
| Language | JavaScript/JSX (only `prisma.config.ts` is TypeScript) |
| Styling | Tailwind CSS v4 + shadcn/ui ("new-york" style) |
| Animation | Framer Motion (`motion`), Lenis (smooth scroll), Lottie |
| Components | Radix UI primitives, Vaul (drawers) |
| Charts | Recharts |
| Forms | react-hook-form + zod |
| ORM | Prisma 7 (`@prisma/adapter-pg` + `pg` driver) |
| Database | PostgreSQL |
| Auth | Clerk |
| Background jobs | Inngest (cron + event-driven) |
| Security | Arcjet (bot detection, shield, rate limiting) |
| AI | Google Gemini (`gemini-2.5-flash`) |
| Email | Brevo (active) — `resend` present but unused |
| PDF generation | `@react-pdf/renderer` |
| Package manager | pnpm |

## 2. External Services (env vars, see `.env`)

- **Clerk** — `CLERK_SECRET_KEY`, `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`
- **PostgreSQL** — `DATABASE_URL`, `DIRECT_URL`
- **Arcjet** — `ARCJET_KEY`
- **Gemini** — `GEMINI_API_KEY`
- **Brevo** — `BREVO_API_KEY`, `BREVO_SENDER_EMAIL`, `BREVO_SENDER_NAME`
- **Inngest** — `INNGEST_EVENT_KEY`, `INNGEST_SIGNING_KEY`, `INNGEST_DEV`
- **VAPID** (web push) — `NEXT_PUBLIC_VAPID_PUBLIC_KEY`, `VAPID_PRIVATE_KEY`, `VAPID_SUBJECT` — **provisioned, not implemented** (no push code in the repo)

## 3. Folder Structure

```
app/
  (auth)/                     Clerk sign-in / sign-up pages
  (main)/                     Authenticated app shell
    dashboard/                 Main dashboard (accounts, budget, overview)
    account/[id]/               Single account detail + transaction table
    transaction/create/         Add/edit transaction + AI receipt scanner
    advisor/                    AI financial advisor report list + detail
    groups/                     Splitwise-style bill splitting
  api/
    advisor/[reportId]/pdf/     Advisor report PDF download
    dashboard/transactions-pdf/ Transaction export PDF
    inngest/                     Inngest webhook/serve endpoint
    seed/                        Demo data seeding
  layout.js, page.js             Root layout + landing page
  manifest.js                    Dynamic PWA manifest
  icon.js / apple-icon.js        Dynamically generated app icons

actions/                       Server actions (account, transaction, budget,
                                 advisor, groups, seed, send-email)
lib/
  prisma.js                     Prisma client singleton
  checkUser.js                  Clerk <-> DB user sync
  arcjet.js                     Arcjet client config
  utils.js                      cn() + formatINR()
  bill-formatter.js             Group bill/summary text generator
  advisor/                       buildPayload / generateReport / renderPdf
  dashboard/renderTransactionsPdf.js
  inngest/                        client.js, function.js (all cron/event jobs)

components/                    Shared UI: header, footer, hero, onboarding
  ui/                           shadcn/ui primitives + custom icon set

hooks/use-fetch.js             Generic client hook for server actions
data/                          categories.js, landing.js (static content)
emails/template.jsx            Single React Email template, multi-purpose
prisma/schema.prisma           Database schema
public/sw.js                   Minimal PWA service worker
```

## 4. Database Schema (PostgreSQL via Prisma)

| Model | Purpose |
|---|---|
| `User` | Mirrors Clerk identity (`clerkUserId`, `email` unique) |
| `Account` | Bank/wallet account, running `balance`, `type` (CURRENT/SAVINGS) |
| `Transaction` | Income/expense entry; supports recurring fields |
| `Budget` | One monthly budget cap per user + alert dedupe timestamp |
| `Group` | Bill-splitting group |
| `GroupMember` | Non-authenticated participant (name + email only) |
| `GroupExpense` | A shared expense within a group |
| `ExpenseShare` | Per-member portion of an expense, settlement tracking |
| `FinancialReport` | Persisted AI advisor report (markdown + health score) |

Enums: `TransactionType`, `AccountType`, `TransactionStatus`, `RecurringInterval`.

All money fields use Prisma `Decimal`. Cascading deletes throughout. Foreign keys indexed.

## 5. Core Features

1. Multi-account tracking with auto-maintained balances
2. Transaction management (income/expense, categories, receipts)
3. Recurring transactions (daily/weekly/monthly/yearly, cron-driven)
4. AI receipt scanning (Gemini OCR/parsing)
5. Monthly budgeting with threshold alerts
6. Dashboard analytics (category breakdown, PDF export)
7. Account charts (income vs expense over time)
8. Monthly AI-generated email reports
9. AI Financial Advisor (18-section deep report)
10. Splitwise-style group expense splitting (no login needed for members)
11. Bill export + email settlement reminders
12. Onboarding tour (7-slide walkthrough)
13. PWA install support
14. Landing/marketing site
15. Demo data seeding

## 6. Distinctive Design Choices

- 18-section AI Financial Advisor with health score, forecasts, and an India-specific wealth roadmap (SIP/ELSS/PPF/NPS/80C)
- Hand-rolled markdown → PDF renderer (no third-party markdown-to-PDF lib)
- Delayed, self-correcting budget alerts (sleep-then-recheck pattern via Inngest)
- India-first throughout: INR formatting, lakh/crore numbering, Indian tax/investment terms
- No-login group members for bill splitting
- Layered rate limiting: Arcjet (edge + per-action) + app-level throttles (24h advisor, 5min email reminders)
- VAPID env vars present but web push is not implemented — scaffolded for a future feature

## 7. Known Housekeeping Items

- Stray debug files at repo root: `debug_inngest.js`, `debug_inngest2.js`, `inngest_debug.txt`, `test_db_connection.js`, `test_prisma.js`, `out3.json`, `test_out.txt`, `test_out2.txt`
- `lib/inngest/function.js` — `notifyBudgetThreshold`'s sleep is currently `"1m"` with a comment `// TESTING: was "3h". Revert before commit.` — should be reverted to 3h before shipping
- `resend` dependency appears unused (Brevo is the active email provider)
- Both `pnpm-lock.yaml` and `package-lock.json` exist — pick one and remove the other
