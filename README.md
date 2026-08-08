# AI Based Customer Feedback Analyzer

An n8n workflow that automatically collects customer feedback from a web form **and** Google Maps reviews, analyzes sentiment using Google Gemini, logs everything to Google Sheets, sends alerts on negative feedback, and reports weekly stats — with built-in error logging.

---

## What It Does

1. **Collects feedback from two sources**
   - A public web form (direct customer submissions)
   - Google Maps reviews for a specific business location (pulled via SerpApi on a schedule)
2. **Validates and normalizes** the incoming data into a common format
3. **Analyzes sentiment** of each piece of feedback using the Google Gemini API (returns sentiment, a numeric score, and a short summary)
4. **Logs results** to a Google Sheet (`Feedback Log` tab), matching on a unique `Review ID` so re-runs update existing rows instead of creating duplicates
5. **Sends alerts** (email) when feedback is negative
6. **Reports weekly stats** on a separate schedule
7. **Logs errors** from any node to a dedicated `Error Log` tab so failures are never silent

---

## Workflow Architecture

```
[Form Submission] ──────────────┐
                                  │
[Schedule Trigger] → [HTTP Request: SerpApi Google Maps Reviews]
                                  │
                          [Split Out] → [Normalize Google Review]
                                  │
                    ┌─────────────┴─────────────┐
                    ▼                            
            [Valid Feedback?] ──false──► [Log Error]
                    │true
                    ▼
      [Gemini - Analyze Sentiment] ──error──► [Log Error]
                    │
                    ▼
         [Parse Gemini Response]
                    │
                    ▼
      [Append or Update Row in Sheet] ──error──► [Log Error]
                    │
                    ▼
                 [If: negative?] ──true──► [Send a Message (email alert)]

[Schedule Trigger (weekly)] → [Get row(s) in sheet] → [Calculate Weekly Stats] → [Send a Message1]
```

### Key nodes

| Node | Purpose |
|---|---|
| `HTTP Request` | Calls SerpApi's Google Maps Reviews engine for a specific `place_id` |
| `Normalize Google Review` | Maps raw SerpApi fields into a consistent schema (`review_id`, `name`, `email`, `feedback_text`, `rating`, `source`, `timestamp`) |
| `On form submission` / `Normalize Input` | Same normalization for direct form submissions |
| `Valid Feedback?` | Filters out empty/invalid entries |
| `Gemini - Analyze Sentiment` | POSTs feedback text to the Gemini API for sentiment/score/summary |
| `Parse Gemini Response` | Parses the raw Gemini output into clean fields, merges with original data |
| `Append or update row in sheet` | Writes to Google Sheets, matched on `Review ID` (prevents duplicates) |
| `If` | Branches to send an alert email when sentiment is negative |
| `Log Error` | Catches error output from any critical node and appends details to `Error Log` |
| `Get row(s) in sheet` / `Calculate Weekly Stats` | Weekly summary report, triggered on its own schedule |

---

## Google Sheet Structure

**Spreadsheet:** `AI Based Customer Feedback Analyzer`

### Tab: `Feedback Log`
| Column | Description |
|---|---|
| Timestamp | When the feedback was created |
| Name | Reviewer/customer name |
| Email | Customer email (N/A for Google Maps reviews) |
| Feedback | Full feedback text |
| Source | `google_maps` or form source |
| Sentiment | `positive` / `neutral` / `negative` |
| Score | Numeric sentiment score |
| Summary | AI-generated one-line summary |
| Rating | Star rating (if available) |
| Review ID | Unique ID used for de-duplication (matching column) |

### Tab: `Error Log`
| Column | Description |
|---|---|
| Timestamp | When the error occurred |
| Workflow | Workflow name |
| Node | Node that failed |
| Error | Error message |

### Tab: `Form Responses 1`
Raw responses from the web form trigger.

---

## Setup Notes

### Google Maps location (`place_id`)
The SerpApi `HTTP Request` node targets a single business location via the `place_id` query parameter. To find a `place_id`:
1. Go to [serpapi.com/playground?engine=google_maps](https://serpapi.com/playground?engine=google_maps)
2. Search for the business by name + area
3. Copy the `place_id` from the `place_results` section of the response
4. Paste it into the `HTTP Request` node's `place_id` query parameter value

> **Note:** Changing this value switches which business's reviews are monitored. To monitor multiple locations simultaneously, duplicate the `HTTP Request` node with a different `place_id` and combine both outputs with a `Merge` node before `Split Out`.

### Review ID / de-duplication
The `Append or update row in sheet` node uses **"Append or Update Row"** with **Column to Match On = Review ID**. This is critical — without it, every run creates duplicate rows instead of updating existing ones.

For Google Maps reviews, `review_id` is derived from the review's unique `link` field (Google does not expose a plain `id` field in SerpApi's response):
```js
review_id: r.review_id || r.link || `${r.user?.name || 'unknown'}-${r.iso_date || index}`,
```

### Gemini API
The `Gemini - Analyze Sentiment` node calls:
```
https://generativelanguage.googleapis.com/v1beta/models/gemini-3.5-flash-lite:generateContent
```
Double-check this URL has no trailing characters (e.g. accidental `\`) — a malformed URL causes 404/502 errors that are easy to miss since the node still "succeeds" structurally.

### Error handling
Every critical node (`HTTP Request`, `Gemini`, `Append or update row in sheet`, `Get row(s) in sheet`, both `Send a message` nodes) has its `Error` output wired to the shared `Log Error` node, which appends `Timestamp / Workflow / Node / Error` to the `Error Log` tab. Check this tab first when troubleshooting.

---

## Testing Checklist

- [ ] Run `Execute workflow` — all nodes should show a green checkmark
- [ ] `Feedback Log` tab updates with no duplicate rows on repeated runs
- [ ] `Error Log` tab has no new entries after a clean run
- [ ] Negative feedback triggers an email alert
- [ ] Weekly stats trigger runs and sends a summary

---

## Status

✅ Form submission pipeline — working
✅ Google Maps reviews pipeline — working
✅ Sentiment analysis (Gemini) — working
✅ De-duplication via Review ID — working
✅ Error logging — working
✅ Weekly stats report — working
