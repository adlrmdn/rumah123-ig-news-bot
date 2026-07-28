# Build Log

This is the actual story of building this, including the parts that didn't work on the first try. I'm writing it because the parts that broke taught me more about how these APIs really behave than the parts that worked cleanly.

## Starting point

The brief: auto-generate Instagram posts from property news, for Rumah123, a Gen-Z property brand in Jakarta and Singapore. I had an existing one-off prompt I'd used before to hand-generate a single Instagram creative (a "couple on a terrace" lifestyle shot with a fixed caption). The job now was different: turn that one-off prompt into a *framework* that runs every hour, unattended, against whatever the news actually is that day — including deciding whether a given article is even worth posting about.

## Picking the AI providers (more churn than expected)

I originally reached for a Qwen image model hosted on NVIDIA's API catalog, since I had an `nvapi-` key on hand. It doesn't work that way: Qwen-Image on NVIDIA's platform is a self-hosted NIM container (`docker run ... nvcr.io/nim/qwen/qwen-image`), not something NVIDIA hosts for you to call over the internet. The "Try API" style endpoint just doesn't exist for that model — the button on its catalog page says "Apply to Self-Host," not "Get API Key." That was about half an hour of trying different endpoint URL guesses before I found the actual container-run instructions buried in the page's own embedded metadata.

Next I tried Gemini's image models with a real API key from Google AI Studio. The key authenticated fine — but every image model returned `RESOURCE_EXHAUSTED` with `limit: 0`. Not a rate limit, a *zero* quota: Google's free tier currently grants no image-generation allowance at all, full stop. A second fresh key hit the same wall. Text generation worked fine on the same key, which confirmed it wasn't a bad key, just a billing-gated feature.

Rather than ask for a credit card to be added, I switched to **Pollinations.ai** for images — free, no API key, a single GET request returns a rendered JPEG. It turned out to also be the simplest architecture: since it returns a hosted URL directly, Instagram's API can fetch the image straight from that URL without needing to download-and-rehost it anywhere.

## The RSS feeds needed real verification, not guessing

