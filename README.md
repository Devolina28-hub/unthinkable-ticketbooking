# Encore — Ticket Booking System

Full-stack ticket booking app: visual seat maps, TTL seat holds that auto-release on
abandonment, a waitlist with automatic re-assignment on cancellation, QR-code tickets,
and a mock "Scan to Pay" checkout.

**Live app/Hosted Application URL:** https://ticket-booking-system-delta-six.vercel.app
(It will take 5-10 seconds to get load please kindly wait for 5 to 10 seconds)

- **Backend:** Node.js, Express, PostgreSQL (Neon, via `pg`), JWT auth
- **Frontend:** React 18 + Vite
- **Email:** Brevo transactional email HTTP API (not SMTP)
- **QR codes:** `qrcode` npm package, shown in-app and emailed

---

## ⚠️ Two things to know before testing

1. **Use your own email to receive anything.** Seeded demo accounts
   (`customer@example.com` etc.) aren't real inboxes. Register at `/register/customer`
   with your real email, then book from that account. If `BREVO_API_KEY` isn't set,
   email sending is skipped entirely (logged, not silent).
2. **Payment is a mock, not a real gateway.** "Continue to payment" shows a QR linking
   to `/pay/:paymentRef` — open it (or scan it) and tap **Yes, pay now** / **No, cancel**.
   Yes waits ~5s (simulated processing) before confirming the booking + emailing the
   ticket. No card details are ever collected; no money moves.

---

## 1. Setup

**Prerequisites:** Node.js 18+, npm, a free Neon Postgres DB.

**Database:** sign up at https://neon.tech → create a project → copy the connection
string (`postgresql://user:pass@...neon.tech/dbname?sslmode=require`).

**Backend:**
```bash
cd backend
cp .env.example .env      # paste DATABASE_URL (Neon) + BREVO_API_KEY etc.
npm install
npm run seed               # demo users, venues, events
npm start                  # http://localhost:4000
```
Tables auto-create on boot (`ensureSchema()`) — no manual migrations.

Demo accounts (password `password123` for all): `admin@example.com`,
`organiser@example.com` / `nina@example.com`, `customer@example.com` / `customer2@example.com`.

**Frontend:**
```bash
cd frontend
npm install
npm run dev                # http://localhost:5173
```
Three login portals: `/login/customer`, `/login/organiser`, `/login/admin` (admin is
seed-only, no public sign-up). `npm run build` for production; point `src/api.js`'s
`BASE` at your deployed backend URL.

### Real email delivery (Brevo API key — not SMTP)

Emails send via Brevo's HTTP API (`api.brevo.com/v3/smtp/email`), not SMTP — most free
hosts block SMTP ports, but HTTPS (443) always works.

1. Sign up free at brevo.com.
2. **Settings → SMTP & API → API Keys** → generate a key → this is `BREVO_API_KEY`.
3. **Settings → Senders, Domains & Dedicated IPs → Senders → Add a sender** → verify it
   via the email Brevo sends. Mandatory — unverified senders get rejected. Put this
   address in `SMTP_FROM` (name kept from the old SMTP setup, but it's just the "from"
   address used with the API now).
4. **Settings → Security → Authorized IPs** → activate/authorize both keys listed there
   so your deployed backend's requests aren't blocked. Authorize all IPs if unsure, then
   tighten later.
5. Restart backend — startup log shows `Brevo API key OK` or the exact failure reason.
6. Set `FRONTEND_URL` to your real deployed frontend URL — it's embedded in QR codes and
   waitlist-offer email links; left as `localhost` it only works on your machine.

If `BREVO_API_KEY` is blank, email sending is skipped (with a log warning) and the rest
of the booking flow still works.

---

## 2. Environment variables

