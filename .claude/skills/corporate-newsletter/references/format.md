# House style — Corporates Newsletter

This is the HTML skeleton and formatting rules for the weekly Corporates Newsletter. Fill it in during Step 4/5 of `SKILL.md`. It's derived directly from `Corporates Newsletter format.html` at the repository root — a real past issue (exported from the newsletter's actual send platform) — so when in doubt, that file is the ground truth and this document is just a distilled version of it for filling in each week without re-reading 800 lines of markup.

The final artifact gets copied as rendered rich text and pasted into Outlook, so every rule below exists to survive that round-trip: table-based layout, inline styles only, no `<style>` blocks, no flexbox/grid/`position`, no external CSS or JS. Outlook's rendering engine (Word) only reliably honors inline styles on `<table>`/`<td>`/`<font>`-era markup.

## Colors

Sampled directly from `Masthead.png` and the footer images so the HTML matches them exactly:

| Use | Hex |
|---|---|
| Masthead / page background (light mint) | `#E3F3EE` |
| Dark green (masthead top bar, section headers) | `#123121` |
| Muted grey (date line, footer subscribe line) | `#7E8C8D` |
| Article link color | `#1155CC` |
| AI-generated note | `#999999` |

## Images: Masthead.png, FooterTop.png, FooterBottom.png

Three fixed images live at the repository root and don't change week to week:

- `Masthead.png` — the full masthead, **including the "Corporates Newsletter" title and tagline already baked into the image**. Don't add an HTML title overlay on top of it; that would duplicate the title. Just place the image and move on.
- `FooterTop.png` / `FooterBottom.png` — the footer's "Quick links and resources" box and the Thomson Reuters logo, split into two images with a gap between them. The gap exists on purpose: the subscribe line goes in that gap as real HTML text (see below), not baked into either image. Don't try to recombine them into one file.

**Embed all three as base64 `data:` URIs, not as Artifact `files` references.** An `<img src="Masthead.png">` served through the Artifact tool's `files` mechanism resolves to a URL scoped to that private artifact; when the rendered page is pasted into Outlook, Outlook has to fetch that URL to display the image, which can silently fail (no auth, no network path from its renderer) and break the pasted layout even though the artifact preview looked fine. `data:image/png;base64,...` puts the actual pixel bytes inline in the HTML, so nothing needs fetching — it survives copy/paste intact. If Bash/Python is available, base64-encode the three PNGs (e.g. `base64 -w0 Masthead.png`) and substitute the result into the `<img src="data:image/png;base64,...">` slots below; write the fully-substituted HTML to your scratchpad before publishing rather than inlining ~150KB of base64 directly in a tool call. If no script execution is available, fall back to the `files` parameter and flag to the user that pasted images may not survive into Outlook.

```html
<img src="data:image/png;base64,{{MASTHEAD_BASE64}}" width="650" alt="Corporates Newsletter" style="display:block;width:100%;max-width:650px;height:auto;border:0;">
```

## One section: "Corporate Tax and Trade"

This newsletter doesn't categorize — there is exactly one section, always named **"Corporate Tax and Trade"**, and every selected article goes under it in the order the user gave. Don't ask the user to categorize, don't invent additional sections, and don't split into multiple sections even if the articles cover different sub-topics (tax, legal, risk, etc.) — they all still go under the one section header.

## Article format is uniform

Every article uses the same format: one fully neutral, factual sentence, active voice, and the *entire sentence* (not just the company name) is the hyperlink. Cover what happened and, if relevant, the stated business reason (e.g. "to expand into the EU e-invoicing market") — no separate headline/summary split, no opinion, no editorializing, no "impressively" / "unfortunately" / competitive spin.

Occasionally the real reference combines two closely related stories into one linked paragraph with a line break between them (e.g. two personnel/product items from the same theme) — that's a judgment call for a genuinely tight pairing, not the default. Default to one article per line.

Separate articles with a blank spacer row (see skeleton below), matching the reference's `<p>&nbsp;</p>` spacer pattern. Set `font-family:Arial,Helvetica,sans-serif` explicitly on every text-carrying tag (`<span>`, `<div>`, `<a>`) rather than relying on it cascading down from the outer table — copy/paste into Outlook doesn't always preserve inherited styles reliably, only ones stated directly on the element.

