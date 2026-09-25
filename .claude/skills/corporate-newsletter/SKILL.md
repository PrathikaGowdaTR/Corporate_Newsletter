---
name: corporate-newsletter
description: "Builds the weekly Corporates competitive newsletter (tax compliance, e-invoicing/AP-AR automation, corporate legal, risk & fraud, ESG, and trade/supply-chain competitors) with the user, stopping at two points for their input: picking which articles go in, and reviewing the finished draft. Nothing gets sent automatically — the deliverable is a ready-to-paste HTML block, because this environment has no email-sending or Outlook-automation tool available."
---

# Corporates newsletter workflow

Builds the weekly Corporates competitive-intelligence newsletter ("Corporates Newsletter") with the user, stopping at two points for their input: picking which articles go in, and reviewing the finished draft. Nothing gets sent automatically — the deliverable is a ready-to-paste HTML block, because this environment has no email-sending or Outlook-automation tool available.

No shell/script execution in this environment. PowerShell is blocked by enterprise policy, so every step below must use `Read`, `Grep`, `Glob`, `WebFetch`, `WebSearch`, and the Browser tools (`mcp__Claude_Browser__*`) — never a Python/Node/Bash script, and never assume one can run. That is also why the source list and archive are kept as plain `.csv` twins of the `.xlsx` originals (see Inputs) — `Read`/`Grep` can parse CSV directly, but not the binary `.xlsx` format or the hyperlinks buried inside its cells.

