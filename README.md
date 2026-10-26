# Hire-Mind# HireMind – AI Resume Screening Automation

An [n8n](https://n8n.io) workflow that screens resumes against a job description using the Groq API. A human recruiter approves every outcome.

## How it works

1. **Trigger:** a new PDF resume is uploaded to a Google Drive folder (polled every minute).
2. **Extract:** the resume is downloaded and its text extracted. The candidate's name and email are pulled from the text.
3. **AI screening:** the resume and the job requirements (must-have and nice-to-have) are sent to Groq (`openai/gpt-oss-120b`), which returns structured JSON: per-requirement status (`met` / `partial` / `not_met`), evidence quotes, a summary, and flags.
4. **Hallucination guard:** every evidence quote is checked against the resume text. Any quote not found is downgraded to `not_met` and flagged.
5. **Tiering in code:** the candidate is tiered `strong`, `potential` or `weak` from the must-have results, not by the model.
6. **ATS score:** a separate 0-100 score covers keywords (60), resume sections (20), contact info (10) and readability (10). It is advisory only.
7. **Log:** results are appended to a Google Sheet.
8. **Approval:** a Switch node routes by tier (strong / weak / potential). Each branch emails the recruiter an *Approve / Disapprove* request and waits.
9. **Candidate email:** when the recruiter approves, a message is sent to the candidate's email.

## Tech stack

n8n, Groq API, Google Drive, Google Sheets, Gmail, JavaScript (Code nodes)

## Full setup guide

### Prerequisites

- A Google account (Drive, Sheets, Gmail)
- A free [Groq](https://console.groq.com) account
- n8n, either:
  - **n8n Cloud:** sign up at n8n.io (free trial), or
  - **Self-hosted:** install Node.js 18+ and run `npx n8n` (opens at `http://localhost:5678`). Docker also works: `docker run -it --rm -p 5678:5678 n8nio/n8n`.

### Step 1: Prepare Google Drive and Sheets

1. In Google Drive, create a folder named **HireMind Resumes**. Resumes must be PDFs.
2. In Google Sheets, create a spreadsheet named **HireMind Results**.
3. In row 1 of the first sheet, add these headers exactly (lowercase, one per column):

   ```
   file_name | candidate_summary | tier | must_haves_met | explanation | flags | missing_info | processed_at | ats_score | ats_rating | ats_missing_keywords | ats_format_issues
   ```

### Step 2: Get a Groq API key

1. Go to [console.groq.com](https://console.groq.com) > **API Keys** > **Create API Key**.
2. Copy the key. It is shown only once. Never commit it to GitHub.

### Step 3: Import the workflow

1. In n8n, click **Create workflow**.
2. Open the **⋯** menu (top right) > **Import from file** and select `HireMind.json`.

### Step 4: Create credentials in n8n

Go to **Credentials > Create credential** and add:

| Credential | How to set up |
|---|---|
| **Groq** | Paste the API key from Step 2 |
| **Google Drive OAuth2** | Click *Sign in with Google* (n8n Cloud), or create OAuth client ID and secret in Google Cloud Console (self-hosted) |
| **Google Sheets OAuth2** | Same as above |
| **Gmail OAuth2** | Same as above |

Self-hosted users: in [Google Cloud Console](https://console.cloud.google.com), create a project, enable the **Drive, Sheets and Gmail APIs**, configure the OAuth consent screen (add yourself as a test user), then create an *OAuth client ID* (Web application). Use the redirect URL shown in the n8n credential screen.

### Step 5: Connect the nodes

Open each node below, pick the matching credential, and replace the placeholders:

| Node | What to set |
|---|---|
| **Google Drive Trigger** | Credential, and the folder to watch: **HireMind Resumes** |
| **Download Resume** | Google Drive credential |
| **Call Groq** | Groq credential |
| **Append row in sheet** | Google Sheets credential, document **HireMind Results**, first sheet |
| **Send message and wait for response** (3 nodes) | Gmail credential; replace `YOUR_EMAIL@gmail.com` with the recruiter's email |
| **Send a message** (3 nodes) | Gmail credential; edit the email text as needed |

### Step 6: Set the job description

Open the **Job Description** node and edit:

- `jd_text`: a short role description
- `must_haves`: comma-separated required skills, e.g. `React, JavaScript, TypeScript, Git`
- `nice_to_haves`: e.g. `Tailwind CSS, testing experience`

The candidate emails in the **Send a message** nodes mention the role title, so update them too.

### Step 7: Test

1. Click **Execute workflow** (or activate it), then upload a sample PDF resume to the Drive folder.
2. Within about a minute, check that:
   - a new row appears in **HireMind Results**
   - an *Approve shortlist?* email arrives in the recruiter's inbox
3. Click **Approve** in the email. The candidate email is sent to the address found in the resume. **Test with your own resume first.**

### Step 8: Go live

Toggle the workflow to **Active** (top right). It will now process every PDF added to the folder. Self-hosted n8n must stay running.

## Troubleshooting

| Problem | Fix |
|---|---|
| Trigger doesn't fire | Workflow must be Active; confirm the correct folder is selected and the file is new |
| Empty resume text | The PDF is scanned or image-only; use a text-based PDF |
| Sheet row not added | Headers must match the column names exactly |
| `invalid_json` error | The model returned malformed JSON; re-run the execution |
| Google auth error | Re-connect the credential; for self-hosted, check the redirect URL and that you are a test user |
| Groq `401` or `429` | Wrong API key, or rate limit reached; wait and retry |

## Design notes

- The resume is treated as data, not instructions, to resist prompt injection.
- The prompt tells the model to ignore name, gender, age and nationality, and not to penalize employment gaps.
- Temperature is 0.1 for consistent output, and resume text is capped at 12,000 characters.
- The tool assists screening only. It does not make hiring decisions.
- Never commit API keys, credential IDs or personal email addresses. This repo's JSON uses `YOUR_*` placeholders.

## Author

Karthik Sudhir Prabhu: [GitHub](https://github.com/Karthikprabhu07) · [LinkedIn](https://linkedin.com/in/karthik-prabhu-aba603330)
