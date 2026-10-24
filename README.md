# Serial recap bot (Dangal TV serials -> 2-3 min "motion comic" recaps), standalone repo

## Daily Publish Schedule (9:00 AM – 4:00 PM IST)
The bot runs at **12:00 AM IST (Midnight)** via an external cron trigger and schedules the 8 daily recap videos on YouTube at hourly slots from **9:00 AM to 4:00 PM IST**:

| Show Key | Serial Name | Scheduled YouTube Publish Time (IST) |
| :--- | :--- | :--- |
| `kaisi` | Kaisi Teri Dillagi (कैसी तेरी दिल्लगी) | **09:00 AM IST** |
| `bhabhi` | Hamari Bhabhi Number 1 (हमारी भाभी नंबर 1) | **10:00 AM IST** |
| `sohni` | Sohni Aur Nahi Koi Honi (सोहनी और नहीं कोई होनी) | **11:00 AM IST** |
| `rimjhim` | Rimjhim (रिमझिम) | **12:00 PM IST** |
| `mannsundar` | Mann Sundar (मन सुंदर) | **01:00 PM IST** |
| `atisundar` | Mann Atisundar (मन अतिसुंदर) | **02:00 PM IST** |
| `patibrahma` | Pati Brahmachari (पति ब्रह्मचारी) | **03:00 PM IST** |
| `ishq` | Ishq Junooni (इश्क़ जुनूनी) | **04:00 PM IST** |

## Why We Don't Use GitHub's Built-In Cron (And How to Trigger at 12:00 AM IST Sharp for Free)
GitHub's internal `schedule:` cron frequently delays runs by 30–90 minutes or drops them during peak hours. Instead, this repo uses **external `workflow_dispatch` / `repository_dispatch`** triggered at **12:00 AM IST sharp** using any of the 100% free external schedulers included in `/scheduler`:

