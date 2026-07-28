# Prompts and AI Usage

Every prompt below is copy-pasted from the actual n8n node that sends it (the `jsonBody` field of the corresponding HTTP Request node in `workflow.json`), not reconstructed from memory. `{{article fields}}` shows where the runtime data gets substituted in.

The original creative brief this was adapted from was a manual, one-off "art director" prompt written for a single hand-picked image (couple on a terrace, fixed scenario). The job here was to turn that into a **framework** that runs unattended against whatever article the news pipeline picks each run — same brand voice and visual rules, but the subject and hook change every time based on real news content, and a model decides upfront which article is even worth a post.

## 1. Article selection prompt (Gemini) — picks the best lead-gen article, not just the newest

```
You are the content strategist for Rumah123, a property brand targeting Gen-Z homebuyers and investors in Jakarta and Singapore. Below is a list of property news articles from the last 7 days, each with an index number, publish date, title, and summary.

Your task: pick the ONE article with the strongest potential to generate marketing leads on Instagram, meaning it should be timely, emotionally engaging or surprising, directly relevant to someone currently thinking about buying, financing, or investing in property in Jakarta or Singapore, and not a generic evergreen how-to guide.

Candidates:
{{numbered list of up to ~20 articles: index, ISO pubDate, title, summary}}

Respond with ONLY a JSON object in this exact format, no markdown, no code fences: {"selectedIndex": <number>, "reasoning": "<one sentence reasoning in casual Bahasa Indonesia>"}
```