## HTML skeleton

```html
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background-color:#E3F3EE;">
  <tr>
    <td align="center">
      <table role="presentation" width="650" cellpadding="0" cellspacing="0" border="0" style="max-width:650px;background-color:#ffffff;font-family:Arial,Helvetica,sans-serif;">

        <!-- Masthead (title is already baked into the image) -->
        <tr>
          <td>
            <img src="data:image/png;base64,{{MASTHEAD_BASE64}}" width="650" alt="Corporates Newsletter" style="display:block;width:100%;max-width:650px;height:auto;border:0;">
          </td>
        </tr>

        <!-- Date / confidentiality line -->
        <tr>
          <td style="padding:10px 24px;">
            <span style="font-family:Arial,Helvetica,sans-serif;font-size:13px;color:#7E8C8D;">{{ISSUE_DATE}} &nbsp;|&nbsp; This newsletter is confidential and strictly for internal purposes</span>
          </td>
        </tr>

        <!-- The one section: always "Corporate Tax and Trade" -->
        <tr>
          <td style="padding:16px 24px 6px 24px;">
            <div style="font-family:Arial,Helvetica,sans-serif;font-size:15px;font-weight:bold;color:#123121;">Corporate Tax and Trade</div>
          </td>
        </tr>

        <!-- Repeat per article -->
        <tr>
          <td style="padding:4px 24px 0 24px;">
            <a href="{{ARTICLE_URL}}" style="font-family:Arial,Helvetica,sans-serif;font-size:13px;color:#1155CC;text-decoration:none;">{{ONE ACTIVE-VOICE SENTENCE, FULLY LINKED}}</a>
          </td>
        </tr>
        <tr><td style="padding:8px 24px 0 24px;font-family:Arial,Helvetica,sans-serif;font-size:13px;">&nbsp;</td></tr>
        <!-- (repeat the two rows above for each additional article, omit the spacer after the last one) -->

        <!-- AI-generated note -->
        <tr>
          <td style="padding:10px 24px 18px 24px;">
            <div style="font-family:Arial,Helvetica,sans-serif;font-size:11px;color:#999999;font-style:italic;">All summaries in this section are AI-generated.</div>
          </td>
        </tr>

        <!-- Footer: quick-links image, then a real editable subscribe line, then the logo image -->
        <tr>
          <td>
            <img src="data:image/png;base64,{{FOOTERTOP_BASE64}}" width="650" alt="" style="display:block;width:100%;max-width:650px;height:auto;border:0;">
          </td>
        </tr>
        <tr>
          <td style="padding:14px 24px;text-align:center;">
            <a href="mailto:mci@thomsonreuters.com?subject=Corporates%20Newsletter%3A%20Subscription&body=Hey%2C%20I%20would%20like%20to%20subscribe%20to%20the%20Corporates%20Newsletter.%20Thanks%21" style="font-family:Arial,Helvetica,sans-serif;color:#7E8C8D;font-size:13px;text-decoration:none;">Click here to subscribe to the Corporates Newsletter</a>
          </td>
        </tr>
        <tr>
          <td>
            <img src="data:image/png;base64,{{FOOTERBOTTOM_BASE64}}" width="650" alt="" style="display:block;width:100%;max-width:650px;height:auto;border:0;">
          </td>
        </tr>

      </table>
    </td>
  </tr>
</table>
```

Notes on filling it in:

- `{{ISSUE_DATE}}` — the day you're sending, e.g. `September 22, 2026` (matches the reference's own date line, not the 7-day scan window).
- `{{MASTHEAD_BASE64}}`, `{{FOOTERTOP_BASE64}}`, `{{FOOTERBOTTOM_BASE64}}` — base64 of the three PNGs (see Images section above).
- The subscribe line is real text, not an image — keep it plain grey, no bold, no highlight color. It's meant to look identical to how it reads in `Corporates Newsletter format.html`.
- Drop the AI-generated note only if every article is verbatim/user-written text (shouldn't normally happen here).
- Subject line convention: `Corporates Newsletter - {{Month DD, YYYY}}` (matches the reference issue's own title, which used a dash, not a colon).