1. **Option A — Google Apps Script (`scheduler/google_apps_script.gs`, Recommended & 100% Free)**:
   - Open [script.google.com](https://script.google.com), create a project, and paste `scheduler/google_apps_script.gs`.
   - Set `GITHUB_OWNER`, `GITHUB_REPO`, and `GITHUB_PAT` (a GitHub Personal Access Token with `repo` and `workflow` permissions).
   - Run `setupTriggers()` once. It installs an exact **12:00 AM IST (`Asia/Kolkata`)** daily trigger plus a **2-hour Watchdog Log-Inspector** that checks GitHub run logs and automatically re-triggers any failed show.
2. **Option B — Cloudflare Worker (`scheduler/cloudflare_worker.js`, 100% Free)**:
   - Deploy `scheduler/cloudflare_worker.js` on Cloudflare Workers Free Tier with cron `30 18 * * *` (12:00 AM IST) and `30 20,22,0,2 * * *` (watchdog checks).
3. **Option C — cron-job.org (100% Free Web Cron)**:
   - Create a free job at [cron-job.org](https://cron-job.org) set to **00:00 Asia/Kolkata (12:00 AM IST)**.
   - Method: `POST` to `https://api.github.com/repos/<OWNER>/<REPO>/actions/workflows/recap.yml/dispatches`
   - Headers: `Authorization: Bearer <YOUR_GITHUB_PAT>`, `Accept: application/vnd.github+json`
   - Body: `{"ref":"main","inputs":{"only_show":"all","test_mode":"false","upload":"true","publish_now":"false"}}`

## 3-Layer Self-Healing Log Watchdog & Zero-Duplicate Protection
1. **In-Job Log Capture & Auto-Retry (`recap.yml` & `main.py`)**:
   - Every show job streams its logs into `state/<show>_status.json`. If Attempt 1 fails (e.g., temporary Gemini 503 or network timeout), the job reads the status log, cleans up temp files, waits 45 seconds, and retries automatically inside the same run.
2. **Post-Run Watchdog Supervisor (`recap/watchdog.py`)**:
   - After the matrix finishes (and whenever triggered with `only_show=watchdog`), `watchdog.py` inspects the GitHub Actions job logs, reads `state/<show>_status.json`, and checks Dangal TV's live feed. If any show failed, timed out, or has an unprocessed new episode, it automatically dispatches a targeted rerun for **only that show** (up to 4 self-healing attempts per day).
3. **Triple Anti-Duplicate Lock (Never Uploads an Episode Twice)**:
   - **Video ID + Episode Number State (`state/<show>.json`)**: Stores both the source YouTube video ID (`JNdlYnNneKE`) and the episode number (`ep:452`, `target_ep:453`). Even if Dangal TV re-uploads the same episode with a new video ID, it is never processed twice.
   - **Live YouTube Channel Pre-Check (`youtube.already_uploaded()`)**: Before generating a video AND again right before uploading, the bot queries your YouTube channel's uploads list (including private/scheduled videos). If an episode is already uploaded or scheduled on your channel, it syncs `state/<show>.json` and skips immediately.

## Facebook Pages and Instagram Reels (Opt-In)

YouTube remains the existing default destination. Facebook and Instagram publishing are disabled unless their corresponding `FACEBOOK_ENABLED` or `INSTAGRAM_ENABLED` setting is explicitly `true`. The story, narration, and generated illustrations are shared. Facebook receives the existing landscape MP4; Instagram receives an independently rendered 1080×1920 MP4 with a blurred background made from each scene image, Hindi captions, and the same word timings. The vertical version is not cropped from the finished landscape video, and its audio stream is copied from the landscape output.

### Local requirements and preview

The existing Python dependencies remain in `requirements.txt`. Video rendering also requires FFmpeg/ffprobe, a Hindi-capable font, and Pillow built with RAQM for correct Devanagari shaping (the Actions workflow installs FFmpeg, Noto Devanagari, and RAQM libraries). On Windows, install FFmpeg and add its `bin` directory to `PATH`; install a Devanagari font supported by `recap/render.py` and a Pillow/RAQM build. Without RAQM, Pillow warns that it falls back to basic layout, so inspect Hindi text carefully before publishing.

Create a local `.env` using `.env.example` as a reference. Never commit `.env`, access tokens, or private media URLs. For an Actions preview, set `test_mode=true`, leave `upload=false`, leave Facebook and Instagram disabled, and set `render_vertical=true`. This generates a short landscape test video and its vertical counterpart but does not publish either. Local test runs also require the existing source/API configuration needed by the episode generator.

To run YouTube-only publishing, leave `FACEBOOK_ENABLED=false` and `INSTAGRAM_ENABLED=false` (or unset) and use the existing YouTube secrets and `upload=true` behavior. Existing landscape output names and YouTube metadata/upload code remain unchanged.

### Step-by-step Meta setup

The publisher code is opt-in, so do not enable a destination until its app permissions, current API version, and a test publish have been confirmed. Facebook uses the landscape MP4. Instagram uses the vertical MP4 and needs Meta to fetch that file from a public HTTPS host.

1. **Prepare the accounts.** Make sure you have administrator/full content access to the Facebook Page. Change the Instagram account to a Professional **Business** or **Creator** account. For the Facebook Login-based Instagram Graph API route, link the Instagram professional account to the Facebook Page. Confirm you can manage both from the same Meta business/account.
2. **Create the Meta app.** Open [Meta for Developers](https://developers.facebook.com/), create an app owned by the correct business, and add the Facebook Login/Graph API and Instagram publishing products that Meta currently recommends for your account type. Complete any business verification or app setup checks shown in the dashboard. Do not paste credentials into this repository.
3. **Configure app access.** Add your Facebook user, Page, and Instagram professional account as app roles/test users while the app is in development. In the app dashboard, enable the permissions Meta currently requires for Page video publishing and Instagram Reel publishing. Permissions and approval requirements differ by login product and app mode; use Meta's current permissions dashboard and docs rather than copying scope names from an old tutorial.
4. **Obtain and verify credentials.** Use Meta's official login/token flow for the product you selected. Obtain the Facebook Page ID and a Page access token with publishing access, and the Instagram professional-account ID and token that is valid for the Instagram publishing endpoints. Verify both accounts and permissions using Meta's official tools. Do not send tokens in chat, paste them in workflow logs, or add them to source files.
5. **Set up Instagram video hosting.** Configure an HTTPS host that accepts the generated vertical MP4 from this workflow and returns a stable public URL readable by Meta without login. Set `INSTAGRAM_VIDEO_URL_TEMPLATE` to that URL pattern, for example `https://your-host.example/recaps/{filename}`. `{filename}` is replaced with the vertical MP4 filename. **This repository does not upload the MP4 to a hosting provider**, so the template alone is not enough: add and test a secure upload step to your chosen host before enabling Instagram. Do not use a URL that requires cookies, a private account, or a secret embedded query string.
6. **Add GitHub Actions secrets.** In the repository, open **Settings → Secrets and variables → Actions → New repository secret** and add the values below. Keep all IDs/tokens in Secrets, not Variables.
7. **Add GitHub Actions variables.** In the same **Settings → Secrets and variables → Actions** area, add the Variables below. Set `META_GRAPH_API_VERSION` only after checking which version is currently supported in Meta's official Graph API changelog.
8. **Test privately first.** Leave `FACEBOOK_ENABLED` and `INSTAGRAM_ENABLED` false. Run a video preview and check both local outputs. Then enable one destination at a time using an app/Page/account you control, review the resulting post, and verify the saved `state/<show>_publishing.json`. Do not use a real public Page/account for initial testing unless you intend to publish publicly.
9. **Maintain access.** Monitor token expiry and app permission changes. This bot does not refresh Meta tokens automatically. Renew them through the same official Meta flow, replace the corresponding Actions Secret, and reauthorize if access is revoked.

#### GitHub Actions Secrets

| Secret | Value |
| --- | --- |
| `FACEBOOK_PAGE_ID` | ID of the one Page this integration should publish to |
| `FACEBOOK_PAGE_ACCESS_TOKEN` | Page access token authorized for Page video publishing |
| `INSTAGRAM_USER_ID` | ID of the Instagram professional account |
| `INSTAGRAM_ACCESS_TOKEN` | Access token authorized for the Instagram publishing flow you configured |
| `INSTAGRAM_VIDEO_URL_TEMPLATE` | Public HTTPS file URL template, e.g. `https://your-host.example/recaps/{filename}` |

#### GitHub Actions Variables

| Variable | Value |
| --- | --- |
| `FACEBOOK_ENABLED` | `false` until verified; set `true` to request Facebook publishing |
| `FACEBOOK_CAPTION` | Optional Facebook description; blank uses generated episode description |
| `INSTAGRAM_ENABLED` | `false` until verified; set `true` to request Reel publishing |
| `INSTAGRAM_CAPTION` | Optional Reel caption; blank uses generated episode text and hashtags |
| `META_GRAPH_API_VERSION` | Meta-supported Graph API version in `vNN.N` format; deliberately no default |

#### What this code currently calls

The Facebook adapter is written for the Page video route `/{page-id}/videos` using `start`, `transfer`, and `finish` phases. The Instagram adapter is written for Reel container creation, status polling, and `/{ig-user-id}/media_publish`. These exact routes, parameters, permissions, and upload formats must be confirmed against Meta's current docs before enabling a production app. The adapters accept pre-obtained tokens; token acquisition and refresh are not implemented.

The Meta developer pages could not be fetched during implementation, so the exact active API version, permission names, app-review requirements, account eligibility, caption limits, token renewal details, and upload parameter requirements have not been verified here. `META_GRAPH_API_VERSION` has no default intentionally; the code refuses to call an unconfigured version instead of silently selecting an unverified one. Check [Meta's Graph API documentation](https://developers.facebook.com/docs/graph-api/) and current Instagram publishing docs before enabling either platform.

### Independent publishing state and recovery

Per-episode platform status is written atomically to `state/<show>_publishing.json` and committed by the existing Actions state-save step. Successful destinations are skipped on retries. Confirmed transient failures can be retried; missing configuration is marked `blocked_config` and becomes eligible once its required variables are supplied. Permanent API errors are not retried automatically. An interrupted/ambiguous request is recorded as `unknown` and is not retried automatically, because the remote service may already have published it. Check the Page/account before manually resetting an ambiguous state to `failed`. `state/*.lock` files are local coordination artifacts and ignored by Git.

For safe tests, keep both Meta enable variables false and use `render_vertical=true` only for a non-publishing preview. Run tests with `python -m unittest discover -s tests -v`; API unit tests use mocked HTTP responses and never contact Meta. To enable a destination, first test with a non-public/test Page and account where available, review the produced MP4 and caption, set the Graph API version verified from Meta's docs, configure secrets and the public Instagram media URL, and only then set that destination's enable variable to `true`.