Some sources may be permanently unreachable in this environment (network/policy-blocked domains, bot detection with no working feed alternative, or JS-rendered listings the tools can't see into) rather than just flaky on a given week. Step 2 point 4 below covers the standing fallback for both cases — don't re-litigate which sources are in which bucket each run, just apply the fallback whenever a source can't be scanned directly.

## Inputs

* **Source list**: `CTT_NL_Sources.csv` at the repository root by default (columns: `Source, Type, Link`). `Type` is `Press` (the company's own newsroom/press page), `Page` (a third-party aggregator/trade page), or `Link` (a company's own LinkedIn feed) — treat `Press` and `Link` as the company's own channel and `Page` as a trade/aggregator source when applying the relevance rules in Step 2. If the user names a different file or path when invoking this skill, use that instead.
  * `CTT_NL_Sources.xlsx` is the master file the source list is generated from — it stores each link as a hyperlink on a generic cell label (`Press`/`Page`/`Link`), which is exactly the kind of thing `Read` can't extract from a binary spreadsheet without running code. If the user says they've updated the `.xlsx`, regenerate `CTT_NL_Sources.csv` from it (ask the user to re-export it, or use the `xlsx` skill/Excel "Save As CSV" if available) rather than trying to parse the `.xlsx` directly.
* **House style reference**: `references/format.md` in this skill folder. Read it before Step 4 (summarizing/formatting) — it has the HTML skeleton to fill in, plus the masthead/footer image handling.
* **Masthead and footer images**: `Masthead.png` and `FooterBottom.png` at the repository root. Fixed content, reused unedited every week — see `references/format.md` for how to embed them in the Artifact. The masthead already has the "Corporates Newsletter" title baked in; `FooterBottom.png` is just the blank gap plus the Thomson Reuters logo — the "Quick links" box, its Atrium/Viva Engage/email links, and the subscribe line are all real HTML text with fixed URLs (see `references/format.md`), not images.
* **House style master reference**: `Corporates Newsletter format.html` at the repository root — a real past issue of this newsletter, exported from its actual send platform. `references/format.md` is a distilled version of it for weekly use; if anything is ambiguous, this file is the ground truth to check against.
* **Checklist template**: `references/checklist-template.html` in this skill folder. Used in Step 3 to hand the shortlist to the user as a checkable page instead of a wall of chat text — see that step for how to fill it in.
* **Historical archive**: `CTT_NL_Database.csv` at the repository root — the user's manually-curated archive of past newsletter items going back to 2025 (columns: `Year, Date, Headline, Weblink, Company name, Event type, Sub Segment, Trend, International NL`). It's large (1,150+ rows) — never read the whole file into context; use `Grep` for specific headlines, company names, or keywords instead. `CTT_NL_Database.xlsx` is the master file this CSV is generated from; treat it the same way as `CTT_NL_Sources.xlsx` above if it needs updating. Two uses:
   1. **Duplicate check**: before finalizing the shortlist in Step 3, spot-check any candidate item whose headline/company looks like it might already be old news by grepping the archive for that company name or a distinctive phrase from the headline — flag a likely repeat rather than silently including or excluding it.
   2. **Relevance calibration**: the archive's breadth (tax compliance, indirect/direct tax, e-invoicing, AP/AR automation, corporate legal tech, risk/fraud/KYC, ESG, trade compliance/supply chain, funding, M&A, product launches, executive moves, earnings — sourced from the same kind of company-newsroom/aggregator sites as this scan) is the working definition of "what counts as in-scope" per Step 2's relevance rules below — when in doubt whether something is on-topic, check whether a similar `Event type` or `Sub Segment` shows up in the archive. `Event type` (Partnership, Product Launch, Funding, Acquisition/M&A, Product Enhancement, Expansion, Regulation, Earnings, Certification, Appointment, Report, etc.) and `Sub Segment` (Tax Compliance, E-invoicing, AP/AR Automation, Corp Legal, Corp Risk, ESG, SCM, Direct Tax, Statutory Reporting, Tax Research, BEPS, TMS, etc.) are also useful as loose scan-order groupings in the Step 3 checklist (see there for why those groupings don't carry through to the final newsletter).

## Step 1 — Read the source list

Read `CTT_NL_Sources.csv` with the `Read` tool (plain CSV, no encoding tricks needed). Reconstruct the full `(source, type, link)` list. Needed for Step 2B only — skip this if the user is pasting a digest (Step 2A) and hasn't asked for a supplementary scan.

## Step 2 — Get this week's candidate stories

There are two ways into this step. **2A (pasted digest) is the default** — use it whenever the user pastes news text, and don't run an automated scan alongside it unless they ask for one. Fall back to **2B (automated scan)** only when the user has no digest to paste and wants you to go find stories yourself. **2C (standing aggregator check) always runs, every week, regardless of which of 2A/2B is used** — it's a fixed supplementary check, not an alternative to either.

### Step 2C — Standing aggregator check (always run)

In addition to whatever comes out of 2A or 2B, always check these six aggregator/trade pages every week — they're the `Type = Page` rows from `CTT_NL_Sources.csv` that the user has specifically called out as a standing requirement, not just part of the general source list:

* `https://www.businesswire.com/newsroom?industry=1050097&language=en`
* `https://www.businesswire.com/newsroom?industry=1000020&language=en`
* `https://www.vatupdate.com/`
* `https://www.internationaltaxreview.com/north-america?00000181-3b63-d208-ade1-7fe7b4b60000-page=2`
* `https://news.bloombergtax.com/daily-tax-report`
* `https://www.accountingweb.co.uk/latest-news-and-comment`

Try `WebFetch` on each first — don't assume the network block from past runs still holds, it costs nothing to check. As of this writing all six return `EGRESS_BLOCKED`, so in practice this means one `WebSearch` per source (source name + "news" + current month/year, e.g. `"VATupdate newsletter week September 2026 e-invoicing VAT developments"`), applying the same relevance and exclusion rules as everywhere else in Step 2.

Expect a thin yield here and don't burn excess effort chasing it — these are roundup/link-heavy aggregator pages, and `WebSearch`'s recall for a specific dated article buried in one is weak (the same bias problem noted in 2B: generic queries surface recurring recap content — e.g. Bloomberg Tax's annual projected-rates report, EY's recurring international-tax-developments bulletin — much more readily than the one dated, concrete item actually worth including that week). One or two solid items from all six sources combined in a given week is normal, not a sign something's wrong. Tag anything found this way the same as other `WebSearch`-sourced items (`"found via web search — verify before use"`).

### Step 2A — User-pasted digest (default)

The user pastes raw news text — company name as a heading, bullets or freeform text underneath, in whatever form they already have it (no need to ask them to restructure it). For this path:

* **Don't fetch or verify anything at this stage.** No `WebFetch`, `WebSearch`, or `CTT_NL_Sources.csv` lookups — this is pure text triage against what they gave you. Verification happens later, in Step 4, and only for the items they actually select.
* Read through every company's bullets and apply the same relevance and exclusion rules as Step 2B point 3 below (concrete company actions in scope vs. the standing exclusions) — the rules are the same regardless of how the raw material arrived.
* Split compound bullets into separate candidate items where a single bullet actually describes multiple distinct stories (e.g. two unrelated product updates run together in one sentence); don't force genuinely separate stories into one line just because the source text did.
* If a whole company's block is clearly off-topic or mismatched (e.g. content that's about a same-named but unrelated thing — a trade-compliance vendor's block that's actually full of unrelated automotive-safety-feature news, a company block that's pure macroeconomic/energy research with no tie to the newsletter's competitive scope), skip the whole block and say so in your chat message rather than force-fitting a few items out of it.
* Items at this stage won't have a working source URL (the pasted text usually doesn't include one) — that's expected and fine. Don't fabricate one. Leave the checklist item without a link and note once, prominently, that links get located after selection (Step 4), not before.

