# Build Log

Notes on what broke while building this and how it got fixed.

## AI provider choices

- **Image generation**: tried NVIDIA's Qwen-Image API first — turned out to be self-host-only (no hosted endpoint), not usable directly. Tried Gemini's image models next — authenticated fine but the free tier grants zero image-generation quota. Settled on **Pollinations.ai**: free, no key, returns a hosted image URL directly.
- **Text generation**: Gemini (`gemini-flash-latest`) for article selection, captions, and image-brief writing — free tier works fine for text, capped at 20 requests/day per model. A real weekly run only uses ~3 calls, so this isn't a practical constraint in production, only during heavy same-day testing.

## RSS feeds

Several guessed feed URLs for Jakarta/Singapore property outlets 404'd or 403'd. Verified each candidate with a plain status-code check before wiring it in; landed on 4 working feeds (99.co Singapore, EdgeProp Singapore, Detik Properti, Rumah123's own guide feed).

## Getting the Instagram token

The longest part of the build. Meta's Graph API Explorer only shows permission checkboxes for whichever "use case" is currently enabled on the app — not the full list from their docs. Needed four scopes together before the Instagram Business Account ID would resolve: `instagram_basic`, `instagram_content_publish`, `pages_show_list`, `pages_read_engagement`. Exchanged the short-lived token for a long-lived Page token (`expires_at: 0`, i.e. doesn't expire) via the standard `fb_exchange_token` flow.

## n8n Merge node default

The "combine 4 RSS feeds" node failed with a "Fields to Match" error — `mode: "combine"` defaults to joining by matching fields, not concatenation. Fixed with `mode: "append"`.

## Approval security fix

The first version used n8n's plain one-click Approve/Reject email links. In testing, one resolved itself ~60 seconds after sending with no one clicking anything, and published a real post. Root cause (confirmed by reading n8n's own source): that link type resolves on a bare GET request, protected only by a bot-user-agent check. Fixed by switching to n8n's `customForm` response type — a real form with a Decision dropdown and Submit button that only resolves on an actual POST. Verified live: an untouched execution sat in "waiting" for 2+ minutes with no auto-resolve, while a genuinely human-submitted one went through correctly.

## Image quality

Two rounds of prompt fixes, both driven by actually looking at the output rather than assuming the first version was fine:
1. Style anchor was originally guessed ("bright outdoor lifestyle") — rewritten to match Rumah123's real ad style (moody navy-blue studio lighting) after seeing 3 real reference posts.
2. Hand/finger rendering was inconsistent (a known weakness of diffusion image models generally). Tightened the prompt to explicitly require correct five-finger anatomy and a calmer, non-gripping hand pose, rather than avoiding hands altogether.
3. Images were converging on the same "person + miniature house" scene regardless of article topic — fixed by requiring the prop be chosen specifically for each article's actual subject.

## Scheduling

Approval request goes out Monday 9 AM; actual publish is held until 5 PM the same day via n8n's Wait node (`resume: "specificTime"`), giving reviewers a window rather than posting the instant someone clicks approve.

## Known limitations

- No text/logo/CTA is baked into the image pixels — Pollinations can't reliably render small legible text. A production version would composite the headline, CTA pill, and logo on top using n8n's built-in Edit Image node.
- Hand/finger rendering quality still varies run to run — inherent to the free image model, not something prompt wording fully eliminates.
