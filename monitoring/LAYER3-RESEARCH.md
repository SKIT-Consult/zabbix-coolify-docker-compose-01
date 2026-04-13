# Layer 3: External Services API Research

Research date: 2026-04-13

---

## 1. Bunny Stream API

### Authentication
- Header: `AccessKey: {YOUR_STREAM_API_KEY}`
- Key found in: bunny.net dashboard > Video Library > API section
- Two key types: Account API key (for library management) and Library API key (per-library, for video operations)

### Key Endpoints

**List Video Libraries (account-level)**
```
GET https://api.bunny.net/videolibrary
Headers: AccessKey: {ACCOUNT_API_KEY}
Params: page, perPage (5-1000), search
```
Response includes per library: `VideoCount`, `TrafficUsage`, `StorageUsage`, library name, dates.

**Get Single Library**
```
GET https://api.bunny.net/videolibrary/{id}
Headers: AccessKey: {ACCOUNT_API_KEY}
```
Same fields as above for a single library.

**List Videos in Library**
```
GET https://video.bunnycdn.com/library/{libraryId}/videos
Headers: AccessKey: {LIBRARY_API_KEY}
Params: page (default 1), itemsPerPage (default 100), search, collection, orderBy
```
Response: `totalItems`, `currentPage`, `itemsPerPage`, `items[]`.
Each video has: `status` (0=Created, 1=Uploaded, 2=Processing, 3=Transcoding, 4=Finished, 5=Error), `encodeProgress` (int 0-100), `transcodingMessages[]`.

**Get Single Video**
```
GET https://video.bunnycdn.com/library/{libraryId}/videos/{videoId}
Headers: AccessKey: {LIBRARY_API_KEY}
```

### Rate Limits
- Not explicitly documented for Stream/Management API
- Returns HTTP 429 when exceeded; implement exponential backoff
- CDN-level rate limiting configurable via Bunny Shield (separate feature)

### Zabbix vs n8n

| What to Monitor | Zabbix (HTTP Agent) | n8n Needed? |
|---|---|---|
| Library video count, storage | YES - single GET, parse JSON | No |
| Video encoding errors | YES - GET videos, filter status=5 | No for count; n8n for alerts with details |
| Storage growth trend | YES - store StorageUsage, Zabbix calculates trend | No |

**Simplest health check:** `GET https://api.bunny.net/videolibrary` - returns 200 with library list. Parse `VideoCount` and `StorageUsage` for library 477336.

**Zabbix implementation:**
- HTTP Agent item: GET `https://api.bunny.net/videolibrary/{id}` with AccessKey header
- Dependent items via JSONPath: `$.VideoCount`, `$.StorageUsage`, `$.TrafficUsage`
- Trigger: video count drops (delta < 0), storage exceeds threshold

---

## 2. Mailjet Statistics API

### Authentication
- HTTP Basic Auth: `{MJ_APIKEY_PUBLIC}:{MJ_APIKEY_PRIVATE}`
- Base URL: `https://api.mailjet.com/v3/REST/`

### Key Endpoints

**API Key Totals (simplest health check + usage)**
```
GET https://api.mailjet.com/v3/REST/apikeytotals
Auth: Basic {public}:{private}
```
Returns lifetime totals for the API key: messages sent, delivered, bounced, etc.

**Stat Counters (flexible stats)**
```
GET https://api.mailjet.com/v3/REST/statcounters
Params:
  CounterSource=APIKey
  CounterTiming=Message (by send time) or Event (by event time)
  CounterResolution=Lifetime|Day|Hour
  FromTS=2026-04-12T00:00:00Z
  ToTS=2026-04-13T00:00:00Z
```
Returns: MessageSentCount, MessageDeliveredCount, MessageOpenedCount, MessageClickedCount, MessageBouncedCount, MessageSpamCount, MessageBlockedCount, MessageUnsubscribedCount.

