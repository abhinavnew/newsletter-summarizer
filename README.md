# Newsletter Summarizer Agent



An end-to-end automation that reads newsletters from Gmail, summarizes them using GPT-4.1, and delivers a structured daily digest to Telegram — with deduplication, category filtering, and a permanent log in Google Sheets.



Built on **self-hosted n8n (Docker)** with zero recurring infrastructure cost beyond the OpenAI API/Claude API.



---



## What It Does



Most newsletter readers suffer from the same problem: too many subscriptions, too little time. This agent solves it by:



1. Pulling newsletters from Gmail across **inbox and Promotions tab** (20+ active senders)

2. Stripping email HTML down to clean readable text

3. Sending each newsletter to GPT-4.1 with a strict summarization prompt

4. Formatting summaries by category and delivering them as a **Telegram digest**

5. Logging every processed newsletter to **Google Sheets** as an append-only archive

6. **Never re-summarizing** an email it has already processed — across any number of re-runs



---



## Architecture

Form Trigger (on-demand with date range + filters)

│

▼

Set Date Range → Read Sender Config (n8n Data Table)

Read Summaries Log (Google Sheets) → Build Dedup Set

│

▼

For each active sender (parallel):

Gmail Search (in:anywhere, full MIME body)

│

▼

Dedup: Skip Already-Seen Message IDs

│

▼

Strip HTML → Clean Text (capped at 12k chars)

│

▼

OpenAI GPT-4.1 → Structured JSON Summary

│

├── Format + Chunk → Send Telegram Digest (≤3900 chars/message)

└── Split Out → Append Rows to Google Sheets











**27 nodes** across trigger, config, dedup, fetch, clean, summarize, format, deliver, and log stages.



---



## Technical Highlights



### Gmail Full-Body Fetch

The Gmail API's default mode returns only a 200-character snippet. This workflow uses `format: raw` (MIME decode) to fetch the **complete email body** — enabling the LLM to extract actual article URLs, read full content, and produce accurate summaries rather than hallucinating from snippets.



### Deduplication Across Runs

Google Sheets serves as the state store. On every run, the workflow reads all previously logged `Gmail Message ID` values into an in-memory Set, then skips any email whose ID is already present. This makes re-runs and overlapping date ranges completely safe — no duplicate Telegram messages, no duplicate Sheets rows.



### HTML Cleaning Pipeline

Raw email HTML is stripped of `<style>` and `<script>` blocks, all tags removed, HTML entities decoded, whitespace normalized, and the result capped at 12,000 characters before it reaches the LLM. A separate regex pass extracts "view online" URLs from anchor text while filtering out unsubscribe, tracking, and redirect links.



### Structured LLM Output

OpenAI is called with `response_format: json_object` and a strict system prompt that:

- Prohibits adding context from training data (summaries contain only what the newsletter states)

- Requires flagging internal contradictions rather than silently resolving them

- Outputs a fixed 7-field JSON schema: category, author, date, 3–5 bullet summaries, worth-reading flag, and canonical article URL



### Telegram Message Chunking

Telegram enforces a 4096-character limit per message. The formatting node groups summaries by category, then splits the full digest into chunks of ≤3,900 characters at newsletter boundaries — so no summary is ever cut mid-way. The workflow sends N sequential messages if the digest is long.



### Sender Config via n8n Data Tables

Active senders are managed in an n8n-native data table (`newsletter_senders`) with columns for Gmail search query, display name, category hint, and active/paused flag. Adding or pausing a newsletter requires no workflow edits — just a table row change.



### Date Range Handling

The Gmail API's `before:` parameter is exclusive. A naive "May 3 only" query using `after:2026/05/03 before:2026/05/03` returns zero results. The workflow automatically adds one day to the end date to form the correct exclusive upper bound, regardless of what the user enters.



### Category and Author Filtering

An optional filter at the aggregation stage allows the user to request summaries for a specific category (e.g. "Emerging Tech") or author without re-fetching from Gmail — the full set is fetched and filtered in memory.



---



## Stack



| Layer | Tool |

|---|---|

| Workflow engine | n8n (self-hosted, Docker) |

| Email source | Gmail API (OAuth2, read-only scope) |

| LLM | OpenAI GPT-4.1 via HTTP Request node |

| Delivery | Telegram Bot API |

| State / archive | Google Sheets |

| Sender config | n8n Data Tables (SQLite) |

| Trigger | n8n Form Trigger (on-demand) + Scheduled Routine (daily) |



---



## Sender Coverage



20+ newsletters across five categories:



| Category | Examples |

|---|---|

| **Product Principles & Building** | Lenny's Newsletter, Elena Verna, Farnam Street, A16Z, Peter Yang |

| **Emerging Tech** | The Pragmatic Engineer, Dr Marily Nika, Jyoti Nookula |

| **PM Interview Prep** | Nancy Chu, Alex Rechevskiy |

| **Personal Finance** | Weekend Investing, Value Research, Smallcase |

| **Other** | Configurable per sender |



---



## Output Sample

RODUCT PRINCIPLES AND BUILDING



✦ Lenny's Newsletter — May 15, 2026

• The fastest-growing B2B products in 2026 share one trait: they embed into existing workflows rather than asking users to change behaviour.

• Lenny's analysis of 40 PLG companies shows median time-to-first-value under 8 minutes correlates with 2× better 30-day retention.

• The "expansion revenue first" model is replacing land-and-expand — companies now design for seat growth before closing the initial deal.

﻿📖﻿ Worth reading: Concrete benchmarks for PLG metrics rarely published outside paid research.

﻿🔗﻿ lennysnewsletter.com/p/









---



## Key Design Decisions



**Why n8n over a custom Python script?**

n8n provides built-in credential management (OAuth2 token refresh), a visual debugger for each node, and native connectors for Gmail, Telegram, and Google Sheets — eliminating ~500 lines of boilerplate auth and retry code. The workflow logic itself stays in readable JavaScript code nodes.



**Why Google Sheets as the dedup store?**

Zero additional infrastructure. The Summaries Log doubles as a human-readable archive the user can open and search in any browser. A dedicated database would add operational overhead with no meaningful performance benefit at this scale.



**Why GPT-4.1 over a smaller model?**

Newsletter content varies enormously in structure — plain-text essays, HTML-heavy promotional emails, multi-topic roundups. GPT-4.1 reliably extracts the author's primary argument and ignores decorative or promotional content. Smaller models produced inconsistent JSON and missed nuance in long-form newsletters during testing.



---



## Running It



Trigger on-demand via the n8n Form with optional `start_date`, `end_date`, `category`, and `author` parameters. Leave dates blank to default to the previous day.



A daily scheduled routine fires automatically and delivers the previous day's digest to Telegram each morning.



---



## Credentials Required



- Gmail OAuth2 (read-only scope)

- OpenAI API key

- Telegram Bot Token

- Google Sheets OAuth2



All credentials are stored in n8n's encrypted credential store — none are hard-coded in the workflow JSON.
