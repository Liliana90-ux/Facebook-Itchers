ReelScope

ReelScope is a private Facebook Reels analytics and scheduling dashboard for one owner and the Facebook Pages they administer. It stores daily post-level snapshots so results remain available after Meta's recent-insights window moves on.

It is deliberately not a SaaS product: there are no user accounts, competitor tools, tag scores, or public onboarding flows.

What is included

Facebook Login OAuth with CSRF state checking

Long-lived user-token exchange and encrypted Page-token storage

Proactive refresh attempts when the tracked token lifetime has seven days left

A visible reconnect state for expired, revoked, or otherwise invalid tokens

Daily/manual insight snapshots persisted in Cloudflare D1

Views, average watch time, reach, engagement rate, optional retention, and comparison with each Page's own historical average

Best-time recommendations computed only from the connected Pages' post history

Native Reels upload and scheduling through Meta's video_reels start → upload → finish flow

Rate-limit and transient-error retry with exponential backoff

A single Meta adapter at lib/meta-graph.ts, including the API version, fields, insight metric names, OAuth, retry logic, and publishing calls

Prerequisites

Node.js 22.13 or newer

A Meta developer account

A Meta app in Development mode

Your Facebook account must be an app administrator/developer/tester and must administer the two target Pages

1. Configure the Meta app

Create an app in the Meta App Dashboard. A Business app type/use case is the most natural fit for Page management.

Add Facebook Login to the app.

In Facebook Login settings, add this exact Valid OAuth Redirect URI:

http://localhost:5173/api/auth/facebook/callback

Keep the app in Development mode. Add your own Facebook account under App roles if it is not already an administrator.

In the Graph API/permissions configuration, make these permissions available to your development-role account:

Permission

Why ReelScope needs it

pages_show_list

Lists the Pages your Facebook account manages and obtains their Page tokens.

pages_read_engagement

Reads Page posts/Reels and engagement-visible Page content.

read_insights

Reads Page and post-level insight metrics.

pages_manage_posts

Uploads and schedules a Reel on your Pages.

Meta's official references: Page access tokens, Insights edge, Page Video Reels edge, and Graph API error handling.

Development-mode access is enough for the app's administrator/developer/tester accounts. Do not submit the app for review just to use this privately with your own Pages.

2. Configure local secrets

Copy the example secrets file:

cp .dev.vars.example .dev.vars

Generate a 32-byte AES key for token encryption:

node -e "console.log(require('crypto').randomBytes(32).toString('base64'))"

Generate a separate random value for CRON_SECRET, then fill in .dev.vars:

FB_APP_ID=your_meta_app_id
FB_APP_SECRET=your_meta_app_secret
FB_REDIRECT_URI=http://localhost:5173/api/auth/facebook/callback
FB_GRAPH_API_VERSION=v26.0
FB_ALLOWED_PAGE_IDS=123456789012345,987654321098765
TOKEN_ENCRYPTION_KEY=the_generated_base64_key
CRON_SECRET=a_different_long_random_value
DASHBOARD_TIME_ZONE=Asia/Dhaka
DASHBOARD_URL=http://localhost:5173

FB_ALLOWED_PAGE_IDS is strongly recommended. Set it to the tech Page ID and meme/shorts Page ID so the app stores only those two Pages even if your Facebook account administers others.

.dev.vars, .env, local databases, build output, and Worker state are ignored by Git. Never commit real app secrets or tokens.

3. Install and run

npm install
npm run dev

Open http://localhost:5173. Until Facebook is connected, the interface shows clearly labelled sample data so the dashboard can be explored.

Click Connect Facebook, approve the four permissions, then click Sync now. The first sync creates local tables automatically and records the first historical snapshot. Subsequent daily syncs update that day's snapshot without deleting older days.

Useful checks:

npm run lint
npm run build

The health endpoint at GET /api/health reports whether the database and required configuration are available; it never returns secret values.

4. Run the daily historical sync

The app exposes POST /api/cron/sync, protected by CRON_SECRET. The development server must be running when the local scheduled command fires.

Test it manually:

npm run sync:daily

Linux/macOS cron example for 3:10 AM daily:

