# AI-Assisted Website Quality Auditor for Personal Services Websites

## Overview
This project audits the quality of Personal Services business websites
(tutoring, cleaning, fitness, consulting, etc.) by combining two distinct
sources of evidence:

1. **Automatically detected facts** — collected directly from each site's
   live HTML using Python.
2. **AI-generated judgment** — produced by the Gemini API, evaluating those
   facts against a fixed set of criteria.

The two are kept in separate fields throughout the pipeline so the final
report never presents an AI opinion as a verified fact.

## Websites Audited
- karachitutors.com
- dossaniplus.com
- fitathome.pk
- safaiwala.pk
- timesconsultant.com

Only public homepages were fetched — no login bypass, no aggressive
crawling, no vulnerability scanning.

## Methodology

### Step 1 — Define Evaluation Criteria
Six criteria were fixed in advance, each with a Professional / Basic /
Outdated rubric: Website Completeness, Contact & Business Information,
Call-to-Action (CTA), Social Media Presence, Mobile Responsiveness, and
Visual & Content Quality.

### Step 2 — Collect Automatically Detected Facts
For each URL, `requests` fetches the page and `BeautifulSoup` parses it to
extract:
- HTTP status code
- Total links found (and which are social-media links, matched against a
  fixed platform list: Facebook, Instagram, LinkedIn, YouTube, Twitter/X,
  TikTok)
- Number of `<form>` elements found (used as a contact-form signal)

These values are stored as plain data (`audit_website()` returns a dict)
with **no interpretation applied** — they are counts and status codes only.

### Step 3 — Generate AI Judgment
The collected facts (as JSON) are passed into a prompt sent to Gemini
(`generate_ai_judgment()`). The prompt explicitly instructs the model to:
- Score professionalism only from the facts given, not assumptions
- Not treat "site is reachable" as evidence of quality on its own
- Not assume a feature exists if it wasn't detected
- Return a fixed JSON structure: `professionalism_score`,
  `completeness`, `business_functionality`, `problems`,
  `missing_features`, `recommendations`, `priority`

The response is validated as JSON before use; malformed or empty
responses are caught and reported as errors rather than silently
accepted.

### Step 4 — Assemble the Report
Each finished report is a nested object with two top-level keys:
```json
{
  "website": "...",
  "automatically_detected_facts": { ... },
  "ai_generated_judgment": { ... }
}
```
This structure is what physically separates fact from judgment — nothing
from `automatically_detected_facts` is ever AI-written, and nothing in
`ai_generated_judgment` is presented as independently verified.

### Step 5 — Manual Review
Each AI-generated score was manually compared against the auditor's own
judgment of the same site, with a written reason for any adjustment
(e.g. AI score 80 vs. manual score 75 for karachitutors.com, with the
reasoning logged alongside the delta). This manual pass is a third,
clearly-labeled layer — distinct from both the automatic facts and the
raw AI output.

### Step 6 — Prompt Improvement
Based on the manual review, the prompt was revised (stricter rules
against over-scoring reachable-but-thin sites, explicit instruction not
to assume undetected features exist) and re-run on a test site to confirm
the scoring became more conservative and evidence-grounded.

## How Facts vs. AI Judgment Stay Separated
| | Source | Where it lives |
|---|---|---|
| **Facts** | Python + Requests + BeautifulSoup, deterministic | `automatically_detected_facts` |
| **AI Judgment** | Gemini API, generative | `ai_generated_judgment` |
| **Manual Review** | Human comparison of the two | `manual_review` log, kept separate from both |

The AI is never given the ability to edit or report the facts — it only
receives them as read-only input and returns judgment in its own,
clearly labeled block.

## Requirements
- Python 3, `requests`, `beautifulsoup4`, `pandas`
- Gemini API key (Google `genai` client)

