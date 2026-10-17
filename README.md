# clean-reader
Updating the architecture of the mobile reader 9x16 app for greater control and flexibility
# Clean Reader API

A small, stateless web service that turns a link into clean, sanitised article HTML. It is the Python extraction code from the Streamlit app (recipes, articles, PDF/Word/EPUB, Substack, Wikipedia, Internet Archive) moved behind one login-protected endpoint, with the security checks a public internet service needs.

It stores nothing. Your library stays in Supabase; the offline reader (PWA) calls this service to capture an article, then saves the result to Supabase itself.

```
Reader app --login token--> POST /v1/extract --> fetches the page --> cleaned HTML back
```

## Endpoints

`GET /healthz` returns `{"ok": true}`. No login.

`POST /v1/extract` needs the header `Authorization: Bearer <Supabase access token>`.

```json
{ "url": "https://example.com/some-article", "method": "auto" }
```

`url` may also be a whole shared message that contains a link. `method` is optional: `auto`, `recipe_scraper`, `recipe_schema`, `jump_to_recipe`, `article_text`, `wayback`, `wikipedia`, `substack` (plus `archive_today`, `jina`, `medium` if enabled).

Success:

```json
{ "ok": true, "url": "...", "title": "...", "content_html": "<h1>...</h1><p>...</p>",
  "word_count": 1234, "method": "Article text", "cached": false }
```

Failure is always `{ "ok": false, "error": "<code>", "message": "<readable text>" }`:

| Status | error | Meaning |
|---|---|---|
| 400 | `bad_request`, `bad_url`, `bad_method`, `method_disabled` | Bad input or a method turned off on this server |
| 400 | `url_not_allowed` | The link points somewhere not allowed (internal addresses, odd ports, non-web links) |
| 401 | `unauthorized` | Missing, invalid or expired login |
| 403 | `forbidden` | Account not on `ALLOWED_USER_EMAILS` |
| 413 | `too_large` | Page or request bigger than the limit |
| 422 | `no_content`, `unreadable`, `too_large` | The page loaded but nothing readable came out |
| 429 | `rate_limited` | Too many requests (see `Retry-After`) |
| 502 | `fetch_failed` | The site couldn't be reached |
| 503 | `busy` | All workers busy, retry shortly |

## Deploy on Google Cloud Run (about 15 minutes)

Prerequisites: a Google Cloud project with billing enabled (the free monthly allowance covers light use, but a card must be on file) and the `gcloud` CLI.

1. Copy `env.example.yaml` to `env.yaml` and fill in your Supabase URL, the address your reader app is served from, and **your own email** under `ALLOWED_USER_EMAILS`.
2. Deploy from this folder:

```bash
gcloud run deploy clean-reader-api \
  --source . \
  --region us-east1 \
  --allow-unauthenticated \
  --max-instances 2 \
  --concurrency 8 \
  --memory 512Mi \
  --timeout 60 \
  --env-vars-file env.yaml
```

Choose a region that qualifies for Cloud Run's free tier (check the pricing page; `us-central1` and `us-east1` are typical).

`--allow-unauthenticated` is intended: Google's gateway stays open and the app itself requires a valid Supabase login for everything except `/healthz`.

3. Check it:

```bash
curl https://YOUR-SERVICE-URL/healthz                      # {"ok":true}
curl -X POST https://YOUR-SERVICE-URL/v1/extract \
     -H 'Content-Type: application/json' -d '{"url":"https://example.com"}'   # 401
```

4. Get a real token to try a full request (replace the values):

```bash
curl -s "https://YOUR-PROJECT.supabase.co/auth/v1/token?grant_type=password" \
  -H "apikey: YOUR-ANON-KEY" -H "Content-Type: application/json" \
  -d '{"email":"you@example.com","password":"..."}'          # copy access_token
curl -X POST https://YOUR-SERVICE-URL/v1/extract \
  -H "Authorization: Bearer ACCESS_TOKEN" -H 'Content-Type: application/json' \
  -d '{"url":"https://en.wikipedia.org/wiki/Rail_transport"}'
```

### Keeping costs near zero