10 3 * * * cd /absolute/path/to/reelscope-facebook-analytics && /usr/bin/npm run sync:daily >> /tmp/reelscope-sync.log 2>&1

On Windows, create a Task Scheduler task that runs npm run sync:daily in this project directory once per day. Use a time when the local app normally remains running.

Daily collection matters: Meta does not promise unlimited historical availability for every Page/post insight. ReelScope persists every collected snapshot locally and does not attempt to reconstruct missing past days.

Token lifecycle

Facebook Login first returns a short-lived user token. ReelScope exchanges it for a long-lived user token, fetches Page access tokens from /me/accounts, encrypts all stored tokens with AES-GCM, and records the returned expiry.

Before a sync or scheduled upload, the app checks token health. When the tracked long-lived lifetime has seven days left, it attempts another server-side exchange and refreshes the Page tokens. Page tokens derived from a long-lived user token can be non-expiring in normal conditions, but can still become invalid when the owner changes a password, removes the app, loses a Page role, or Meta revokes access. A Graph error with OAuth code 190 marks every affected Page as reauth_required and the dashboard shows an explicit Reconnect prompt; it never fails silently.

Meta may require interactive login instead of accepting a refresh attempt. That is expected and is why the reconnect path is part of the product.

Insight calculations

The Meta adapter currently requests:

post_video_views

post_video_avg_time_watched

post_impressions_unique as reach

post_engaged_users

post_video_retention_graph when Meta exposes it for that Page/post/API version

Engagement rate is engaged users ÷ reach × 100. A Reel's comparison is against the mean latest-view count for all captured Reels on that same Page. The best-time score buckets posts by their local publish weekday/hour, normalizes views within each Page, then weights normalized views (60%), engagement rate (25%), and average watch time (15%). The dashboard reports the bucket sample size and lowers confidence for sparse history.

This is a directional recommendation from your own history, not an industry benchmark or a causal guarantee. Keep syncing and posting to improve its sample.

If Meta renames a metric or changes an edge, patch lib/meta-graph.ts or change FB_GRAPH_API_VERSION. The OAuth routes, storage, dashboard, and recommendation code do not contain Graph endpoint details.

Scheduling notes

The composer keeps the Page access token on the server. It:

starts a Reel upload at /{page-id}/video_reels;

proxies the binary video to Meta's returned rupload.facebook.com URL; and

finishes with video_state=SCHEDULED and scheduled_publish_time.

The UI enforces Meta's current scheduling window of 10 minutes to 6 months. The local proxy caps a file at 100 MB to avoid excessive memory use. Meta still validates codec, dimensions, aspect ratio, duration, and Page eligibility; an upload can therefore be accepted but fail during processing. Failed jobs remain visible in the local scheduling table with the returned error message.

Rate limits and failures

lib/meta-graph.ts retries HTTP 429, HTTP 5xx, Meta transient errors, and common rate-limit codes 4, 17, 32, and 613. It honors Retry-After when present and otherwise uses bounded exponential backoff with jitter. Non-retryable errors are returned to the dashboard with Meta's request ID when available.

The daily collector is intentionally conservative: it reads up to three feed pages per connected Page, persists successful Reel snapshots, and records partial sync failures without deleting good historical data.

Local API routes

Route

Purpose

GET /api/auth/facebook/start

Starts owner Facebook Login.

GET /api/auth/facebook/callback

Exchanges the code and stores encrypted Page tokens.

GET /api/dashboard

Returns historical totals, Reels, comparisons, recommendations, and queue data.

POST /api/sync

Runs a manual insight snapshot.

POST /api/cron/sync

Runs the secret-protected daily snapshot.

POST /api/schedule

Uploads and schedules a Reel.

POST /api/disconnect

Deletes the connection, tokens, Pages, snapshots, and scheduled-job history.

Deliberate scope limits

One owner only; no multi-user authorization model

Only Pages the owner administers

No competitor analysis

No keyword, tag, or SEO scoring

No attempt to bypass Meta permissions, review, or verification

Important: Any future public or multi-user version requires Meta App Review for the relevant permissions and may require Business Verification. This project intentionally does not build around, proxy, or attempt to evade those requirements.