**Bounce Statistics**
```
GET https://api.mailjet.com/v3/REST/bouncestatistics
```
Returns bounce details: bounce timestamp, campaign ID, contact ID, permanent vs transient.

**Message Sent Statistics**
```
GET https://api.mailjet.com/v3/REST/messagesentstatistics
```
Summary of message statuses and events.

### Zabbix vs n8n

| What to Monitor | Zabbix (HTTP Agent) | n8n Needed? |
|---|---|---|
| Delivery rate (sent vs delivered) | YES - apikeytotals or statcounters | No |
| Bounce count/rate | YES - statcounters or bouncestatistics | No |
| Spam complaints | YES - statcounters MessageSpamCount | No |
| Daily send volume | YES - statcounters with Day resolution | No |
| Detailed bounce analysis | Possible but complex | n8n better for digest |

**Simplest health check:** `GET https://api.mailjet.com/v3/REST/apikeytotals` - returns 200 if credentials valid. Contains lifetime delivery stats.

**Zabbix implementation:**
- HTTP Agent item with Basic auth: GET `apikeytotals`
- Dependent items: `$.Data[0].MessageSentCount`, `$.Data[0].MessageDeliveredCount`, `$.Data[0].MessageBouncedCount`
- HTTP Agent item: GET `statcounters?CounterSource=APIKey&CounterTiming=Message&CounterResolution=Day&FromTS={now-1d}`
- Triggers: bounce rate > 5%, spam complaints > 0, delivery rate drops below 95%

---

## 3. Paddle API

### Authentication
- Bearer token: `Authorization: Bearer pdl_live_...`
- Base URL: `https://api.paddle.com`
- API version specified via header, not path

### API Key Expiry
- Keys have expiry dates (default: 90 days from creation, max: 1 year)
- Key prefix: `pdl_` (since May 2025)
- Expired keys cannot be revalidated
- Webhook event `api_key.expiring` fires before expiry
- **No API endpoint to check key expiry date** - must track manually or use webhook

### Key Endpoints

**List Event Types (lightest validation call)**
```
GET https://api.paddle.com/event-types
Headers: Authorization: Bearer {API_KEY}
```
Returns list of event types. Validates key is active (200 = valid, 401 = invalid/expired).

**List Events (recent activity)**
```
GET https://api.paddle.com/events
Headers: Authorization: Bearer {API_KEY}
Params: per_page (default 50, max 200)
```
Returns events from last 90 days. Can verify subscription/transaction activity.

**List Transactions**
```
GET https://api.paddle.com/transactions
Headers: Authorization: Bearer {API_KEY}
Params: per_page, status, created_at[gte], created_at[lte]
```

### Key Expiry Detection Strategy
1. **Webhook approach (recommended):** Subscribe to `api_key.expiring` event
2. **Polling approach:** Call any endpoint; 401 = expired. But this only detects AFTER expiry.
3. **Manual tracking:** Record key creation date, calculate expiry (creation + 90 days default)

### Zabbix vs n8n

| What to Monitor | Zabbix (HTTP Agent) | n8n Needed? |
|---|---|---|
| API key validity | YES - GET event-types, check HTTP code | No |
| Recent transactions | YES - GET transactions, parse count | No |
| Key expiry warning | NO - no API to check expiry date | n8n webhook receiver |
| Transaction anomalies | Possible but complex | n8n better |

**Simplest health check:** `GET https://api.paddle.com/event-types` with Bearer token. HTTP 200 = key valid, 401 = key expired/invalid.

**Zabbix implementation:**
- HTTP Agent item: GET `https://api.paddle.com/event-types` with Bearer auth header
- Check HTTP response code (200 vs 401)
- Trigger: response code != 200 -> CRITICAL "Paddle API key invalid/expired"
- For expiry warning: calculate days remaining from known creation date using a Zabbix calculated item, or use n8n webhook to receive `api_key.expiring` event

---

## 4. n8n Executions API

