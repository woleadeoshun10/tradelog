# TradeLog
 
A self-hosted trading journal web app for HeroFX TradeLocker accounts. Single HTML file — no build step, no backend, no dependencies to install.
 
---
 
## Features
 
- **Auto-sync with TradeLocker** — connects directly to HeroFX demo and live accounts via the TradeLocker API, imports closed positions automatically
- **Multi-account switching** — switch between any of your TradeLocker accounts from a dropdown, clears and re-imports on switch
- **Cloud auth via Supabase** — sign up with email, trades and journals sync to the cloud so you can access from any device or share with teammates
- **Trade log** — full table of all trades with date, pair, direction, session, entry/exit, P&L, R:R, source, and quick delete
- **Calendar view** — monthly calendar with green/red borders per day, P&L displayed per cell, click any day to see breakdown
- **Dashboard** — P&L bar chart by period, win/loss/BE breakdown with progress bar, session distribution (London/NY/Asia/Overlap), best/worst day, top mistakes
- **Weekly view** — week-by-week breakdown with win rate and P&L per week
- **Trade journals** — per-trade journal entries with setup notes, execution, lessons, confidence rating, emotional state, R:R, strategy tag, and mistake tags
- **Timeframe filters** — 1D / 1W / 1M / 3M / 6M / 1Y / ALL + custom date range
- **CSV export** — export filtered trades to CSV
- **Dark / light mode** — persisted to localStorage
- **Responsive** — works on iPhone, iPad, and desktop
---
 
## Stack
 
| Layer | Tech |
|---|---|
| Frontend | Vanilla HTML/CSS/JS — single file, no framework |
| Auth & DB | [Supabase](https://supabase.com) (email auth + PostgreSQL) |
| Trade data | [TradeLocker API](https://tradelocker.com) via HeroFX |
| Hosting | None required — open the HTML file directly in any browser |
 
---
 
## Setup
 
### 1. Supabase
 
1. Create a free project at [supabase.com](https://supabase.com)
2. Go to **SQL Editor** and run the setup script:
```sql
-- TRADES TABLE
create table if not exists trades (
  id text not null,
  user_id uuid references auth.users(id) on delete cascade,
  source text, acct text, date text, time text, pair text, dir text,
  session text, entry text, exit text, sl text, tp text,
  lots numeric, pnl text, result text, rr text, strategy text,
  notes text, mistakes jsonb, tl_order_id text,
  updated_at timestamptz default now(),
  primary key (id)
);
 
-- JOURNALS TABLE (with screenshot URLs)
create table if not exists journals (
  trade_id text not null,
  user_id uuid references auth.users(id) on delete cascade,
  rr text, strategy text, mistakes jsonb, setup text,
  execution text, lessons text, confidence text, emotion text,
  image_urls jsonb default '[]'::jsonb,
  saved_at timestamptz, updated_at timestamptz default now(),
  primary key (trade_id, user_id)
);
 
-- Add screenshot column if journals table already exists
alter table journals add column if not exists image_urls jsonb default '[]'::jsonb;
 
-- ROW LEVEL SECURITY
alter table trades enable row level security;
alter table journals enable row level security;
 
drop policy if exists "Users own trades" on trades;
drop policy if exists "Users own journals" on journals;
 
create policy "Users own trades" on trades
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
 
create policy "Users own journals" on journals
  for all using (auth.uid() = user_id) with check (auth.uid() = user_id);
 
-- STORAGE BUCKET FOR JOURNAL SCREENSHOTS
insert into storage.buckets (id, name, public)
values ('journal-images', 'journal-images', true)
on conflict (id) do update set public = true;
 
drop policy if exists "Users upload journal images" on storage.objects;
drop policy if exists "Users view journal images" on storage.objects;
drop policy if exists "Users update journal images" on storage.objects;
drop policy if exists "Users delete journal images" on storage.objects;
 
create policy "Users upload journal images" on storage.objects
  for insert with check (
    bucket_id = 'journal-images'
    and auth.uid()::text = (storage.foldername(name))[1]
  );
 
create policy "Users view journal images" on storage.objects
  for select using (
    bucket_id = 'journal-images'
    and auth.uid()::text = (storage.foldername(name))[1]
  );
 
create policy "Users update journal images" on storage.objects
  for update using (
    bucket_id = 'journal-images'
    and auth.uid()::text = (storage.foldername(name))[1]
  ) with check (
    bucket_id = 'journal-images'
    and auth.uid()::text = (storage.foldername(name))[1]
  );
 
create policy "Users delete journal images" on storage.objects
  for delete using (
    bucket_id = 'journal-images'
    and auth.uid()::text = (storage.foldername(name))[1]
  );
```
 
3. Copy your **Project URL** and **anon/public key** from **Settings → API**
4. Replace `SB_URL` and `SB_KEY` at the top of `trading-journal-v2.html`
### 2. Open the app
 
Open `trading-journal-v2.html` directly in Chrome. No server needed.
 
### 3. Sign up
 
Click **Sign Up**, enter your email and password. Confirm your email, then sign in.
 
### 4. Connect TradeLocker
 
Go to the **⚡ Sync** tab, enter your HeroFX email and TradeLocker password under Demo or Live, and click **Connect & Sync**. Your accounts will appear in a dropdown — select whichever account you want to track.
 
---
 
## P&L Calculation
 
TradeLocker's order history API does not return P&L values directly. P&L is calculated as:
 
```
P&L = (exitPrice - entryPrice) × lots × contractSize   (LONG)
P&L = (entryPrice - exitPrice) × lots × contractSize   (SHORT)
```
 
Contract sizes used:
 
| Instrument | Contract Size |
|---|---|
| XAUUSD / XAGUSD | 100 |
| BTCUSD / ETH / crypto | 1 |
| NAS100 / indices | 100 |
| OIL / WTI / BRENT | 100 |
| Forex | 100 |
 
Each position is split into individual trades by pairing each open fill with each close fill, matching TradeLocker's per-trade display.
 
---
 
## Multi-user
 
Anyone can open the same HTML file, create their own Supabase account, and see only their own data. Row Level Security ensures complete data isolation between users.
 
---
 
## File Structure
 
```
trading-journal-v2.html   — entire app (HTML + CSS + JS, ~1600 lines)
supabase-setup.sql        — database setup script
README.md                 — this file
```
 
---
 
## License
 
MIT
 
