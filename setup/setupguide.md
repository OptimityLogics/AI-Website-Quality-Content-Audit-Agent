# Setup Guide – AI Website Quality & Content Audit Agent

This guide explains how to import, configure, test, and run the n8n workflow.

## 1. Prerequisites

- Access to an n8n instance.
- Permission to crawl the website being audited.
- A PDF.co API account and API credential.
- A Google account that can authorize n8n to upload files to Google Drive.
- An Anthropic API credential if you plan to enable the optional AI content review.
- The required n8n nodes/packages:
  - PDF.co node: `n8n-nodes-pdfco`
  - Anthropic LangChain node: `@n8n/n8n-nodes-langchain`

Install community nodes using the method supported by your n8n deployment. Restart n8n if the installation method requires it.

## 2. Import the workflow

1. Download `workflow/AI-Website-Quality-Content-Audit-Agent.json` from this repository.
2. Open your n8n workspace.
3. Select **Workflows** and use **Import from File** (the menu wording may vary by n8n version).
4. Select the downloaded JSON file.
5. Open the imported workflow and review the nodes and connections.

Credential references and instance-specific workflow metadata have intentionally been removed from the repository JSON. You must select your own credentials in the relevant nodes.

## 3. Configure credentials

### PDF.co

1. Open **Convert Report to PDF**.
2. Create or select a PDF.co API credential in n8n.
3. Confirm the credential is selected for the node.
4. The node converts the generated HTML report into an A4 PDF.

### Google Drive

1. Open **Upload PDF Report to Drive**.
2. Create or select a Google Drive OAuth2 credential.
3. Complete the Google authorization flow.
4. Select the Drive and destination folder in the node. The imported workflow is configured for **My Drive** and its root folder by default; change this if required.
5. Confirm the account has permission to create files in the selected folder.

### Anthropic (optional)

1. Open **Ask Claude (Anthropic)**.
2. Create or select your Anthropic API credential.
3. Confirm the configured model is available to your account.
4. If you do not want AI-assisted content review, leave `useAI` set to `false` in the webhook request. The deterministic audit and report generation still run without this optional step.

Do not paste API keys directly into node parameters or workflow JSON. Store them in n8n credentials.

## 4. Understand the webhook input

The workflow starts with the **Audit Webhook** node:

- Method: `POST`
- Path: `website-audit`
- Response: controlled by response nodes at the end of the workflow

Send a JSON body with the following fields:

| Field | Required | Default | Notes |
|---|---|---|---|
| `startUrl` | Yes | — | Full website URL beginning with `http://` or `https://`. |
| `maxPages` | No | 10 | Maximum crawl size; workflow caps this at 25. |
| `crawlDelayMs` | No | 500 | Delay between requests; workflow enforces a minimum of 250 ms. |
| `useAI` | No | `false` | Boolean `true` or string `"true"` enables Claude review. |

Example:

```json
{
  "startUrl": "https://example.com",
  "maxPages": 10,
  "crawlDelayMs": 500,
  "useAI": false
}
```

Replace `https://example.com` with a site you own or are explicitly authorised to test.

## 5. Test the workflow

1. Save the workflow after configuring credentials.
2. Open **Audit Webhook** and click **Listen for Test Event** / **Execute Workflow**, depending on your n8n version.
3. Copy the displayed test webhook URL.
4. Send a POST request:

```bash
curl -X POST "https://YOUR_N8N_HOST/webhook-test/website-audit" \
  -H "Content-Type: application/json" \
  -d '{
    "startUrl": "https://example.com",
    "maxPages": 5,
    "crawlDelayMs": 500,
    "useAI": false
  }'
```

5. Review the execution in n8n. Check the crawl, report-generation, PDF conversion, download, and Google Drive upload nodes.
6. Confirm that the webhook response reports completion and contains a PDF report link.

For a production execution, activate the workflow and use the production webhook URL shown by n8n (normally `/webhook/website-audit` rather than `/webhook-test/website-audit`).

## 6. Optional AI content review

To enable AI-assisted review, send:

```json
{
  "startUrl": "https://example.com",
  "maxPages": 10,
  "crawlDelayMs": 500,
  "useAI": true
}
```

The workflow samples up to eight crawled pages and asks Claude to identify possible inconsistencies such as conflicting facts, naming, tone, contact details, prices, or dates. The model is instructed to use evidence from the supplied excerpts and not invent facts or quotations.

AI output is an unverified interpretation. Check the cited page content before taking action. If the AI node fails or returns no content, the report still indicates that AI review did not produce notes; rule-based findings are separate.

## 7. What the audit checks

The deterministic checks include:

- Page title and meta description presence/length.
- HTML language and viewport metadata.
- H1 presence/count and heading-level skips.
- Image alt attributes and filename-like alt text.
- Accessible link names and generic link text.
- Empty or placeholder link targets.
- Form controls without an associated label or accessible name.
- Placeholder copy and low-text pages.
- Broken link targets, within the configured additional-check limit.
- Duplicate page titles and multiple contact email addresses.

The workflow also reads `robots.txt` and skips paths matching the parsed wildcard-agent disallow rules.

## 8. Report output

The generated report includes:

- Target URL and crawl date.
- Pages crawled and extra link checks.
- Findings grouped by page and severity.
- Evidence and suggested fixes.
- Optional AI-assisted content review notes.
- Crawl notes and summary tables.
- Grouped GitHub issue drafts.

The GitHub issue drafts are report content only. This workflow does not publish issues to GitHub.

The PDF is uploaded to the configured Google Drive folder. The final webhook response includes a status and, when available, the PDF name and link.

## 9. Troubleshooting

| Issue | What to check |
|---|---|
| Workflow import reports a missing node | Install the required PDF.co or Anthropic node package, then reload n8n. |
| PDF conversion fails | Confirm the PDF.co credential, API quota, and node configuration. |
| Drive upload fails | Reconnect Google Drive OAuth2 and verify destination-folder permissions. |
| AI review fails | Verify the Anthropic credential and model availability, or set `useAI` to `false`. |
| Audit returns a blocked response | Check that `startUrl` is a valid absolute HTTP(S) URL. |
| Some pages are not crawled | The workflow has a 25-page cap, respects parsed robots.txt rules, and skips non-HTML resources. |
| Some links are not checked | The workflow checks at most 30 additional link targets per run. |

## 10. Security and responsible use

- Only audit websites you own or have written permission to test.
- Respect the website's robots.txt rules and applicable terms.
- Keep API keys and OAuth credentials in n8n's credential store.
- Do not commit credentials, tokens, private URLs, or execution data.
- Treat automated and AI-generated findings as recommendations that require human verification.
- The workflow's target-host check is not a true domain allowlist. It derives the comparison host from the submitted URL itself; do not rely on it as an authorization or security boundary.

---
