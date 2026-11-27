# LinkedIn Job Alert Automation (n8n)

**LinkedIn Job Alert Automation **: an n8n workflow that scrapes the latest LinkedIn jobs through Apify, standardizes the data, saves every job to Google Sheets, generates an AI market summary with Google Gemini, and returns the result to the caller through a webhook.

---

## Overview

A recruitment agency calls a single webhook URL with a job title, a location and a result limit. The workflow does the rest:

1. Starts an Apify LinkedIn jobs scraper run.
2. Waits for the run to finish.
3. Fetches the scraped dataset.
4. Cleans and standardizes each job record.
5. Saves each job to a Google Sheet.
6. Builds summary data and asks an AI Agent (Google Gemini) to write a short hiring-market summary.
7. Saves the summary to a second sheet and sends it back as the webhook response.

## Workflow Diagram

```
Webhook-Trigger (GET)
        |
        v
HTTP_POST_Apify scraper  -- start actor run
        |
        v
Wait
        |
        v
HTTP_Get  -- fetch dataset items
        |
        v
Data_Process_Job fields standardize (Code)
        |------------------------------------> Make Jobs save_Sheet (Google Sheets, append)
        v
Make Summary data (Code)
        |
        v
AI Agent  <--- Google Gemini Chat Model
        |
        v
Final summary format (Code)
        |
        v
Summary Sheet (Google Sheets, append)
        |
        v
Respond to Webhook
```

## Nodes

| # | Node | Type | Purpose |
|---|------|------|---------|
| 1 | `Webhook-Trigger` | Webhook (GET) | Entry point. Receives `jobTitle`, `location` and `limit` as query parameters. |
| 2 | `HTTP_POST_Apify scraper` | HTTP Request (POST) | Starts the Apify actor run with the search inputs. |
| 3 | `Wait` | Wait | Pauses so the Apify run can finish before fetching results. |
| 4 | `HTTP_Get` | HTTP Request (GET) | Fetches the dataset items from Apify (15 jobs in the test run). |
| 5 | `Data_Process_Job fields standardize` | Code | Normalizes each job into a fixed set of fields. |
| 6 | `Make Jobs save_Sheet` | Google Sheets (append) | Appends every standardized job as a row. |
| 7 | `Make Summary data` | Code | Aggregates all jobs into one item (companies, locations, counts) for the AI. |
| 8 | `AI Agent` + `Google Gemini Chat Model` | AI Agent | Writes a short summary of the hiring trends. |
| 9 | `Final summary format` | Code | Formats the AI output into the final summary record. |
| 10 | `Summary Sheet` | Google Sheets (append) | Stores the generated summary. |
| 11 | `Respond to Webhook` | Respond to Webhook | Returns the final response to the caller. |

## Data Saved to Google Sheets

**Jobs sheet** (one row per job):

| Column | Description |
|--------|-------------|
| Job Title | Title of the posting |
| Company | Hiring company |
| Location | Job location |
| Posted Time | When the job was posted |
| Job URL | Link to the LinkedIn job |
| Company URL | Link to the company page |

**Summary sheet** (one row per run): the AI-generated summary of top hiring companies, top locations and hiring trends.

## Requirements

- n8n (cloud or self-hosted)
- An [Apify](https://apify.com) account, API token and a LinkedIn jobs scraper actor
- A Google account with Google Sheets access (OAuth2 credential in n8n)
- A Google Gemini API key (credential in n8n)

## Setup

1. **Import the workflow** into n8n (Workflows > Import from File).
2. **Create credentials** in n8n:
   - Google Sheets OAuth2
   - Google Gemini (PaLM) API
3. **Prepare the Google Sheet** with two tabs, one for jobs and one for summaries, with the column headers listed above.
4. **Configure Apify:**
   - In `HTTP_POST_Apify scraper`, set the actor URL and your Apify token. Keep the token in an n8n credential or environment variable, never hard-coded in the workflow.
   - In `HTTP_Get`, use the dataset URL returned by the run.
5. **Select your spreadsheet and tabs** in `Make Jobs save_Sheet` and `Summary Sheet`.
6. **Select the Gemini credential** in `Google Gemini Chat Model`.
7. **Activate / Publish** the workflow to get the production webhook URL.

## Usage

Call the webhook with a GET request:

```
GET https://<your-n8n-domain>/webhook/<your-path>?jobTitle=Data Analyst&location=Dhaka&limit=15
```

| Parameter | Description | Example |
|-----------|-------------|---------|
| `jobTitle` | Job title to search | `Data Analyst` |
| `location` | Location to search | `Dhaka` |
| `limit` | Maximum number of jobs | `15` |

**PowerShell:**

```powershell
Invoke-RestMethod -Uri "https://<your-n8n-domain>/webhook/<your-path>?jobTitle=Data%20Analyst&location=Dhaka&limit=15"
```

**Expected result:**

- New rows appear in the jobs sheet.
- A new summary row appears in the summary sheet.
- The webhook responds with the final summary.

## Example Summary Output

> The most active hiring companies this run were X, Y and Z, with most roles based in A and B. Demand is concentrated in ... roles, suggesting ...

*(Replace with a real output from your test run.)*

## Known Limitations

- The `Wait` node uses a fixed delay. If the Apify run takes longer than the delay, the dataset fetch returns incomplete or empty data.
- There is no status check or failed-run handling between the scraper start and the dataset fetch.
- Long Apify runs can exceed the webhook response timeout of the caller.

## Suggested Improvements

- Replace the fixed `Wait` with a polling loop: `GET run status` > `IF SUCCEEDED` > else `Wait` and retry, with a maximum number of attempts.
- Add an error branch (or Error Trigger workflow) for `FAILED`, `ABORTED` and `TIMED-OUT` runs that notifies you and returns a clear error to the caller.
- Validate `jobTitle`, `location` and `limit` right after the webhook and return a 400 response when missing.
- Deduplicate jobs by Job URL before appending to the sheet.

## Tech Stack

n8n, Apify, Google Sheets, Google Gemini, JavaScript (Code nodes)

## Author

**ZS ROYAL ENGINEERING**: AI automation engineering
GitHub: [zsroyalengineering-dot](https://github.com/zsroyalengineering-dot)
