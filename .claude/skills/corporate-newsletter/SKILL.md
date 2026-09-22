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
* **Masthead and footer images**: `Masthead.png` and `Footer.png` at the repository root. Fixed content, reused unedited every week — see `references/format.md` for how to embed them in the Artifact.
* **Checklist template**: `references/checklist-template.html` in this skill folder. Used in Step 3 to hand the shortlist to the user as a checkable page instead of a wall of chat text — see that step for how to fill it in.
* **Historical archive**: `CTT_NL_Database.csv` at the repository root — the user's manually-curated archive of past newsletter items going back to 2025 (columns: `Year, Date, Headline, Weblink, Company name, Event type, Sub Segment, Trend, International NL`). It's large (1,150+ rows) — never read the whole file into context; use `Grep` for specific headlines, company names, or keywords instead. `CTT_NL_Database.xlsx` is the master file this CSV is generated from; treat it the same way as `CTT_NL_Sources.xlsx` above if it needs updating. Two uses:
   1. **Duplicate check**: before finalizing the shortlist in Step 3, spot-check any candidate item whose headline/company looks like it might already be old news by grepping the archive for that company name or a distinctive phrase from the headline — flag a likely repeat rather than silently including or excluding it.
   2. **Relevance calibration**: the archive's breadth (tax compliance, indirect/direct tax, e-invoicing, AP/AR automation, corporate legal tech, risk/fraud/KYC, ESG, trade compliance/supply chain, funding, M&A, product launches, executive moves, earnings — sourced from the same kind of company-newsroom/aggregator sites as this scan) is the working definition of "what counts as in-scope" per Step 2's relevance rules below — when in doubt whether something is on-topic, check whether a similar `Event type` or `Sub Segment` shows up in the archive. `Event type` (Partnership, Product Launch, Funding, Acquisition/M&A, Product Enhancement, Expansion, Regulation, Earnings, Certification, Appointment, Report, etc.) and `Sub Segment` (Tax Compliance, E-invoicing, AP/AR Automation, Corp Legal, Corp Risk, ESG, SCM, Direct Tax, Statutory Reporting, Tax Research, BEPS, TMS, etc.) are also useful references for category names if the user asks for suggestions during categorizing.

## Step 1 — Read the source list

Read `CTT_NL_Sources.csv` with the `Read` tool (plain CSV, no encoding tricks needed). Reconstruct the full `(source, type, link)` list.

## Step 2 — Scan every source for the last 7 days

Today's date matters here — check it (a system reminder usually states it; otherwise ask or infer from context) and compute the 7-day cutoff before you start.

For each link:

