# Resume Analyzer API (Express + Gemini)

Production-ready Node.js backend for an AI-powered Resume Analyzer SaaS. Accepts a resume (plain text) + a job description, calls Google Gemini (`gemini-1.5-flash`) to analyze/rewrite, and returns **strict structured JSON**.

## Requirements

- Node.js 18+
- A Google Gemini API key

## Setup (local)

```bash
cd backend
npm install
cp .env.example .env
```

Set `GEMINI_API_KEY` in `.env`.

Run locally:

```bash
npm run dev
```

Health check:

```bash
curl -s http://localhost:8080/healthz
```

## API

### POST `/analyze`

Request body (JSON):

```json
{
  "resumeText": "string",
  "jobDescription": "string"
}
```

Example:

```bash
curl -sS -X POST "http://localhost:8080/analyze" \
  -H "Content-Type: application/json" \
  -d '{
    "resumeText": "Software Engineer\\n- Built REST APIs in Node.js\\n- Improved CI pipelines",
    "jobDescription": "We need a Node.js engineer with REST API experience, CI/CD, and cloud deployment."
  }' | jq .
```

Successful response schema:

- `matchScore`: number (0–100)
- `overallFit`: `"Poor" | "Fair" | "Good" | "Great"`
- `missingKeywords`: `string[]`
- `matchedKeywords`: `string[]`
- `rewrittenBullets`: `{ original, improved, rationale }[]`
- `atsWarnings`: `string[]`
- `suggestions`: `string[]`
- `roleSeniority`: `"Junior" | "Mid" | "Senior"`

Errors:

- `400` validation errors (missing inputs)
- `502` Gemini failures or malformed AI output

### POST `/parse`

Parses resume text into sectioned, schema-validated JSON. This endpoint does not require a job description.

Request body (JSON):

```json
{
  "resumeText": "string",
  "inputType": "file | text | linkedin",
  "fileName": "optional-string"
}
```

Successful response:

```json
{
  "success": true,
  "data": {
    "version": "2",
    "source": {
      "inputType": "text",
      "rawText": "original resume text",
      "importedAt": "2026-02-07T00:00:00.000Z",
      "parsedAt": "2026-02-07T00:00:00.000Z",
      "parser": "gemini-section-parser-v2"
    },
    "resumeData": {
      "basics": {},
      "work": [],
      "education": [],
      "projects": [],
      "awards": [],
      "skills": {},
      "languages": []
    },
    "sections": [
      {
        "id": "section-1",
        "title": "Header",
        "kind": "header",
        "canonicalTarget": "none",
        "lines": ["Jane Doe"]
      }
    ],
    "sectionPresence": {
      "summary": false,
      "work": true,
      "projects": true,
      "skills": true,
      "education": true,
      "awards": false,
      "languages": true
    },
    "customSections": [],
    "notes": []
  }
}
```

Behavior:

- Primary parser uses Gemini with strict JSON output.
- Gemini is the only parse engine. If Gemini output is malformed or fails schema validation after retries, the request fails.
- Fallback usage is surfaced in `data.notes`.

Errors:

- `400 INVALID_INPUT`
- `500 PARSE_FAILED` (when Gemini parsing fails)

## Environment variables

- `GEMINI_API_KEY` (required): Gemini API key
- `PORT` (optional): defaults to `8080` (Cloud Run standard)

## Docker (Cloud Run ready)

Build and run locally:

```bash
cd backend
docker build -t resume-analyzer-api .
docker run --rm -p 8080:8080 -e GEMINI_API_KEY="YOUR_KEY" resume-analyzer-api
```

## Deploy to Google Cloud Run

Prereqs:

- `gcloud` installed and authenticated
- A GCP project and billing enabled

Set variables:

```bash
PROJECT_ID="YOUR_PROJECT_ID"
REGION="us-central1"
SERVICE_NAME="resume-analyzer-api"
```

Enable APIs:

```bash
gcloud services enable run.googleapis.com artifactregistry.googleapis.com cloudbuild.googleapis.com --project "$PROJECT_ID"
```

Build & deploy from source (Cloud Build):

```bash
gcloud run deploy "$SERVICE_NAME" \
  --source . \
  --region "$REGION" \
  --project "$PROJECT_ID" \
  --allow-unauthenticated \
  --set-env-vars GEMINI_API_KEY="YOUR_KEY"
```

Then call the service:

```bash
SERVICE_URL="$(gcloud run services describe "$SERVICE_NAME" --region "$REGION" --project "$PROJECT_ID" --format='value(status.url)')"
curl -sS -X POST "$SERVICE_URL/analyze" -H "Content-Type: application/json" -d '{"resumeText":"...","jobDescription":"..."}'
```