| Variable | Purpose | Default |
|---|---|---|
| `PORT` | API port | `4000` |
| `JWT_SECRET` | Auth token signing secret | *(change in production)* |
| `JWT_EXPIRES_IN` | Token lifetime | `7d` |
| `DATABASE_URL` | Neon Postgres connection string | *(required)* |
| `SEAT_HOLD_TTL_MINUTES` | Seat hold duration before auto-release | `10` |
| `WAITLIST_OFFER_TTL_MINUTES` | Waitlist offer validity | `15` |
| `PAYMENT_QR_TTL_MINUTES` | Mock payment session validity | `10` |
| `HOLD_SWEEPER_INTERVAL_SECONDS` | Background sweeper frequency | `30` |
| `BREVO_API_KEY` | Brevo email API key | *(blank → email skipped)* |
| `SMTP_FROM` | "From" name/address for Brevo API | `"Encore Tickets <no-reply@yourdomain.com>"` |
| `FRONTEND_URL` | Used in QR codes + offer email links | `http://localhost:5173` |

---

## 3. Database schema

```
users            id, name, email, password_hash, role(admin|organiser|customer)
venues           id, name, address, created_by
venue_seats      id, venue_id, row_label, seat_number, category        -- physical layout
events           id, title, description, type, venue_id, organiser_id, event_date, event_time
event_pricing    id, event_id, category, price
event_seats      id, event_id, venue_seat_id, row_label, seat_number, category,
                 status(available|held|offered|booked), held_by, hold_expires_at, booking_id
bookings         id, booking_ref, event_id, customer_id, status(confirmed|cancelled),
                 total_amount, qr_data_url, created_at, cancelled_at
booking_seats    id, booking_id, event_seat_id
payment_sessions id, payment_ref, event_id, customer_id, seat_ids[], amount,
                 status(pending|approved|declined|expired), qr_data_url, booking_id,
                 expires_at, decided_at                                 -- mock "scan to pay"
waitlist         id, event_id, category, customer_id,
                 status(waiting|offered|expired|booked|cancelled),
                 offered_seat_id, offer_expires_at, joined_at
```

`event_seats` is a **per-show snapshot** of `venue_seats` — each event gets an
independent seat map, even at the same venue. A partial unique index caps each
customer to one active waitlist entry per event+category.

---

## 4. Seat hold & TTL

1. `POST /api/events/:eventId/seats/hold { seat_ids }` claims seats via a **conditional
   UPDATE**: `... SET status='held' ... WHERE status='available' RETURNING id`. Zero
   rows back → seat wasn't free → request fails. All seats in one transaction, so it's
   all-or-nothing (no partial holds).
2. `hold_expires_at = now + SEAT_HOLD_TTL_MINUTES`.
3. A cron sweeper (every `HOLD_SWEEPER_INTERVAL_SECONDS`) releases expired holds back to
   `available`; the same check also runs on every seat-map read, so the UI never shows a
   stale hold between sweeper ticks.
4. Frontend polls every 4s and shows a live countdown, but the server is always the
   final authority on confirm.

## 5. Concurrency protection

Every state change (hold, confirm, cancel, waitlist-offer) uses the same **conditional
UPDATE ... WHERE status = 'expected' RETURNING id** pattern instead of read-then-write.
Postgres serializes concurrent updates to the same row — the second of two racing
requests always finds zero matching rows and gets a clean `409 Conflict`. Verified with
a genuine `Promise.all` simultaneous-request test: exactly one `200`, one `409`, no
double-booking.

## 6. Waitlist auto-assignment & time-limited offers

- `POST /api/waitlist { event_id, category }` joins a FIFO queue, ordered by `joined_at`.
- On cancellation or offer expiry, `offerSeatToNextInLine()` picks the oldest `waiting`
  entry, flips the seat `available → offered`, sets `hold_expires_at = now +
  WAITLIST_OFFER_TTL_MINUTES`, and emails a time-limited link.
- Customer calls `POST /api/waitlist/:id/complete` before it expires to convert the
  offer into a real booking.
- If unused, the sweeper marks it `expired`, releases the seat, and immediately
  re-offers it to the next person in line — cascading automatically.

## 7. Mock payment flow (Scan to Pay)

No real gateway — no Stripe/Razorpay key anywhere in the code, no card form.