### Authentication
- Header: `X-N8N-API-KEY: {API_KEY}`
- Create key in: n8n UI > Settings > API
- Must enable API via environment variable: `N8N_PUBLIC_API_ENABLED=true` (may already be set)
- Base URL: `http://{n8n-host}:{port}/api/v1`

### Key Endpoints

**List Executions**
```
GET /api/v1/executions
Headers: X-N8N-API-KEY: {key}
Params:
  status=error          (allowed: canceled, error, success, waiting)
  workflowId={id}       (filter to specific workflow)
  limit=10              (results per page)
  cursor={next_cursor}  (pagination)
```
Response includes: execution id, status, startedAt, stoppedAt, workflowId, finished (bool), mode.

**Get Single Execution**
```
GET /api/v1/executions/{id}
Headers: X-N8N-API-KEY: {key}
```
Returns full execution data including node outputs and error messages.

**List Workflows**
```
GET /api/v1/workflows
Headers: X-N8N-API-KEY: {key}
```
Health check - validates API key and n8n is responsive.

### Zabbix vs n8n

| What to Monitor | Zabbix (HTTP Agent) | n8n Needed? |
|---|---|---|
| n8n health (is it up?) | YES - GET /api/v1/workflows, check 200 | No |
| Failed execution count | YES - GET /api/v1/executions?status=error&limit=1, check totalItems | No |
| Failed execution details | Possible but complex JSONPath | n8n workflow better |
| Specific workflow failures | YES - add workflowId filter | No |

**Simplest health check:** `GET /api/v1/workflows` - returns 200 if n8n is up and API key is valid.

**Zabbix implementation:**
- HTTP Agent item: GET `{n8n_url}/api/v1/executions?status=error&limit=1` with X-N8N-API-KEY header
- Note: n8n runs inside Coolify on same Docker network, use internal hostname
- Dependent items: total error count from response
- Trigger: new errors detected (count increases)
- For detailed error info: n8n workflow that queries its own API and formats alerts

---

## Summary: Recommended Approach

### Pure Zabbix (HTTP Agent items) - no n8n needed

| Service | Health Check | Key Metrics |
|---|---|---|
| Bunny Stream | GET videolibrary/{id} -> 200 | VideoCount, StorageUsage, TrafficUsage |
| Mailjet | GET apikeytotals -> 200 | Delivered, Bounced, Spam counts + rates |
| Paddle | GET event-types -> 200 | HTTP code (200=valid, 401=expired) |
| n8n | GET workflows -> 200 | Execution error count |

### Needs n8n (complex logic, notifications, webhooks)

| Use Case | Why n8n |
|---|---|
| Paddle key expiry warning | Receive `api_key.expiring` webhook, no polling API exists |
| Detailed error digests | Aggregate n8n execution errors with node-level detail |
| Bounce analysis reports | Parse Mailjet bounce details into actionable format |
| Bunny encoding failure alerts | Get video details + error messages for failed encodes |

### Polling Intervals (recommended)

| Service | Interval | Rationale |
|---|---|---|
| Bunny library stats | 5m | Low volume, storage changes slowly |
| Mailjet statcounters | 5m | Email delivery is time-sensitive |
| Paddle key check | 1h | Key expiry is gradual |
| n8n failed executions | 1m | Workflow failures need fast detection |

### Zabbix Macro Suggestions

```
{$BUNNY_API_KEY}         - Account API key from bunny.net dashboard
{$BUNNY_LIBRARY_ID}      - Library ID (477336)
{$BUNNY_LIBRARY_KEY}     - Per-library API key
{$MAILJET_API_KEY}       - Public API key
{$MAILJET_SECRET_KEY}    - Private/secret API key
{$PADDLE_API_KEY}        - Bearer token (pdl_live_...)
{$PADDLE_KEY_CREATED}    - Key creation date (for expiry calc)
{$N8N_API_KEY}           - n8n API key
{$N8N_URL}               - n8n base URL (internal Docker network)
```
