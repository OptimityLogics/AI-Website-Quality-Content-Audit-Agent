# AI Website Quality & Content Audit Agent

An n8n workflow that crawls a website within a bounded scope, runs automated SEO, accessibility, content, and link checks, optionally asks Claude to review sampled page content for possible inconsistencies, and generates a PDF audit report.

## What it does

1. Receives an audit request through an n8n webhook.
2. Validates that the supplied start URL is a valid HTTP or HTTPS URL.
3. Crawls up to the configured page limit on the target host and checks `robots.txt` disallow rules for the wildcard user-agent.
4. Runs deterministic checks for:
   - **SEO:** missing or long page titles, missing or incorrectly sized meta descriptions, and missing viewport metadata.
   - **Accessibility:** missing HTML language, missing H1, multiple H1s, skipped heading levels, images without useful alt attributes, unnamed links, generic link text, and form fields without associated labels.
   - **Content:** placeholder text, thin content, and multiple contact email addresses.
   - **Links:** empty or placeholder links and broken internal/external link targets (up to 30 extra link checks per run).
5. Optionally sends excerpts from up to eight crawled pages to Claude for possible content inconsistencies. AI observations are explicitly unverified and do not replace rule-based findings.
6. Builds a report containing findings, evidence, suggested fixes, crawl notes, summary tables, and grouped GitHub issue drafts.
7. Converts the report to PDF, uploads it to Google Drive, and returns the report link and execution status in the webhook response.

## Workflow architecture

```text
Audit Webhook
  -> Audit Config
  -> Authorised Target?
       | true
       v
  Bounded Crawl & Rule Checks
       -> Build AI Prompt
       -> Use AI Review?
            | true  -> Ask Claude (Anthropic) --|
            | false ----------------------------|
                                                v
                                         Generate Report
                                                |
                                   Build GitHub Issue Drafts
                                                |
                                        Add Crawl Summary
                                                |
                                        Convert Report to PDF
                                                |
                                           Download PDF
                                                |
                                      Upload PDF Report to Drive
                                                |
                                      Respond - Audit Complete

Invalid target -> Stop - Not Authorised -> Respond - Not Authorised
Crawl failure  -> Respond - Audit Error
```

## Repository structure

```text
.
├── README.md
├── setup/
│   └── setupguide.md
└── workflow/
    └── AI-Website-Quality-Content-Audit-Agent.json
```

## Requirements

- An n8n instance with permission to create and execute workflows.
- Credentials configured in n8n for:
  - **PDF.co** – converts the generated HTML report to PDF.
  - **Google Drive OAuth2** – uploads the generated PDF to the selected Drive folder.
  - **Anthropic** – optional; used only when `useAI` is enabled.
- The required community/integration node packages available in the n8n instance:
  - `n8n-nodes-pdfco`
  - n8n LangChain Anthropic node package (`@n8n/n8n-nodes-langchain`)

## Quick start

1. Download `workflow/AI-Website-Quality-Content-Audit-Agent.json`.
2. In n8n, choose **Import from File** and import the JSON.
3. Open the PDF.co, Google Drive, and (if required) Anthropic nodes and select credentials created in your n8n instance.
4. Save the workflow.
5. Execute a test request using the test webhook URL. Example request:

```bash
curl -X POST "https://YOUR_N8N_HOST/webhook-test/website-audit" \
  -H "Content-Type: application/json" \
  -d '{
    "startUrl": "https://example.com",
    "maxPages": 10,
    "crawlDelayMs": 500,
    "useAI": false
  }'
```

See [setup/setupguide.md](setup/setupguide.md) for the full installation and configuration steps.

## Webhook request fields

| Field | Type | Default / limit | Description |
|---|---|---|---|
| `startUrl` | string | Required | Absolute HTTP or HTTPS URL to begin the audit. |
| `maxPages` | number | 10; capped at 25 | Maximum number of HTML pages to crawl. |
| `crawlDelayMs` | number | 500; minimum 250 ms | Delay between crawl requests and additional link checks. |
| `useAI` | boolean or string | `false` | Set to `true` to enable the optional Claude content review. |

## Important behaviour and limitations

- **Use only on websites you own or have explicit permission to audit.** The workflow's target-host comparison is derived from the submitted start URL; it is not an independently maintained domain allowlist or access-control mechanism.
- The crawler stays on the start URL's hostname for page discovery, honours parsed wildcard `robots.txt` disallow rules, and limits the crawl to 25 pages. It also performs at most 30 additional link-target checks per run.
- The checks are automated heuristics, not a complete SEO review, accessibility audit, or WCAG conformance assessment. Verify each finding manually.
- The optional AI review examines text samples from up to eight crawled pages and may be incorrect. Validate all AI-generated observations against the source pages.
- GitHub issue drafts are included in the report only. **The workflow does not create or publish GitHub issues and does not modify the audited website.**
- PDF generation and Drive upload depend on valid service credentials and the availability of those services.
- The imported workflow has credential references removed. Configure your own credentials after import; never commit API keys, OAuth tokens, or other secrets to this repository.
