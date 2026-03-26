# Restaurant Finder and Caller

An n8n workflow that takes a natural language query (e.g. "best romantic restaurants with 4.9 rating near me"), finds matching restaurants using Firecrawl, extracts their phone numbers with GPT-4o-mini, and initiates outbound AI voice calls to each one via ElevenLabs and Twilio.

## How It Works

```
Chat message
  → Auto-detect location from IP
  → Build location-aware search query
  → Firecrawl web search (up to 8 results)
  → GPT-4o-mini extracts restaurant names + phone numbers
  → Validate and deduplicate E.164 phone numbers
  → For each restaurant:
      → 2s throttle
      → ElevenLabs outbound call (via Twilio)
      → Log result to Google Sheets
  → Return summary to chat
```

## Prerequisites

- n8n instance (self-hosted or cloud)
- [Firecrawl](https://firecrawl.dev) account
- [OpenAI](https://platform.openai.com) account
- [ElevenLabs](https://elevenlabs.io) account with a configured Conversational AI agent
- Twilio phone number linked to ElevenLabs
- Google account with a prepared Google Sheet (see setup below)

## Setup

### 1. Import the workflow

In n8n: **Workflows > Import from file** and select `n8n-workflow.json`.

### 2. Create credentials

Create the following credentials in n8n (**Settings > Credentials**):

| Credential name | Type | Header name | Value |
|---|---|---|---|
| Firecrawl API | HTTP Header Auth | `Authorization` | `Bearer <your_firecrawl_key>` |
| OpenAI API | HTTP Header Auth | `Authorization` | `Bearer <your_openai_key>` |
| ElevenLabs API | HTTP Header Auth | `xi-api-key` | `<your_elevenlabs_key>` |
| Google Sheets OAuth2 | Google Sheets OAuth2 API | — | Follow OAuth flow |

### 3. Set n8n variables

In n8n: **Settings > Variables**, add:

| Variable | Description |
|---|---|
| `ELEVENLABS_AGENT_ID` | Agent ID from your ElevenLabs Conversational AI dashboard |
| `TWILIO_FROM_NUMBER` | Your Twilio phone number in E.164 format (e.g. `+12025551234`) |
| `GOOGLE_SHEETS_ID` | The ID from your Google Sheet URL: `.../spreadsheets/d/<ID>/edit` |

### 4. Prepare the Google Sheet

Create a Google Sheet with a tab named **Calls Log** and the following headers in row 1:

```
Timestamp | Query | Location | Restaurant | Phone | Address | Call ID | Status | Error
```

### 5. Configure your ElevenLabs agent

The workflow passes three dynamic variables to the agent at call time:

| Variable | Value |
|---|---|
| `restaurant_name` | Name of the restaurant being called |
| `restaurant_address` | Address of the restaurant (if found) |
| `user_query` | The original user query |

Reference these in your agent prompt using `{{restaurant_name}}`, `{{restaurant_address}}`, and `{{user_query}}`.

## Usage

Open the chat interface on the workflow and type a natural language query:

```
best romantic restaurants with 4.9 rating near me
best sushi restaurants in Manhattan
top rated steakhouses in Austin TX
```

The workflow will:
1. Detect your location automatically (based on n8n server IP)
2. Search the web for matching restaurants
3. Extract and validate phone numbers
4. Call each restaurant with your ElevenLabs AI agent
5. Log every call attempt to Google Sheets
6. Return a summary with call IDs and status

## Workflow Nodes

| Node | Description |
|---|---|
| When chat message received | Chat trigger — entry point |
| Get Location from IP | Calls `ip-api.com` (free, no auth) to detect city and region |
| Build Search Query | Combines user query with detected location. Skips location injection if user already specified one |
| Firecrawl Search | Searches the web for up to 8 relevant results with full page markdown |
| Validate Search Results | Throws a descriptive error if Firecrawl returns no data |
| Extract Phone Numbers | Sends consolidated search content to GPT-4o-mini for structured extraction |
| Parse and Validate Phones | Validates E.164 format, deduplicates by phone number, strips invalid entries |
| Throttle Between Calls | 2-second wait between calls to avoid rate limiting |
| ElevenLabs Outbound Call | Initiates the outbound call via the ElevenLabs Twilio integration |
| Format Call Result | Normalises the API response — handles both success and failure without crashing |
| Log to Google Sheets | Appends one row per call attempt to the Calls Log sheet |
| Aggregate Results | Collects all per-restaurant results into a single item |
| Build Final Summary | Returns a structured summary to the chat with initiated/failed split |

## Edge Cases Handled

- Firecrawl API error or empty results: flow stops with a clear error message
- OpenAI extraction failure or malformed JSON: flow stops with a preview of the raw response
- Phone number not in E.164 format: entry is skipped and logged
- Duplicate phone numbers: deduplicated before calling
- ElevenLabs call fails for one restaurant: logged as `failed`, remaining restaurants continue
- Google Sheets logging fails: non-blocking, call flow continues unaffected
- IP geolocation fails: falls back to generic location string, flow continues

## Notes

- Location detection uses the public IP of the machine running n8n. For local deployments this is the user's IP. For cloud-hosted n8n, it will reflect the server's location.
- The workflow calls up to 8 restaurants per query (configurable in the Firecrawl Search node via the `limit` parameter).
- The 2-second throttle between calls can be adjusted in the **Throttle Between Calls** node.

## License

MIT