### Step 2B — Automated scan (fallback, only if asked)

Today's date matters here — check it (a system reminder usually states it; otherwise ask or infer from context) and compute the 7-day cutoff before you start.

For each link in `CTT_NL_Sources.csv`:

1. Try `WebFetch` first, asking it directly for news items published in the last 7 days with their headline, one-line description, and publish date. Give it the actual cutoff date so it doesn't have to guess "recent."
2. If `WebFetch` comes back empty, blocked, or clearly wrong (e.g. a JS-rendered site that returns no article text), fall back to the Browser tool: `navigate` to the URL, then `get_page_text` (or `read_page` if the text extraction misses the article list), and read dates/headlines yourself from what comes back.
3. Judge relevance by source `Type` and content, not a fixed list — the CSV will change over time:
   * `Type = Press` or `Type = Link` (a company's own newsroom, press page, or LinkedIn feed) — surface every genuine, concrete company action from the past 7 days: product launches/enhancements, integrations, partnerships, funding rounds, M&A, geographic expansion, regulatory/mandate changes, leadership appointments, earnings/results, and *technical* certifications (e.g. a Peppol Access Point authorization, a GROW-with-SAP or SOC 2 certification — something that changes what the product can actually do or where it can operate). Skip pure fluff (job postings, generic "we love our customers" filler, event photo recaps with no news) — flag borderline cases in the shortlist rather than silently dropping them.
   * `Type = Page` (a third-party aggregator/trade-press page, e.g. BusinessWire industry feeds, VATupdate, International Tax Review, Bloomberg Tax's daily report, AccountingWEB, the TaxTech 500 LinkedIn page) — surface items relevant to the market Thomson Reuters' Corporates business competes in. Concretely, that means anything touching: indirect/direct tax compliance, e-invoicing and AP/AR automation, corporate legal tech and contract lifecycle management, corporate risk/fraud/AML/KYC, ESG and sustainability reporting, trade compliance and supply-chain visibility, and AI as it relates to any of those (AI-native compliance tools, agentic AI in tax/legal/finance ops, GenAI-driven product features) — even where the piece is framed as general tech/business commentary rather than "tax/legal industry news" per se. Don't include stories with no connection to any of those threads just because they're within the date window (e.g. unrelated consumer-tech reviews, general macroeconomic commentary).
   * **Standing exclusion, regardless of source type or how it's framed — applies to both 2A and 2B**: analyst/industry *recognition* ("Named a Leader/Major Player" in an IDC MarketScape, Gartner Magic Quadrant, G2 grid, "Sample Vendor" in a Gartner Hype Cycle, an industry "Top 100" list placement, or similar); industry awards ("Best of," "Top Rated," a named awards-program win); and event/webinar *hosting or attending* announcements (a company announcing or promoting its participation in a conference, webinar, or summit). Also treat as excluded: pure thought-leadership/opinion content with no concrete company action, generic customer-success marketing with no specific dated action, and off-topic filler unrelated to the newsletter's competitive scope. Confirmed by checking `CTT_NL_Database.csv`: zero rows mention "award," essentially none are recognition placements, and `Event type = Event` appears only 4 times across 1,150+ rows — this newsletter has never run this kind of story, so don't start now even when a source is otherwise on-topic. A genuine regulatory certification (see above) is not the same thing as a marketing award — don't conflate the two.
4. If a site can't be fetched at all — network/policy-blocked, bot-blocked, broken link, or JS-rendered with no dates visible — don't just drop it, and don't stop at noting it as "couldn't scan." Always run a `WebSearch` fallback for that source before moving on: query the company/publication name plus a few relevant keywords (e.g. "partnership", "launch", "funding", "acquisition", "compliance" — whatever fits the source) and the current month/year. Apply the same relevance rules from point 3 above to whatever comes back.
   * Only surface an item if a search result actually confirms a publish date inside the 7-day window — WebSearch results are frequently from adjacent weeks, months, or undated, so don't include anything you can't pin to the window. A near-miss just outside the window (e.g. one day early) is worth mentioning to the user as a "just outside the window, include anyway?" note rather than silently adding or dropping it.
   * Tag anything found this way in the shortlist (e.g. `"found via web search — not a direct site scan, verify before use"`), since these results are approximate: no guarantee of exhaustiveness, and the link may point to a third-party writeup rather than the company's own page.
   * `WebSearch`'s `site:` operator is unreliable in this environment (it tends to ignore the filter and return generic/unrelated results) — don't rely on it; use plain keyword queries instead.
   * Still note the underlying fetch failure as "couldn't scan directly: <reason>" in your chat message so the user knows this source needed the fallback, even when the WebSearch step turns up nothing.
   * **Watch for a bias in what surfaces**: a generic query like "`<company>` news September 2026" tends to over-return analyst-recognition placements and press-release-syndicated funding/M&A news (these get picked up widely and rank well), while under-returning the smaller, more numerous partnership/integration/product-enhancement stories that actually make up most of `CTT_NL_Database.csv`. Don't stop at the first page of generic results and conclude a source had little news — try a second, more specific query per company (e.g. add "partners with" / "integrates" / "launches" / "expands to") before moving on, especially if the generic query only turned up recognition/award-type results (which don't count anyway, per the exclusion above).

Scanning ~105 sources takes a while. Give the user a brief progress note if it's taking more than a few dozen sources (e.g. "scanned 30/105 so far, X items found") rather than going silent for the whole step.

## Step 3 — Shortlist and hand off to the user (checkpoint 1)

This newsletter does not use categorization — every item runs under a single section, **"Corporate Tax and Trade"** (see Step 4). So the checklist groups found items by *scan source/theme* purely for the user's own scanning convenience, not because that grouping means anything downstream — it's discarded once they pick numbers.

Present the shortlist as an interactive checklist Artifact, not a plain chat list — the user finds it much easier to review and check off dozens of items visually than to type out numbers. Read `references/checklist-template.html` and follow the fill-in instructions in its top comment: number every found item sequentially across all sections (1 through N), group them into loose groupings (by theme is fine, it's just for readability), and flag borderline/low-relevance items with the `tag` field rather than dropping them. List "couldn't scan" sources (2B) or "skipped whole block" companies (2A) as plain text in your chat message below the artifact, not inside the checklist itself (they have nothing to check).

Coming out of Step 2A, items won't have a real `url` yet — that's expected, not a gap to fill in before publishing. Leave `url` empty (the template's rendering just won't show a link for that item) rather than guessing or reusing an unrelated link, and don't burn time trying to locate sources for the *whole* shortlist before the user has even picked — that work only happens for the items they select (Step 4).

Write the filled-in file (e.g. `wire_desk_<date>.html` in your scratchpad directory — a fresh file each week is fine, no need to reuse last week's) and publish it with the Artifact tool, title "Corporates Wire Desk", icon `"newspaper"` (reuse that title and icon every week so it reads as the same recurring tool, not a new thing each time — pass `icon` only on the first publish of a given week's artifact, and omit it on any re-publish to that same file/URL). Tell the user to check what they want and hit Copy, then paste the result back into chat.

If the Artifact tool fails or isn't available for some reason, don't burn more than one retry on it — fall back to a plain numbered markdown list grouped by source instead, and ask the user which numbers to include (accepting shorthand like `"1,3,7"`, `"all from Avalara"`, `"skip the Kintsugi one"`).

Whichever form it takes, confirm your interpretation of their selection back to them if there's any ambiguity, and do not proceed to summarizing until they've responded. This is the first checkpoint.

## Step 4 — Verify, source, summarize, and format

Read `references/format.md` now if you haven't already this run.

No categorization step here — whatever numbers the user gives you (e.g. "Include these numbered items in the newsletter: 5, 8"), all of them go under the single section **"Corporate Tax and Trade"**, in the order given. Don't ask them to categorize, don't invent additional sections, and don't publish a second categorize-view artifact — that machinery belongs to newsletters with multiple sections, which this one isn't.

**If any selected item came from Step 2A (a pasted digest) without a real `url` yet, this is where you find and verify it — not before.** For each such item:

* `WebSearch` for the specific claim (company name + the concrete action described, e.g. "Company X launches Y" or "Company X partners with Z") to find the actual source and confirm it's real.
* Check that what you find actually matches what the digest described — company, action, and rough timing. A source that's in the right neighborhood but describes a different deal, an older version of the same product, or a different company entirely is not a match.
* If nothing confirms the claim, or what you find contradicts it (e.g. the searches turn up the *other* company's own integration list and it doesn't mention this one), don't guess a plausible-looking link and don't drop it silently either — tell the user specifically what you checked and what didn't match, and ask whether they have a source or want it dropped. This happened with a "Kintsugi integrates with FreshBooks" claim that turned out to check out nowhere; flagging it beat either fabricating a citation or quietly cutting a story the user asked for.
* Once confirmed, use that real, verified URL as the article's link — never the placeholder or a guessed URL.

Write a neutral, factual summary for each article — no opinion, no editorializing, just what happened and why it matters competitively. Format is uniform (see `references/format.md`): one active-voice sentence per article, with the whole thing hyperlinked — no separate headline/summary split, no section gets a longer paragraph treatment than any other.

The section gets the italic line "All summaries in this section are AI-generated" at the end (only omit it if every article in it is verbatim/user-written text, which shouldn't normally happen here).

Fill in the HTML skeleton from `references/format.md` — masthead first (fixed content, only the date line changes), then the "Corporate Tax and Trade" section with its articles, then the footer. Also draft a subject line: `Corporates Newsletter - <Month DD, YYYY>` (matches the real reference issue's own title style).

## Step 5 — Review with the user (checkpoint 2)

Publish the draft as an Artifact (rendered HTML) so the user can see it formatted, not just as raw markup in chat — this also gives them something they can copy the rendered output from and paste into Outlook (pasting raw HTML tags as text is not what they want; pasting copied rich text from a rendered view preserves the formatting).

**Embed the two images as base64 `data:` URIs directly in the HTML, not as Artifact `files` references.** This matters for Outlook fidelity specifically: an `<img src="Masthead.png">` served through the Artifact tool's `files` mechanism resolves to a URL scoped to that private artifact, and when the rendered page is copied and pasted into Outlook, Outlook has to fetch that URL itself to display the image — which can silently fail (no auth, no network path from Outlook's renderer), breaking the pasted layout even though the artifact preview looked fine. A `data:image/png;base64,...` URI has the actual pixel bytes inline in the HTML itself, so nothing needs to be fetched — it survives copy/paste intact. If this environment has Bash/Python available, base64-encode the two PNGs directly (e.g. `base64 -w0 Masthead.png`) and substitute the resulting strings into the `<img src="data:image/png;base64,...">` slots in `references/format.md`'s skeleton; write the fully-substituted HTML to your scratchpad before publishing, since inlining base64 directly in a tool call is unwieldy. If no script execution is available in this environment, fall back to the `files` parameter approach and flag to the user that pasted images may not survive into Outlook, since there's no other way to get image bytes into the page without one.

Use the `title` parameter (e.g. "Corporates Newsletter Draft") and pass `icon` only on the first publish of this issue (e.g. `"mail"`); omit `icon` on every re-publish in the review loop below so it doesn't change. Also show the subject line suggestion.

Ask for corrections. Loop on their feedback — re-render and re-publish the same artifact URL each time — until they say it's good to go. If they report the pasted result doesn't match the preview, ask specifically what differs (missing images, wrong font, collapsed spacing, lost colors) rather than guessing at another fix blind.

## Step 6 — Hand off the final draft

Once approved:

* Give the user the final HTML (they can copy it from the rendered Artifact) and the subject line.
* Remind them: open a new email in Outlook, paste the copied rendered content into the body, fill in the "To" field themselves (distribution list is their own responsibility), and send it themselves. This skill does not send email or touch Outlook directly — there is no email tool available in this environment.
