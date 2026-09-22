# House style — Corporates Newsletter

This is the HTML skeleton and formatting rules for the weekly Corporates Newsletter. Fill it in during Step 4/5 of `SKILL.md`. The final artifact gets copied as rendered rich text and pasted into Outlook, so every rule below exists to survive that round-trip: table-based layout, inline styles only, no `<style>` blocks, no flexbox/grid/`position`, no external CSS or JS. Outlook's rendering engine (Word) only reliably honors inline styles on `<table>`/`<td>`/`<font>`-era markup.

## Colors

Pulled from `Masthead.png` / `Footer.png` so the HTML matches the fixed images exactly:

| Use | Hex |
|---|---|
| Masthead background (light mint) | `#E3F3EE` |
| Primary teal (section bars, headline accents) | `#0F6D63` |
| Secondary teal (links, meta text) | `#2C6E62` |
| Body text | `#222222` |
| Meta / muted text | `#666666` |
| Section divider | `#DDDDDD` |
| Footer light box | `#F5F5F5` |

## Embedding the masthead and footer images

`Masthead.png` and `Footer.png` live at the repository root and never change week to week. Don't try to base64-encode or shell out to embed them — there's no script execution here. Instead, pass them straight through the `Artifact` tool's `files` parameter on every publish call in Step 3/5 that needs them:

```
files: { "Masthead.png": "Masthead.png", "Footer.png": "Footer.png" }
```

Then reference them by that same relative path in the HTML `<img src>` — the Artifact tool serves them alongside the page, and the browser resolves them to absolute URLs when the user copies the rendered content, so the image keeps working after paste into Outlook (as a normal linked image, same as any marketing email — this is standard, not a bug):

```html
<img src="Masthead.png" width="640" alt="" style="display:block;width:100%;max-width:640px;height:auto;border:0;">
```

## Summary length depends on the category

Check the category name for **every** article before writing its summary — it's easy to default to one style for the whole draft and get later sections wrong.

- **Top-news-style category** (e.g. a section literally called "Top News," "This Week's Headlines," or similar — ask the user if unsure whether a category counts): full treatment, 2–4 neutral, factual sentences. Cover what happened, the companies involved, and why it matters competitively (e.g. what capability it adds, what market it targets, how it compares to adjacent players). Headline is hyperlinked; the summary text itself is plain (not linked).
- **Every other category**: one sentence, active voice, and the *entire sentence* — not just the company name — is the hyperlink. No separate headline/summary split for these rows.

Neutral and factual only — no opinion, no editorializing, no "impressively" / "unfortunately" / competitive spin. State what happened and, if relevant, the stated business reason (e.g. "to expand into the EU e-invoicing market"), not Claude's own read on whether it's good or bad for anyone.

## HTML skeleton

```html
<table role="presentation" width="100%" cellpadding="0" cellspacing="0" border="0" style="background-color:#E3F3EE;">
  <tr>
    <td align="center">
      <table role="presentation" width="640" cellpadding="0" cellspacing="0" border="0" style="max-width:640px;background-color:#ffffff;font-family:Arial,Helvetica,sans-serif;">

        <!-- Masthead -->
        <tr>
          <td style="position:relative;">
            <img src="Masthead.png" width="640" alt="" style="display:block;width:100%;max-width:640px;height:auto;border:0;">
          </td>
        </tr>
        <tr>
          <td style="padding:18px 24px 4px 24px;background-color:#E3F3EE;">
            <div style="font-size:24px;font-weight:bold;color:#0F6D63;">Corporates Newsletter</div>
            <div style="font-size:13px;color:#666666;margin-top:2px;">Week of {{DATE_RANGE}}</div>
          </td>
        </tr>

        <!-- ==== Repeat this block per section ==== -->
        <tr>
          <td style="padding:20px 24px 4px 24px;border-top:3px solid #0F6D63;">
            <div style="font-size:17px;font-weight:bold;color:#0F6D63;text-transform:uppercase;letter-spacing:0.5px;">{{SECTION_NAME}}</div>
          </td>
        </tr>

        <!-- Top-news-style article row (full treatment) -->
        <tr>
          <td style="padding:10px 24px 0 24px;">
            <a href="{{ARTICLE_URL}}" style="font-size:15px;font-weight:bold;color:#0F6D63;text-decoration:none;">{{HEADLINE}}</a>
            <div style="font-size:12px;color:#666666;margin:2px 0 6px 0;">{{SOURCE}} &middot; {{DATE}}</div>
            <div style="font-size:14px;color:#222222;line-height:1.5;">{{2-4 SENTENCE SUMMARY}}</div>
          </td>
        </tr>

        <!-- One-line article row (every other category) -->
        <tr>
          <td style="padding:8px 24px 0 24px;">
            <a href="{{ARTICLE_URL}}" style="font-size:14px;color:#2C6E62;text-decoration:underline;">{{ONE ACTIVE-VOICE SENTENCE, FULLY LINKED}}</a>
          </td>
        </tr>

        <!-- AI-generated note — one per section, after its last article row -->
        <tr>
          <td style="padding:8px 24px 18px 24px;">
            <div style="font-size:11px;color:#999999;font-style:italic;">All summaries in this section are AI-generated.</div>
          </td>
        </tr>
        <!-- ==== end repeatable section block ==== -->

        <!-- Footer -->
        <tr>
          <td>
            <img src="Footer.png" width="640" alt="" style="display:block;width:100%;max-width:640px;height:auto;border:0;">
          </td>
        </tr>

      </table>
    </td>
  </tr>
</table>
```

Notes on filling it in:

- `{{DATE_RANGE}}` — the 7-day window this issue covers, e.g. `September 15–22, 2026`.
- Repeat the section block once per category from the user's Categorize reply, in the order they gave (or ask if they didn't specify an order).
- Within a section, use the top-news row style only if the section itself is a "top news" style category (see above); otherwise use the one-line row style for every article in it.
- Drop the AI-generated note only for a section made entirely of verbatim/user-written text (shouldn't normally happen here).
- Subject line convention: `Corporates Newsletter: {{Month DD, YYYY}}` — use the issue date (the day you're sending), not the start of the 7-day window.
