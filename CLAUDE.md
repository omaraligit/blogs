# GreenCrew Blog

SEO "how to" articles for [GreenCrew](https://greencrew.space), a website and service uptime monitoring app with public status pages and Discord, Slack, and Telegram alerts (similar to UptimeRobot). greencrew.space can't be fetched from this environment, so rely on this description.

Posts live in `greencrew/blog/*.md`. The filename is the URL slug (`/blog/<slug>`). A GitHub Action rebuilds `greencrew/blog/index.json` after a push to `main`, so never edit `index.json` by hand.

## Workflow

1. `git checkout main && git pull`, then create a new branch from `main` named `blog/<topic-summary>`. Never branch from `example`, and never commit `example-blog.md`.
2. Read the existing filenames and H1 titles in `greencrew/blog/` and **don't repeat a topic that's already published**. Avoid big overlaps too (for example, no second Certbot or reverse proxy post).
3. Write **3 posts per batch** unless told otherwise. The default theme is Nginx and related subjects: web servers, TLS, Linux ops, reverse proxies, uptime, monitoring.
4. Commit only the new posts, push the branch, and open a PR into `main` with `gh pr create` (if `gh` is missing, give the user the compare URL). The user reviews and merges. Never push to `main` directly.

## Post Format

Follow this header block exactly. The index builder parses these lines:

```markdown
# How to <Do the Thing> (<Key Terms That Answer the Query>)

Author: [greencrew](https://greencrew.space/blog)

Tags: Nginx, Topic, Topic, DevOps

Description: <The actual answer in 1–2 sentences, including the key command or config, followed by what the guide covers.>

Date: YYYY-MM-DD   (today's date)
```

## Writing Rules

- **Intent-driven title and description** that already contain the answer, phrased like the search query.
- Open with a bold **"Short answer:"** paragraph that solves the problem in 3–5 sentences.
- Aim for **~1,500+ words**. Use plenty of **tables** (directive references, comparisons, troubleshooting "Symptom | Cause | Fix") and **copy-paste code blocks**.
- Use H2s that match search phrasing: "What Is X?", "Step 1: …", "Troubleshooting …", "Frequently Asked Questions" (3–4 questions), "Conclusion".
- SEO keywords go in the title, headings, and opening paragraph, written naturally with no keyword stuffing. Use the `/marketing-skills:ai-seo` guidance.
- Cite official docs (nginx.org, letsencrypt.org, etc.). Be technically accurate, and don't invent statistics or version numbers.
- **Link to GreenCrew wherever it's relevant**, usually a monitoring or alerting section near the end plus the conclusion. Make it a natural next step, not an ad.
- Only claim GreenCrew features we know exist: uptime/HTTP monitoring, public status pages, Discord/Slack/Telegram alerts. Don't claim check intervals, pricing, SSL-expiry alerts, or multi-region checks unless the user confirms them.
- Add internal links to related existing posts with `/blog/<slug>`.
- Match the voice of the existing posts: direct, practical, second person.
