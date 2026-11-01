# Daily Cyber Awareness Quiz

Adds fresh quizzes to the existing **Quiz** page. The 10 original quizzes, the certificates, the 75% pass rule and English/Hindi switching are untouched: daily quizzes are played with the same `assets/quiz.js`.

## What visitors see
- `/quiz/` – below the original 10 cards, a "Daily Cyber Awareness Quiz" section with the latest quizzes (same card design) and a "Previous daily quizzes" button. **If no daily quiz exists yet, nothing is shown and the page looks exactly as before.**
- `/quiz/daily/` – the archive of all daily quizzes. `/quiz/daily/?d=YYYY-MM-DD` plays one.
- On every question: level, topic, official source name, link to the original reference, publication date, explanation. Pass mark = 75% rounded up (4 questions → 3, 10 questions → 8; the original quizzes are 15/20).

## How the daily job works (`.github/workflows/daily-quiz.yml`)
Runs at 03:30 UTC (= 09:00 IST) and on demand.
1. Offline self-tests run first.
2. `daily-quiz-tools/sources.mjs` lists the approved sources (Indian government hosts only; nothing is fetched from other hosts). New or changed items since the last run are picked up (`assets/daily/state.json` remembers what was already used).
3. For each item the AI writes original questions (English + Hindi) **only from the fetched text**, and must copy one exact passage from the source as evidence.
4. Checks before anything is published: structure, Hindi is Devanagari, evidence really is in the source, no copied wording, no number that is not in the source, not a duplicate of any earlier daily question or any of the 10 original questions, a **second independent AI check** must pick the same answer from the source alone, a third check that the explanation is supported by the source, and a fourth check that the Hindi text really matches the English (same meaning, same option order). Duplicate detection compares the question wording and, for reworded questions, also the correct answer.
5. Valid questions are written (max 10 per day) to `assets/daily/quizzes/DATE.json`, the archive index `assets/daily/index.json` is updated, then everything is committed and GitHub Pages is asked to rebuild.
6. **No new relevant information → no questions are created.** The quota of 10 is a maximum, never a target.
7. **Any failure** (source down, AI error, validation error) → nothing is written or committed, old quizzes stay online, the error is in the Actions log and run summary, and the next run retries.

## Setup (once)
1. Push this folder to your GitHub repository (the same repo as your site).
2. Settings → Secrets and variables → Actions → **New repository secret**: name `ANTHROPIC_API_KEY`, value = your key from console.anthropic.com. (Never put it in any file.)
3. Optional: Variables → `QUIZ_MODEL` to choose the model (default `claude-sonnet-5-5`).
4. Settings → Actions → General → Workflow permissions must allow **Read and write**.
5. The workflow is enabled automatically. GitHub only runs scheduled workflows from the default branch.

## Manual test
Actions → **daily-cyber-quiz** → Run workflow.
- Tick **dry run** first: it generates and validates, prints the quiz in the log, saves nothing.
- Then run it without dry run. Open the run summary to see what happened, then `/quiz/` on your site.
- The step "Show what every approved source really returns" shows, for each source, whether the page gave readable text (`OK`) or almost nothing (`THIN`, usually a page that needs JavaScript) or failed.

## Adding or fixing sources
- Edit `daily-quiz-tools/sources.mjs` (`kind: "links"` = a listing page whose new links become items, `kind: "page"` = one page watched for changes).
- Or, with no code: put exact official URLs in `daily-quiz-tools/manual-sources.json`, e.g. `[{"name":"I4C","url":"https://cybercrime.gov.in/…"}]`. Each is used once, and again only if the page text changes.
- Other hosts must be added to `ALLOWED_HOST_SUFFIXES` on purpose.

## Approved sources (all Indian-government hosts; nothing else is fetched)
CERT-In advisories list, I4C (cybercrime.gov.in and i4c.mha.gov.in/awareness.aspx - the web home of the CyberDost awareness campaign; CyberDost's X/Facebook/Instagram/Telegram pages are NOT fetched), ISEA (isea.gov.in and infosecawareness.in), Cyber Swachhta Kendra (csk.gov.in), MyGov Stay Safe Online. `infosecawareness.in` and `i4c.mha.gov.in/awareness.aspx` were added in the final review; their addresses come from government/CBSE circulars but have not been fetched live yet.

## Honest limits
- **Indian government sites may block or slow requests from GitHub's servers (outside India).** If the `--diagnose` step shows `FAILED` or `THIN` for a source, that source gives nothing; use `manual-sources.json` with a specific page, or run the job from a runner located in India.
- On GitHub Pages there is no server, so the certificate registration call (`/api/certificate`) used by ALL quizzes cannot succeed there: the page then says the certificate is not registered online but can still be downloaded. This is existing behaviour, unchanged.
- Live fetching of the official sites and the AI calls have **not** been run from GitHub yet. Pages that need JavaScript (some government portals) give no readable text and are skipped; use `manual-sources.json` with a specific article/advisory URL for those.
- Many CERT-In advisories are technical (for administrators). The AI is told to skip those, so some days produce nothing.
- "Recent" means new or changed since the last run. Official pages often show no publication date, so the question's date is the day it was published on this site.
- The questions are AI-written from official text and machine-checked, not reviewed by a person. Spot-check them occasionally; you can delete a bad question from its JSON file.
- State files are public (GitHub Pages serves the whole repo). They contain only URLs and hashes.

## Files
New: `assets/daily-quiz.js`, `assets/daily-quiz.css`, `assets/daily/` (topics, index, state, quizzes/), `quiz/daily/index.html`, `daily-quiz-tools/`, `.github/workflows/daily-quiz.yml`, this file.
Changed: `quiz/index.html` (+1 stylesheet link, +1 empty container `<div id="daily-quiz-root">`, +1 script tag), `sitemap.xml` (+1 line for /quiz/daily/), `.github/workflows/update-news.yml` (push now rebases and retries so it cannot clash with the daily job).
Tests: `node --test daily-quiz-tools/tests/daily.test.mjs` (offline).
