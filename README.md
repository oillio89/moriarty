# Moriarty — Sync & Deploy Setup

Moriarty works fully offline with no setup (data saves to your browser's local
storage). This guide is for the **optional** upgrade: syncing your data across
devices via a free Supabase project, then deploying the app to GitHub Pages.

If you skip this guide entirely, the app still works exactly as before —
locally, in whatever browser you open it in.

---

## 1. Create a free Supabase project

1. Go to [supabase.com](https://supabase.com) → sign up (free tier is enough).
2. Create a new project. Pick any name/region, set a database password
   (you won't need it day-to-day).
3. Wait a minute or two for it to finish provisioning.

## 2. Create the data table

In your Supabase project, open the **SQL Editor** and run:

```sql
create table moriarty_data (
  user_id uuid primary key references auth.users(id) on delete cascade,
  data jsonb not null default '{}'::jsonb,
  updated_at timestamptz not null default now()
);

alter table moriarty_data enable row level security;

create policy "Users can manage their own data"
on moriarty_data
for all
using (auth.uid() = user_id)
with check (auth.uid() = user_id);
```

This creates one row per signed-in user, holding their entire Moriarty data
as a single JSON blob, and locks it down so people can only ever read or
write their own row.

## 3. Get your API keys

In your Supabase project: **Settings → API**. You need two values:

- **Project URL** (looks like `https://xxxxx.supabase.co`)
- **anon public** key (a long string starting with `eyJ...`)

## 4. Configure the app

Open `moriarty.html`, find this block near the top of the `<script>` section:

```js
const SUPABASE_URL = 'YOUR_SUPABASE_URL';
const SUPABASE_ANON_KEY = 'YOUR_SUPABASE_ANON_KEY';
```

Replace both placeholder strings with your real values from step 3, and save.
That's the only code change needed — everything else detects this
automatically and switches from local-only to synced mode.

## 5. Allow your deployed URL to redirect back

In Supabase: **Authentication → URL Configuration**, add your future GitHub
Pages URL (e.g. `https://yourname.github.io/moriarty/`) — and, for testing,
`http://localhost:5500` or whatever you use locally — to **Redirect URLs**.
This is what lets the magic-link email bring people back to the right page.

## 6. Deploy to GitHub Pages

1. Create a new GitHub repo, push `moriarty.html` into it (rename it to
   `index.html` for the cleanest URL).
2. Repo → **Settings → Pages** → set source to your default branch, root
   folder.
3. GitHub gives you a URL like `https://yourname.github.io/reponame/`.
   Make sure that exact URL is in the Supabase Redirect URLs list from step 5.

## How it behaves

- **No Supabase config (default):** works exactly as before, fully local,
  no sign-in screen.
- **Supabase configured, not signed in:** shows a simple "sign in with
  email" screen, with a "Continue without an account" option that keeps
  using local storage only.
- **Signed in:** data syncs to your Supabase project in the background
  (about a second after each change) and loads from there on any device
  where you sign in with the same email.

## Note on the claude.ai preview link

The published claude.ai artifact link can load the Supabase script, but its
sandbox blocks network calls to external APIs like `*.supabase.co` — so sign-in
and sync will silently fail there even after you fill in your keys. This is
expected. Once the file is deployed on GitHub Pages (a normal website with
regular internet access), sync works as described above.

---

## Live news + Thai translation

The News pillar ships with a fixed snapshot of headlines. There's also a
"Refresh with live news" button that pulls real current headlines and
translates them to Thai on demand — using free services, no signup and no
server needed. Like Supabase, **this only works once deployed outside
claude.ai** (its sandbox blocks these calls too).

**Nothing to configure** — it works out of the box once deployed. It uses:

- **[rss2json.com](https://rss2json.com)** to pull BBC World News' RSS feed
  as JSON. This service explicitly supports CORS from any real domain on its
  free tier (some other news APIs, GNews included, restrict their free tier's
  CORS to `localhost` only — meaning they work while testing locally but
  silently fail once deployed; rss2json doesn't have that restriction).
- **[MyMemory](https://mymemory.translated.net/)** for on-demand Thai
  translation — free, no signup or key needed.

### How it behaves

- Tapping **"🔄 Refresh with live news"** fetches 5 current BBC World
  headlines, cached in the browser so they survive a reload until refreshed
  again.
- Tapping **"แปลไทย"** on a live headline translates it on the spot (the
  static snapshot headlines already have hand-written Thai, so that's
  instant).
- If the fetch fails for any reason (offline, feed down, etc.), the app
  shows a toast and falls back to the static snapshot — it never breaks the
  rest of the app.

### Want a different news source?

Swap the feed URL in the code:

```js
const RSS_FEED_URL = 'https://feeds.bbci.co.uk/news/rss.xml';
```

Any RSS feed works — try your favorite outlet's world/top-stories feed.
rss2json's free tier has a modest daily request limit; if you hit it, an
optional free API key from rss2json.com raises the limit.

Thai translation uses MyMemory (free, no signup needed). Quality is decent
but can be rough on longer, more complex sentences — that's a known
trade-off of using a free, no-backend translation service.
