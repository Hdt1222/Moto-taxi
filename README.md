# MotoGo — WhatsApp Moto-Taxi Bot

A WhatsApp bot for a motorcycle-taxi (ojek) service. Handles the full flow:

- **Customers**: book a ride, get a fare estimate, track status, cancel.
- **Drivers**: go online/offline, get offered nearby rides, accept/decline,
  update ride progress (arrived → started → completed).
- **Pricing**: automatic fare based on distance (base fare + per-km rate),
  using GPS locations shared in WhatsApp.
- **Matching**: offers the ride to the nearest online driver; if they don't
  respond in time, it automatically offers the next nearest driver.

It connects to WhatsApp using **Baileys**, a lightweight library that talks
to WhatsApp over a websocket directly — no embedded Chrome browser, so it
runs comfortably on small/free hosting. You link it like WhatsApp Web, by
scanning a QR code.

> ⚠️ Note: Baileys is an unofficial library. It's great for getting started
> and small operations, but it is not endorsed by WhatsApp and carries some
> risk of number restrictions if used at high volume. For a production
> business at scale, consider migrating to the official WhatsApp Business
> Cloud API (Meta) or a provider like Twilio — the booking/pricing/matching
> logic in this project can be reused with a different transport layer.

## 1. Requirements

- A place to run Node.js 18+ continuously — either your own computer, or a
  free/cheap cloud host (recommended if you don't have a PC — see below)
- A WhatsApp number dedicated to the bot (use a spare number/SIM, not your
  personal one — it will stay logged in)

## 2. Deploying without a computer (cloud hosting from your phone)

1. Create a free account at **render.com** using your phone's browser.
2. Put this project in a GitHub repository (GitHub's website lets you
   create a repo and upload files/a zip from your phone browser — no PC
   needed).
3. On Render: **New → Web Service** → connect your GitHub repo.
   - Build command: `npm install`
   - Start command: `npm start`
   - Plan: Free
4. Deploy. Open the **Logs** tab in Render (works fine in your phone
   browser) — the QR code will print there.
5. On the phone with your WhatsApp Business number: **Settings → Linked
   Devices → Link a Device**, and scan the QR code shown in the logs.
6. Once linked, the logs will show `✅ WhatsApp bot is ready.`

**Important limitation:** Render's free tier can sleep after inactivity
and doesn't guarantee the filesystem persists across every restart, so you
may occasionally need to re-scan the QR code. For a real, reliably-running
taxi business, budget for a small paid tier (Render's cheapest paid plan,
or an equivalent ~$5–7/month host) once you're past testing — the code
doesn't need to change, just the hosting plan.

## 3. Running on your own computer instead

```bash
npm install
cp .env.example .env
```

Edit `.env` to set your currency, fare formula, and admin numbers. Then:

```bash
npm start
```

A QR code will print in the terminal. Open WhatsApp on the bot's phone →
**Settings → Linked Devices → Link a Device**, and scan it. Once linked,
you'll see `✅ WhatsApp bot is ready.` and the bot is live.

Session credentials are saved locally (in the `baileys_auth/` folder), so
you won't need to re-scan on every restart, as long as that folder isn't
deleted.

## 4. Add drivers

Drivers must be registered before they can go online:

```bash
npm run add-driver -- 6281234567890 "Budi Santoso" "B 1234 XYZ"
```

(Use the driver's WhatsApp number in international format, no `+` or
leading `0`.) The driver then messages the bot's number and replies
`online` to start receiving ride offers. They should also share their
location (📎 → Location) once so the matcher knows where they are.

## 5. How a ride flows

**Customer:**
1. Sends `book`
2. Shares pickup location (or types an address)
3. Shares destination location (or types an address)
4. Confirms with `yes` — sees fare estimate
5. Bot finds the nearest online driver and offers them the ride
6. Gets notified as the driver accepts, arrives, starts, and completes the
   ride

**Driver:**
1. Replies `online` to start receiving offers
2. On a new ride offer, replies `1` to accept or `2` to decline
3. Replies `arrived`, then `start`, then `done` as the ride progresses

Other commands: `status`, `cancel`, `menu`, `offline`.

## 6. Project structure

```
src/
  index.js          entry point
  bot.js            WhatsApp client + message router
  db.js             SQLite schema (drivers, rides)
  pricing.js         distance + fare calculation
  sessionStore.js    per-customer conversation state
  driverMatcher.js   finds & offers rides to nearest driver, with timeout/fallback
  customerFlow.js    customer conversation logic
  driverFlow.js      driver conversation logic
  tools/addDriver.js CLI to register a driver
```

Data is stored in a local SQLite file (`taxi.db` by default) — no external
database server needed. Everything (pricing, matching radius/timeout,
currency) is configurable via `.env`.

## 7. Customizing

- **Pricing**: edit `BASE_FARE`, `PER_KM_RATE`, `MINIMUM_FARE` in `.env`.
- **Matching timeout**: `DRIVER_OFFER_TIMEOUT_MIN` controls how long a
  driver has to respond before the next nearest driver is offered.
- **Messages/wording**: edit the strings in `customerFlow.js` /
  `driverFlow.js` directly — e.g. translate to Bahasa Indonesia, add
  emojis, change the "MotoGo" brand name.
- **Ratings, payments, receipts, an admin dashboard**: the `rides` and
  `drivers` tables in `db.js` are a natural place to extend from.

## 8. Known limitations to be aware of

- Single-process, single WhatsApp number — fine for one bot number
  serving many customers/drivers, but won't horizontally scale without
  moving conversation state (currently in-memory) to something shared
  like Redis.
- No payment processing — fare is calculated and communicated, but
  collecting payment (cash, e-wallet, etc.) happens outside the bot unless
  you add an integration.
- Distance is straight-line (haversine), not real road distance — fine
  for short urban moto-taxi trips, but you could swap in a routing API
  (e.g. OSRM, Google Directions) in `pricing.js` for more accuracy.
