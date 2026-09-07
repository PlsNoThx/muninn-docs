# Deploying Muninn (Render + Porkbun)

The app auto-deploys from GitHub: **push to `main` → Render rebuilds → the live
site updates.** So the normal Claude Code / GitHub loop *is* the deploy pipeline.

## One-time setup

### 1. Commit your data (from the Codespace where it lives)

The deployed server needs your taste profile at runtime. The generated data is
now tracked (private repo), so commit it once:

```bash
git pull
npm run enrich -- --favorites-only   # if the enriched CSV isn't present
npm run profile                       # (re)build the card with current wording
git add data/taste_profile.md data/taste_profile.json data/taste_profile_enriched.csv
git commit -m "Add enriched data and taste profile"
git push
```

From now on, refreshing your taste is just: re-run `npm run profile`, commit,
push — and Render redeploys with the new profile automatically.

### 2. Create the Render service

1. Go to <https://dashboard.render.com> → **New** → **Blueprint**.
2. Connect your GitHub and pick the `Muninn` repo. Render reads `render.yaml`
   and proposes the `muninn` web service — **Apply**.
3. When prompted, set the two secret env vars (they're `sync: false`, so Render
   asks for them and never stores them in the repo):
   - `GOOGLE_PLACES_API_KEY`
   - `ANTHROPIC_API_KEY`
4. First deploy runs `npm install` then `npm run serve`. Watch the logs; when it
   says `Muninn running…`, open the `onrender.com` URL to confirm.

> Free tier sleeps after ~15 min idle, so the first request after a nap takes
> ~30–50s to wake. Upgrade to a paid instance later to remove that.

### 3. Attach the subdomain

1. In the Render service → **Settings → Custom Domains** → add
   `muninn.ianquimby.com`. Render shows a **CNAME target** (like
   `muninn-xxxx.onrender.com`).
2. In **Porkbun** → your `ianquimby.com` domain → **DNS** → add a record:
   - Type: **CNAME**
   - Host: `muninn`
   - Answer: the Render CNAME target
3. Wait for DNS to propagate (minutes to ~an hour). Render auto-issues HTTPS.
   The apex `ianquimby.com` stays on Framer, untouched.

## Add-a-place storage (Supabase)

The "Add" tab needs a database (Render's disk is ephemeral). One-time setup:

1. Create a free project at <https://supabase.com>.
2. In the **SQL Editor**, run:
   ```sql
   create table added_places (
     id uuid primary key default gen_random_uuid(),
     created_at timestamptz default now(),
     place_id text, name text, primary_type text, types text,
     price_level int, rating real, user_rating_count int,
     address text, lat double precision, lng double precision, note text
   );
   ```
3. In **Project Settings → API**, copy the **Project URL** and the
   **`service_role`** key (secret — server-side only).
4. Add them as env vars (Render dashboard, and your local `.env`):
   `SUPABASE_URL`, `SUPABASE_SERVICE_KEY`.

Until these are set, the app runs fine — the Add tab just reports that storage
isn't configured. Added places merge into your saved list (recommendations,
Near me) and their notes feed the ranking.

## The ongoing loop

```
edit in Claude Code → push to main → Render auto-deploys → muninn.ianquimby.com
```

- Secrets live only in Render's dashboard and your local `.env` (gitignored).
- Optional: enable **PR Previews** in Render so each pull request gets its own
  temporary URL to check before merging.