1. Try `WebFetch` first, asking it directly for news items published in the last 7 days with their headline, one-line description, and publish date. Give it the actual cutoff date so it doesn't have to guess "recent."
2. If `WebFetch` comes back empty, blocked, or clearly wrong (e.g. a JS-rendered site that returns no article text), fall back to the Browser tool: `navigate` to the URL, then `get_page_text` (or `read_page` if the text extraction misses the article list), and read dates/headlines yourself from what comes back.
3. Judge relevance by source `Type` and content, not a fixed list — the CSV will change over time:
   * `Type = Press` or `Type = Link` (a company's own newsroom, press page, or LinkedIn feed) — surface every genuine company announcement from the past 7 days: product launches/enhancements, partnerships, funding, M&A, expansions, certifications, executive moves, earnings/results. Skip pure fluff (job postings, generic "we love our customers" filler, event photo recaps with no news) — flag borderline cases in the shortlist rather than silently dropping them.
   * `Type = Page` (a third-party aggregator/trade-press page, e.g. BusinessWire industry feeds, VATupdate, International Tax Review, Bloomberg Tax's daily report, AccountingWEB, the TaxTech 500 LinkedIn page) — surface items relevant to the market Thomson Reuters' Corporates business competes in. Concretely, that means anything touching: indirect/direct tax compliance, e-invoicing and AP/AR automation, corporate legal tech and contract lifecycle management, corporate risk/fraud/AML/KYC, ESG and sustainability reporting, trade compliance and supply-chain visibility, and AI as it relates to any of those (AI-native compliance tools, agentic AI in tax/legal/finance ops, GenAI-driven product features) — even where the piece is framed as general tech/business commentary rather than "tax/legal industry news" per se. Don't include stories with no connection to any of those threads just because they're within the date window (e.g. unrelated consumer-tech reviews, general macroeconomic commentary).
4. If a site can't be fetched at all — network/policy-blocked, bot-blocked, broken link, or JS-rendered with no dates visible — don't just drop it, and don't stop at noting it as "couldn't scan." Always run a `WebSearch` fallback for that source before moving on: query the company/publication name plus a few relevant keywords (e.g. "partnership", "launch", "funding", "acquisition", "compliance" — whatever fits the source) and the current month/year. Apply the same relevance rules from point 3 above to whatever comes back.
   * Only surface an item if a search result actually confirms a publish date inside the 7-day window — WebSearch results are frequently from adjacent weeks, months, or undated, so don't include anything you can't pin to the window. A near-miss just outside the window (e.g. one day early) is worth mentioning to the user as a "just outside the window, include anyway?" note rather than silently adding or dropping it.
   * Tag anything found this way in the shortlist (e.g. `"found via web search — not a direct site scan, verify before use"`), since these results are approximate: no guarantee of exhaustiveness, and the link may point to a third-party writeup rather than the company's own page.
   * `WebSearch`'s `site:` operator is unreliable in this environment (it tends to ignore the filter and return generic/unrelated results) — don't rely on it; use plain keyword queries instead.
   * Still note the underlying fetch failure as "couldn't scan directly: <reason>" in your chat message so the user knows this source needed the fallback, even when the WebSearch step turns up nothing.

Scanning ~105 sources takes a while. Give the user a brief progress note if it's taking more than a few dozen sources (e.g. "scanned 30/105 so far, X items found") rather than going silent for the whole step.

## Step 3 — Shortlist and hand off to the user (checkpoint 1)

Present the shortlist as an interactive checklist Artifact, not a plain chat list — the user finds it much easier to review and check off dozens of items visually than to type out numbers. Read `references/checklist-template.html` and follow the fill-in instructions in its top comment: number every found item sequentially across all sections (1 through N), group them into sections the same way you would for the final newsletter (by theme — "Tax & Compliance," "Legal & Risk," "Trade & Supply Chain," etc. — whatever grouping actually fits this week's finds), and flag borderline/low-relevance items with the `tag` field rather than dropping them. List "couldn't scan" sources as plain text in your chat message below the artifact, not inside the checklist itself (they have nothing to check).

Write the filled-in file (e.g. `wire_desk_<date>.html` in your scratchpad directory — a fresh file each week is fine, no need to reuse last week's) and publish it with the Artifact tool, title "Corporates Wire Desk", icon `"newspaper"` (reuse that title and icon every week so it reads as the same recurring tool, not a new thing each time — pass `icon` only on the first publish of a given week's artifact, and omit it on any re-publish to that same file/URL). Tell the user to check what they want and hit Copy, then paste the result back into chat.

If the Artifact tool fails or isn't available for some reason, don't burn more than one retry on it — fall back to a plain numbered markdown list grouped by source instead, and ask the user which numbers to include (accepting shorthand like `"1,3,7"`, `"all from Avalara"`, `"skip the Kintsugi one"`).

Whichever form it takes, confirm your interpretation of their selection back to them if there's any ambiguity, and do not proceed to summarizing until they've responded. This is the first checkpoint.

## Step 4 — Summarize and format

Read `references/format.md` now if you haven't already this run.

If the user's reply came from the checklist's "Categorize" tab (a message shaped like "Include these numbered items in the newsletter, grouped as follows: <Category>: 1, 3 / <Category>: 2, 5 / ..."), use their category names as the section headers verbatim — don't invent your own theming, that decision is theirs now. If some items landed under "Uncategorized," ask them what section they belong in rather than guessing.

If instead they just gave you bare numbers with no categories (whether typed directly in chat or copied from the checklist's Select tab without visiting Categorize), do not invent thematic sections yourself and do not just list the articles in a chat message. Publish a second small Artifact showing only their selected items, one per row: numbered, headline hyperlinked to the source, source/date meta line, and a text input next to it for them to type a section name into — i.e. the same visual pattern as the checklist's Categorize tab (see `references/checklist-template.html` for the `.catrow` markup/style to reuse), just pre-scoped to their picks instead of the full shortlist. Give it a Copy button that produces the same "Include these numbered items in the newsletter, grouped as follows: <Category>: 1, 3 / ..." string. This is itself a checkpoint: wait for their categorized reply before writing any summaries or building the draft. Once they reply, treat their answer exactly like a Categorize-tab reply above.

Write a neutral, factual summary for each article — no opinion, no editorializing, just what happened and why it matters competitively. How long the summary is depends on the category it's in: see the "Summary length depends on the category" section in `references/format.md` — a category that reads as "top news" gets the full 2–4 sentence treatment, everything else gets a single active-voice sentence with the whole thing hyperlinked. This is easy to get backwards, so check the category name for every article before deciding which format to use, not just once at the top of a section.

Every section whose summaries you wrote yourself gets the italic line "All summaries in this section are AI-generated" at the end (only omit it for a section made entirely of user-written or verbatim text, which shouldn't normally happen here).

Fill in the HTML skeleton from `references/format.md` — masthead first (fixed content, only the date changes), then the section/article rows, using the sections, headlines (hyperlinked to the original articles), and summaries. Also draft a subject line: `Corporates Newsletter: <Month DD, YYYY>`.

## Step 5 — Review with the user (checkpoint 2)

Publish the draft as an Artifact (rendered HTML) so the user can see it formatted, not just as raw markup in chat — this also gives them something they can copy the rendered output from and paste into Outlook (pasting raw HTML tags as text is not what they want; pasting copied rich text from a rendered view preserves the formatting). Pass `Masthead.png` and `Footer.png` from the repository root via the Artifact tool's `files` parameter so the images render inline — see `references/format.md` for the exact `<img>` markup to use. Use the `title` parameter (e.g. "Corporates Newsletter Draft") and pass `icon` only on the first publish of this issue (e.g. `"mail"`); omit `icon` on every re-publish in the review loop below so it doesn't change. Also show the subject line suggestion.

Ask for corrections. Loop on their feedback — re-render and re-publish the same artifact URL each time — until they say it's good to go.

## Step 6 — Hand off the final draft

Once approved:

* Give the user the final HTML (they can copy it from the rendered Artifact) and the subject line.
* Remind them: open a new email in Outlook, paste the copied rendered content into the body, fill in the "To" field themselves (distribution list is their own responsibility), and send it themselves. This skill does not send email or touch Outlook directly — there is no email tool available in this environment.