Half the property-news RSS URLs I initially guessed for Jakarta/Singapore outlets (Business Times, Straits Times, Kompas Properti, 99.co's Indonesia feed) returned 403s or 404s. I ended up testing about a dozen candidate feed URLs with plain `curl -w "%{http_code}"` before landing on 4 that actually returned valid RSS: 99.co Singapore, EdgeProp Singapore, Detik Properti, and Rumah123's own guide feed. Two of those lean more "evergreen guide" than "hard news" (Rumah123's own feed is full of things like "How to calculate PBB tax"), which is part of why I added the article-selection step described below — a plain "grab the newest item" approach would happily post about mop-floor tips.

## Getting the Instagram token was the longest part of this build

This is the part I'd warn anyone else about. Meta's Graph API permission model has changed shape more than once, and the Graph API Explorer only shows you the permission checkboxes for whatever "use case" is currently enabled on your app in the dashboard — not the full list from their docs. In order:

1. First token only had `instagram_manage_comments`. No publish permission showed up as an option in Explorer at all, because the app's Instagram product only had the "Moderate comments" use case turned on.
2. Enabling "Publish content" in the dashboard unlocked `instagram_content_publish` in Explorer.
3. `/me/accounts` (list Facebook Pages) came back empty even though a Page existed — because the token didn't have `pages_show_list` yet, a separate checkbox I had to explicitly search for.
4. Even after fixing that, `instagram_business_account` on the Page came back empty. I eventually found current Meta docs stating this field needs `instagram_basic` *in addition to* `instagram_content_publish` — a permission that lives under yet another use-case toggle, "Access profile info," that I hadn't enabled.
5. Only after all four scopes (`instagram_basic`, `instagram_content_publish`, `pages_show_list`, `pages_read_engagement`) were on the same token did the Instagram Business Account ID actually resolve.

After that: exchanged the short-lived user token for a long-lived one (`fb_exchange_token` grant, ~60 days), then derived a Page-scoped token from it, which came back with `expires_at: 0` — Meta's way of saying it never expires as long as the grant stays valid. That's the token actually wired into the workflow.

## A real bug: the Merge node's default mode

First live test run failed immediately with `You need to define at least one pair of fields in "Fields to Match"`. I'd set the "Merge Feeds" node (combining 4 RSS sources) to mode `"combine"` without realizing that n8n's Merge v3 node defaults its combine sub-mode to "combine by matching fields," which needs a join key — not what I wanted at all for concatenating 4 unrelated feeds. Fixed by using mode `"append"` there (simple concatenation), and being explicit about `combineBy: "combineByPosition"` on the one merge node that genuinely does need positional pairing (caption + image, which are always exactly 1-to-1 for a single selected article).

## A real security finding: the approval button auto-triggered itself

This is the most important thing I found, and I went and fixed it rather than just writing it down.

n8n's Send-and-Wait node, in its default "Approval" response mode, emails out Approve/Reject buttons as plain `<a href="...">` links. During a live test, the email went out at `05:25:22` and the workflow recorded `"approved": true` at `05:26:23` — 61 seconds later, with a real post going live on Instagram. Nobody clicked anything in that window. Since n8n was running on `localhost` (unreachable from any remote scanner), the trigger had to be something local — most likely a browser or mail-client link-preview fetch on the same machine.

I read n8n's own source for this (`n8n-nodes-base/dist/utils/sendAndWait/utils.js`) to find the actual mechanism instead of guessing further. Two things it confirmed:

- For the default `responseType: 'approval'`, clicking (or any GET request hitting) the button URL resolves the wait **immediately** — there's no confirmation step at all. The only protection is a check against the `isbot` npm package plus a specific Microsoft Teams/Skype preview-service check on the request's User-Agent header. That catches known bots and Teams link previews, but nothing that presents a normal browser User-Agent, which is exactly the gap that got hit.
- n8n *does* have a genuinely safe pattern already built in, just not the one I'd used: `responseType: 'customForm'`. For that mode, a GET request only renders an HTML form page — the wait is never resolved by the GET. It only resolves on a `POST` from an actual form submission.

**The fix**: switched the node from `approvalType: 'double'` (plain links) to `responseType: 'customForm'` with a single required "Decision" dropdown field (Approve / Reject) and a Submit button, and updated the downstream IF node to check `$json.data.Decision === 'Approve'` instead of `$json.data.approved === true`.

**Verified, not just assumed fixed**: I reactivated the workflow on a 1-minute test schedule. One execution's approval email sat correctly in `waiting` status for 2+ minutes with nobody touching it — no auto-resolve, unlike before. A second execution's email was genuinely opened and submitted by a real person, and *that* one correctly went all the way through to a live Instagram post. Two different outcomes for two different amounts of real human interaction — which is exactly the property that was missing before.

I deleted the one unauthorized test post manually afterward (Instagram's API doesn't allow programmatic deletion under normal publish permissions — that has to be done by hand in the app).

## The visual style needed real references, not a guess

My first version of the image-generation brief described "bright, cozy, outdoor lifestyle photography, warm natural light" — a reasonable-sounding guess, but wrong. Once I was shown 3 real Rumah123 ad posts, the actual style turned out to be almost the opposite: dark navy-blue moody studio lighting, one person genuinely laughing, holding a small symbolic prop (a miniature house model, a phone) that glows warmly against the cool background. I rewrote the style-anchor section of the prompt to match what was actually in front of me instead of what sounded plausible, and the resulting images are noticeably closer to the brand.

## A real quality bug: hands

Even with the style fixed, the generated people had visibly wrong hands and slightly off posture whenever the brief had them gripping the symbolic prop — fingers fused or missing, proportions slightly wrong. This is a well-known weakness of diffusion image models generally, not specific to Pollinations, but it was bad enough to undermine the whole post.

The fix wasn't a different model (Pollinations currently only exposes one, `sana`, via `image.pollinations.ai/models`) — it was prompt engineering: the brief now explicitly places the symbolic prop *on a table or surface in frame* rather than held in the subject's hands, and instructs hands to stay relaxed, in-lap or resting on the table edge, out of focus or mostly out of frame. Objects render far more reliably than hands interacting with objects. Re-ran the exact same article through the real Gemini prompt (not a hand-edited test version) and the resulting image has a clean, correctly-proportioned subject with no visible hand artifacts. The before/after is in this repo's git history on `sample-output-image.jpg` if you want to compare — the current file is the fixed version.

## What's still missing for a production version

1. **Text/logo compositing.** The reference ads have the headline, CTA button, and Rumah123 logo baked directly into the image. Pollinations (free tier) can't reliably render small legible text, so the current output is a clean photo only, with space reserved in the composition for an overlay to be added. The right fix is n8n's built-in Edit Image node (Sharp-based, no external key needed) to draw the headline + CTA pill + hashtag + logo onto the rendered photo before it's sent for approval — I scoped this but didn't build it in this pass.
2. The article-selection step currently caps candidates at ~20 per run to keep the prompt a reasonable size; in a live weekly cadence with proper dedup this naturally shrinks to just the genuinely new items each run, so this isn't a real constraint in practice, but it's worth knowing about if the schedule interval were shortened a lot.