1. `POST /api/payments/initiate` extends the seat holds to match the payment window,
   computes the total, and creates a `payment_sessions` row (`pending`).
2. A QR pointing at `/pay/:paymentRef` is shown; the checkout tab polls
   `GET /api/payments/:paymentRef/status`.
3. Opening/scanning that link shows the amount and two buttons: **Yes, pay now** /
   **No, cancel**.
4. **Yes** → `POST /api/payments/:paymentRef/decision { approve: true }`. The frontend
   waits ~5s (simulated processing) before showing the result, while the backend
   converts the held seats into a `booked` booking and emails the confirmation.
5. **No**, or letting the ~10-min window lapse, releases the seats — no booking created.
6. The original checkout tab picks up the final status via polling.

## 8. QR codes, ticket verification & email

- `generateQrDataUrl(bookingRef)` encodes `${FRONTEND_URL}/ticket/:bookingRef` as a PNG
  — scanning opens a real page, not raw text. **Set `FRONTEND_URL` to your real deployed
  URL** or QR codes will only work on your own machine.
- `/ticket/:bookingRef` (public, no login — venue staff scanning aren't logged in) calls
  `GET /api/bookings/verify/:bookingRef` and shows: ✅ valid, ⛔ cancelled, or ❓ not found.
- Confirmation emails embed the QR as a CID attachment, sent via Brevo's API (see §1).

---

## 9. API reference

`Authorization: Bearer <jwt>` required unless marked public. Roles in brackets.

**Auth** — `POST /auth/register {name,email,password,role?}` (customer/organiser only) ·
`POST /auth/login` · `GET /auth/me`

**Venues** — `GET /venues` · `GET /venues/:id` · `POST /venues` [admin]
`{name,address,layout:[{row_label,seats,category}]}`

**Events** — `GET /events?type=&date=&q=` · `GET /events/:id` ·
`POST /events` [organiser,admin] · `GET /events/:id/summary` [organiser own / admin any]

**Seats** — `GET /events/:eventId/seats` ·
`POST /events/:eventId/seats/hold` [customer] `{seat_ids}` (max 4/booking, frontend-enforced) ·
`POST /events/:eventId/seats/release` [customer]

**Bookings** — `POST /bookings/confirm` [customer] `{event_id,seat_ids}` ·
`GET /bookings/my` [customer] · `POST /bookings/:id/cancel` [customer own, admin] ·
`GET /bookings/verify/:bookingRef` [public]

**Waitlist** — `POST /waitlist` [customer] `{event_id,category}` ·
`GET /waitlist/my` [customer] · `POST /waitlist/:id/complete` [customer own]

**Payments** — `POST /payments/initiate` [customer] `{event_id,seat_ids}` ·
`GET /payments/:paymentRef/status` [public] ·
`POST /payments/:paymentRef/decision` [public] `{approve}`

---

## 10. Project structure

```
backend/src/
  db/index.js, db/seed.js       schema + demo data
  auth.js                        JWT middleware
  routes/                        auth, venues, events, seats, bookings, waitlist, payments
  services/
    holdSweeper.js                TTL release + offer expiry cron
    waitlistService.js            FIFO auto-assignment
    qr.js                          QR generation
    email.js                       Brevo HTTP API
  server.js
frontend/src/
  pages/                          Events, EventSeatMap, Login, Register, PayConfirm,
                                  BookingHistory, MyWaitlist, OrganiserDashboard, AdminVenues
  components/                     Navbar, SeatGrid
  api.js                          fetch wrapper
```

## 11. Deploying

- **DB:** Neon (free) → `DATABASE_URL` in your host's env vars. Persists across restarts.
- **Backend:** Render/Railway/Fly.io → set `DATABASE_URL`, `JWT_SECRET`, `BREVO_API_KEY`,
  `SMTP_FROM`, `FRONTEND_URL`.
- **Frontend:** `npm run build` → deploy `dist/` to Vercel/Netlify → point `src/api.js`'s
  `BASE` at your backend URL.