**Real output**, run against 20 real candidates pulled live from the 4 RSS feeds on 2026-07-28 (candidates included real news like an HDB policy change and a Knight Frank CEO resignation, alongside unrelated lifestyle filler like "Rumah123 (Indonesia)" feed's evergreen guides and a Detik Properti feed piece about the ruins of Troy):

```json
{"selectedIndex": 8, "reasoning": "Pencabutan aturan wait-out 15 bulan ini adalah kabar besar yang sangat relevan dan mendesak, berpotensi tinggi memicu konsultasi dan leads dari calon pembeli HDB resale."}
```

It correctly skipped the Troy-ruins trivia piece, a "Batman's real house is for sale" novelty piece, and several generic home-maintenance guides, and picked *"Government removes 15-month wait-out period for private property owners to buy HDB resale flats"* — a real policy change with direct buying-decision urgency.

## 2. Caption prompt (Gemini) — casual Bahasa Indonesia, hook + CTA + hashtag rule + branding

```
You are the social media copywriter for Rumah123, a Gen-Z focused property brand in Indonesia and Singapore. Write an Instagram caption in easygoing, casual Bahasa Indonesia based on this property news article.

STRUCTURE REQUIRED:
1. Hook line (first 1-2 lines) that stops the scroll, tied to the news angle
2. Body: explain the news in a few short relatable sentences, translate why it matters for someone thinking about buying or investing in property in Jakarta or Singapore
3. Comment-bait CTA: end with a question inviting comments
4. Hashtag rule: end with a hashtag block starting with #Rumah123 #RumahUntukSemua, then 5-8 more relevant hashtags mixing Indonesian and English
5. Product branding: naturally mention Rumah123 at least once as the place to explore listings

Article Title: {{article.title}}
Article Summary: {{article.contentSnippet}}
Article Link: {{article.link}}

Respond with ONLY the caption text, no preamble, no markdown labels.
```

**Real output**, for the HDB wait-out-period article above:

```
Bye-bye aturan nunggu 15 bulan! Mau pindah dari kondominium privat ke flat HDB resale di Singapura sekarang bisa langsung gas! 🇸🇬🚀

Pemerintah Singapura baru aja resmi ngehapus masa tunggu 15 bulan buat pemilik properti privat yang mau beli flat HDB resale.

Artinya apa buat kamu? Pasar makin fleksibel banget! Buat yang mau downsizing, ganti strategi investasi, atau mau cash out aset, prosesnya jauh lebih cepet tanpa drama nunggu berbulan-bulan lagi. Gerakan pasar makin lincah dan banyak peluang properti baru yang terbuka lebar.

Lagi nyari peluang investasi atau mau berburu hunian impian dari Singapura sampai Jakarta? Kamu bisa langsung kepoin dan eksplor ribuan opsi listing terbaik cuma di Rumah123!

Gimana menurut kamu, aturan baru ini bikin pasar properti makin menarik gak nih? Drop pendapat kamu di kolom komentar, ya! 👇

#Rumah123 #RumahUntukSemua #PropertyNews #InfoProperti #SingaporeProperty #HDBResale #InvestasiProperti #PropertyTrends #RealEstateSG
```

## 3. Image brief prompt (Gemini) — art-director reasoning, dynamic subject per article

Instead of a fixed "couple on a terrace" scenario, Gemini reads the article and invents a scenario and a specific prop each time, holding the visual style constant. Rewritten after being shown 3 real Rumah123 ad posts to match their actual style (dark navy-blue studio lighting, warm lamp glow, one candid laughing subject) — see BUILD_LOG.md for the iteration this went through.

```
ROLE: You are the art director for Rumah123, a Gen-Z focused techno-property brand in Indonesia and Singapore, briefing an image generation model for an Instagram feed post.

STYLE ANCHOR (match this exactly, based on Rumah123's real ad campaigns):
- Moody dark navy-blue indoor studio scene: a cozy living room or home interior blurred softly in the background (sofa, bookshelf, framed art, a warm glowing lamp), lit mostly by cool blue ambient light with warm golden lamp glow as accent
- One single Indonesian or Singaporean adult captured mid-genuine laugh: mouth open, eyes crinkled with real joy, an unmistakably happy candid expression, not a subtle smirk or closed-mouth smile
- Hands and fingers must NOT be visible anywhere in the frame: crop the shot at the chest or upper torso, well above where hands would be, so there is no risk of malformed hands or fingers appearing
- Editorial advertising photography quality, shallow depth of field, close crop on face and upper shoulders only, subject sharp and background softly blurred
- Composition: subject positioned center-right, taking up no more than the bottom two-thirds of the frame, leaving the entire top third and left side completely empty and uncluttered for a headline text overlay to be added later
- No embedded text, no logos, no buttons in the image itself, those are added in post-production

VARY THE PROP EVERY TIME, do not default to the same object: pick whichever concrete object best matches THIS SPECIFIC article, drawing from a wide range depending on the topic, for example: a physical house or building only if the article is literally about one specific property; a smartphone or tablet screen showing a chart or listing for market-data or price-trend articles; a stack of documents, a house key, a calculator, a laptop, a newspaper, a pair of keys on a ring, a stock ticker on a screen, a handshake gesture, a moving box, a paint swatch, a set of blueprints, or something else entirely you invent that fits. Never repeat the exact same prop and pose combination used for a different article's topic. A miniature architectural house model should be the exception, used only when the article is specifically about a single named building or development, not the default choice.

TASK: Read the news article below and invent one specific prop and micro-scenario a Gen-Z Indonesian or Singaporean property audience would find aspirational or relatable, tying the prop tightly and specifically to THIS article's actual subject matter (policy, market data, a company, a person, a building), not a generic property theme. Then write one concise vivid image-generation prompt, three to five sentences, plain descriptive language, following the style anchor above exactly, with NO hands or fingers visible in frame.

Article Title: {{article.title}}
Article Summary: {{article.contentSnippet}}

Respond with ONLY the final image-generation prompt text, no preamble, no explanation, no quotes.
```

**Real output** (earlier version of this prompt, for the HDB wait-out-period article — see BUILD_LOG.md for why the current version differs slightly):

```
A stylish young Singaporean adult with a genuine, delighted laugh looks down at a miniature architectural model of a modern HDB resale flat resting on a dark coffee table in front of them. The scene is set in a moody, dark navy-blue living room studio with cool ambient lighting, subtly highlighted by the warm golden glow of a blurred table lamp and bookshelf in the background. Captured in editorial advertising photography style with a shallow depth of field, the close crop highlights the subject's expressive face while keeping their hands relaxed and mostly out of frame. Positioned on the center-right, the composition leaves the top third and left side visually open and uncluttered for future text overlays.
```

Rendered via Pollinations.ai — see `sample-output-image.jpg` in this repo for the actual result.

## Original reference brief (for context on where the style rules came from)

The constraints structure (CONSTRAINTS / ROLE / SUBJECT / REASONING / FEW-SHOT REFERENCE / OUTPUT sections) is adapted from an earlier one-off prompt written for a single manually-chosen image:

```
CONSTRAINTS:
- Platform: Instagram | Digital Marketing Post | IG feed native | provide caption
- Design System: [Rumah123's Instagram profile]
- Style Anchor: [3 real Rumah123 post URLs]
- Must include: [hook line, comment-bait CTA, hashtag rule, product branding]
- Language: Easygoing casual Bahasa Indonesia

ROLE: Art director for a Gen-Z focused techno-property brand, briefing engagement breakthrough feed image

SUBJECT/TOPIC: Young Asian Gen-Z Couple, morning home terrace outdoor setting, natural simple forest scenery, depicting fulfilling life scenario, simple modern house and couple as main attention, minimal props / Describing a couple vibrant life after owning a house, engaging peer Gen-Z to start checking out their dream house types #Rumah123 #RumahUntukSemua

REASONING (before finalizing):
1. Focal Point = Couple face/expression, not the pose/activities mechanics - genuine and mid laugh calm, showcasing lively home as main central target
2. Thumbnail test: single subject, uncluttered background - passes at small size
3. Negative space: reserve one-third top reserved for caption text overlay and edge bottom for hook links, logo, slogan
4. Palette: Follow style anchor template, make it more tailored for bright outdoor cozy vibes

FEW-SHOT REFERENCE: assets on rumah123.com

OUTPUT: Generate Instagram digital marketing image post and caption to generate leads
1. IG feed post image with embed text contents and logos
2. IG feed caption matching image post
```

The workflow's prompts keep the REASONING rules (focal point on genuine expression, thumbnail test, negative space, palette-from-style-anchor) and the required-elements list (hook, CTA, hashtag rule, branding), but replace the fixed SUBJECT/TOPIC with a per-article reasoning step, since the whole point of this workflow is that the subject can't be hand-picked — it has to follow whatever the news actually is that week.
