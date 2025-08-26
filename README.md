# AI Tweet Agent — n8n Workflows

Two production-ready n8n workflows that **generate** tweet ideas daily and **auto-post** an approved tweet to X (Twitter) every morning.

- **Workflow A — Tweet Poster (frontstage)**: Finds an *Approved* & *Pending* tweet in Airtable, posts to X, and marks it as *Posted*.
- **Workflow B — Tweet Generator (backstage)**: Uses an AI model to draft tweets from current news, cleans/parses JSON, and creates Airtable records for review.

---

## 🧱 Architecture

```
[Schedule Trigger]
      │
      ▼
[Search Airtable (Approved + Pending)]
      │
      ▼
[Create Tweet (X)]
      │
      ▼
[Update Airtable -> Posted]
```

```
[Schedule Trigger]
      │
      ▼
[Message a Model (Perplexity)]
      │
      ▼
[Code (JSON clean + parse)]
      │
      ▼
[Create Record (Airtable) -> Pending Review]
```

---

## 📦 Files in this repo

```
/workflows/
  ├─ Tweet-Poster.json
  └─ Tweet-Generator.json
README.md
```

> Drop the JSON exports above into **n8n → Import from file**.

---

## 🔑 Requirements

- **n8n** (self-hosted or cloud)
- **Airtable** base + table (`Tweets`) with fields:
  - `Name` (string)
  - `Tweet` (long text)
  - `Status` (options: Pending Review, Approved, Rejected)
  - `Posted` (options: Pending, Posted)
  - `Created At` (date)
- **X (Twitter) API** credentials (OAuth2)
- **Perplexity (or your LLM)** API key *(for Generator only)*

---

## 🔐 Credentials Mapping (n8n)

- `airtableTokenApi`: Airtable Personal Access Token
- `twitterOAuth2Api`: X (Twitter) OAuth2 App
- `perplexityApi`: Perplexity API Key

> Store secrets in **n8n Credentials**. Do **not** hardcode.

---

## ⚙️ Workflow A — Tweet Poster

**Flow:** Schedule → Airtable Search → Create Tweet (X) → Update Airtable

- **Schedule**: run daily (set IST-friendly time, e.g., 09:00)
- **Airtable Search**:
  - **Filter formula**:
    ```
    AND({Status} = 'Approved', {Posted} = 'Pending')
    ```
  - **Sort**: by `Created At` (ascending) so you post the oldest approved first
- **Create Tweet (X)**:
  - **Text**: `={{ $json.Tweet }}`
- **Airtable Update**:
  - Set `Posted` = `Posted` and preserve other fields from the search result

**High-level JSON (excerpt):**
```json
{
  "nodes": [
    {"type": "n8n-nodes-base.scheduleTrigger", "name": "Schedule Trigger"},
    {"type": "n8n-nodes-base.airtable", "name": "Search records", "parameters": {"operation":"search","filterByFormula":"AND({Status} = 'Approved', {Posted} = 'Pending')"}},
    {"type": "n8n-nodes-base.twitter", "name": "Create Tweet", "parameters": {"text":"={{ $json.Tweet }}"}},
    {"type": "n8n-nodes-base.airtable", "name": "Update record", "parameters": {"operation":"update"}}
  ]
}
```

---

## 🤖 Workflow B — Tweet Generator

**Flow:** Schedule → AI (Perplexity) → Code (clean/parse) → Airtable Create

- **Schedule**: run daily or multiple times/day
- **AI Message (Perplexity)**: prompt the model to return **EXACTLY a JSON array of 5 strings** under 280 chars
- **Code Node (TypeScript/JS)**: cleans triple backticks and parses JSON; outputs records for Airtable Create
  
**Code Node (drop-in script):**
```javascript
// Handle Perplexity output
let output = $json;

// If the HTTP node gave an array
if (Array.isArray(output)) {
  output = output[0];
}

let content = output?.choices?.[0]?.message?.content || "";

// Remove ```json ... ``` wrappers
content = content.replace(/```json|```/g, "").trim();

let tweets;
try {
  tweets = JSON.parse(content);
} catch (e) {
  throw new Error("Failed to parse tweets JSON: " + e.message + "\nContent: " + content);
}

return tweets.map(t => ({ 
  json: {
    Name: t.substring(0, 50),         // First 50 chars for Name field
    Tweet: t,                         // Full tweet
    Status: "Pending Review",         // Review pipeline
    Posted: "Pending",                // Default state before posting
    Created_At: new Date().toISOString()
  }
}));
```
  
- **Airtable Create**: map fields (`Name`, `Tweet`, `Status`, `Created At`)

**Field mapping tips:**
```text
Name         ->  ={{ $json.Tweet.slice(0, 50) }}
Tweet        ->  ={{ $json.Tweet }}
Status       ->  ={{ $json.Status }}
Created At   ->  ={{ $json.Created_At.substring(0,10) }}
```

---

## 🌐 External Services Used

- **Airtable** — data store & review queue
- **X (Twitter) API** — posting endpoint
- **Perplexity (or any LLM)** — generates draft tweets from live news
- *(Optional)* Slack/Email — send success/failure notifications

---

## 🧪 Testing

1. Run **Generator** first → confirm 5 new records appear in Airtable with `Status = Pending Review`.
2. Manually set a few to **Approved**.
3. Run/post with **Poster** → confirm X post + Airtable row updates to `Posted`.

---

## 🛡️ Production Notes

- Respect X rate limits and content policies.
- Add error branches and retries for API calls.
- Log run IDs to a sheet or database for observability.
- Keep posts ≤280 chars; trim/validate before posting.
- Timezone: set cron to IST (or convert to UTC in n8n).

---

## 📄 License

MIT — use at your own risk; no warranties.

---

## 🙋 Support

Create an issue on the repo or DM me on LinkedIn.