- `--max-instances 2` caps how far it can scale, which caps the worst-case bill. Keep it.
- Create a **budget alert** in Google Cloud Billing (for example at $5) so a surprise reaches you by email.
- Idle instances cost nothing. The first request after a quiet period takes a few extra seconds.
- Turn off new sign-ups in Supabase (Authentication settings) or keep `ALLOWED_USER_EMAILS` set. Otherwise anyone who creates an account on your Supabase project can use your extraction service.

### Login keys

New Supabase projects sign logins with public keys, and this service fetches them automatically. Nothing to configure. Older projects use a shared secret instead: set `SUPABASE_JWT_SECRET` (store it with `--set-secrets` and Secret Manager rather than in `env.yaml`). Check Project Settings, then the JWT/API keys section, to see which you have.

## Using it from the offline reader

Add a capture box to the PWA that does this (the PWA's Content Security Policy `connect-src` must include the API address):

```js
const { data: { session } } = await sb.auth.getSession();
const res = await fetch(`${API_URL}/v1/extract`, {
  method: "POST",
  headers: { "Content-Type": "application/json", Authorization: `Bearer ${session.access_token}` },
  body: JSON.stringify({ url }),
});
const a = await res.json();
if (!res.ok) throw new Error(a.message);
await sb.from("saved_articles").upsert(
  { user_id: session.user.id, url: a.url, title: a.title, content: a.content_html, word_count: a.word_count },
  { onConflict: "user_id,url" });
```

Saving needs insert and update policies on `saved_articles` (`auth.uid() = user_id`). The Streamlit app already writes with your login, so they should exist.

## What protects it

| Risk | Protection |
|---|---|
| Tricked into reading internal services (cloud metadata, localhost, private networks) | Only http/https on ports 80/443. The host is resolved here and every address must be public. The connection is pinned to the checked address, so DNS changes can't redirect it. Redirects are followed manually and re-checked at every hop. |
| Huge downloads | Responses stream and stop at `MAX_DOWNLOAD_BYTES`. PDF page limit. Zip-bomb check for Word and EPUB. |
| Anyone using your service | Supabase login required, verified for signature, expiry, issuer and audience. Optional email allowlist. |
| Runaway cost | Per-user rate limits, a concurrency cap, `--max-instances`. |
| Malicious page content | Scripts, styles, forms, images and event handlers are removed. Only a short list of text tags and links survive. |
| Leaking what you read | Logs hold only a short user prefix and the site's host, never the full link or content. |
| Pages sent to third parties | Archive.today, Jina and Freedium are **off** by default (see below). |

### About the third-party fallbacks

`archive_today`, `jina` and `medium` send the link you are reading to outside services and can work around paywalls and access blocks. They are off by default. Turn them on only for your own private use (`ENABLE_THIRD_PARTY_FALLBACKS: "true"`). I would leave them off in anything you publish: app stores and publishers can object, and your readers' links would pass through those services. The Internet Archive's Wayback Machine stays on (`ENABLE_WAYBACK`).

## Known limits

- Rate limits and the cache live in memory, **per instance**. With `--max-instances 2` the effective limits can be up to twice what you set. Fine for a personal app, not for a public one.
- Sites that block automated access may still fail. The service imitates a normal Chrome browser, which helps but isn't a guarantee.
- Extraction runs only on pages reachable without logging in.
- File upload (PDF or Word picked from the phone) is not part of this version.

## Run the tests

```bash
python -m venv .venv && . .venv/bin/activate
pip install -r requirements-dev.txt
python -m pytest -q
```

141 tests cover the security checks, login verification, extraction on fixtures, and the full HTTP API against a local test server.

## Files

```
app/main.py        web API, CORS, rate limit, concurrency cap, logging
app/security.py    safe fetching (internal-address blocking, pinning, size caps)
app/extract.py     extraction and sanitising
app/auth.py        Supabase login verification
app/config.py      settings from environment variables
app/ratelimit.py   per-user limits        app/cache.py   short-lived cache
tests/             test suite
Dockerfile, requirements.txt, env.example.yaml
```
