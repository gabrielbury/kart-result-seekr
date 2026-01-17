Here is the raw Markdown file. You can copy this content directly, save it as `PROJECT_CONTEXT.md` (or similar), and upload it to your AI tool.

I have structured it specifically for LLM context ingestion: it defines **personas**, **strict technical constraints**, **data contracts**, and **architectural flows** to prevent the AI from hallucinating incorrect tools or workflows.

```markdown
# Project Context: Kart Result SEEKR
**Version:** 1.0.0
**Type:** Distributed System / On-Demand Scraping
**Architectural Pattern:** Event-Driven, Ephemeral Workers (Kubernetes Jobs)

## 1. Executive Summary
Kart Result SEEKR is an on-demand scraping solution designed to retrieve specific go-kart racing telemetry (Qualifying, Race, Lap-by-Lap) from a third-party racing track website. The system is triggered by a user via a web interface, orchestrates a temporary container to perform the scraping/PDF generation, stores the results in an S3-compatible bucket (Magalu Cloud), and delivers a download link back to the user via WebSocket.

## 2. Technology Stack Strategy

### 2.1 Frontend
- **Framework:** React (Vite)
- **Language:** TypeScript
- **State/Comms:** Socket.io-client (for real-time updates), Axios (for REST submission).
- **Styling:** TailwindCSS (recommended for speed).

### 2.2 Backend API (The Orchestrator)
- **Runtime:** Node.js
- **Language:** TypeScript
- **Framework:** Fastify or Express.
- **Key Libraries:** - `@kubernetes/client-node` (to dispatch jobs).
    - `socket.io` (to communicate with frontend).
    - `ioredis` (for managing job state and socket mapping).
- **Role:** Does NOT perform scraping. Acts purely as a dispatcher and state manager.

### 2.3 Crawler Worker (The Ephemeral Agent)
- **Runtime:** Node.js
- **Language:** TypeScript
- **Engine:** **Playwright** (Chosen over Selenium for native PDF rendering capabilities, speed, and reliability in Node environments).
- **Key Libraries:**
    - `playwright` (Chromium headless).
    - `archiver` (to zip files).
    - `@aws-sdk/client-s3` (for Magalu Cloud uploads).
- **Lifecycle:** Starts -> Reads Env Vars -> Scrapes -> Zips -> Uploads -> Calls Webhook -> Exits (0/1).

### 2.4 Infrastructure
- **Orchestrator:** Kubernetes (K8s).
- **Storage:** Magalu Cloud Object Storage (S3 Protocol).
- **State Database:** Redis (Ephemeral data: `protocol_id` <-> `socket_id`).

---

## 3. Detailed Architectural Flow

### Step 1: User Request
1. User fills form (Track, Date, Time) on Frontend.
2. Frontend connects to WebSocket (`ws://api/events`).
3. Frontend POSTs data to `POST /api/v1/scan`.

### Step 2: Orchestration (API)
1. API validates input.
2. API generates a UUID v4 (`protocol_id`).
3. API stores mapping in Redis: `SET protocol:{protocol_id} socket:{socket_id}`.
4. API constructs a **Kubernetes Job Manifest**.
    - **Namespace:** `seekr-workers`
    - **Image:** `seekr-crawler:latest`
    - **Env Vars:** `TARGET_DATE`, `TARGET_TIME`, `PROTOCOL_ID`, `WEBHOOK_SECRET`, `MAGALU_CREDS`.
5. API submits Job to K8s API Server.
6. API returns `202 Accepted` with `{ protocol_id }` to Frontend.

### Step 3: Execution (Crawler Container)
1. K8s schedules the Pod.
2. App starts.
3. **Login:** Authenticates with Racing Track portal (creds injected via K8s Secrets).
4. **Navigation:** Locates specific race by Date/Time filters.
5. **Extraction:**
   - Render Qualifying page -> `page.pdf()` -> Save to `/tmp/q.pdf`.
   - Render Race Result page -> `page.pdf()` -> Save to `/tmp/r.pdf`.
   - Render Laps page -> `page.pdf()` -> Save to `/tmp/l.pdf`.
6. **Compression:** Zips files to `/tmp/{protocol_id}.zip`.
7. **Storage:** Uploads zip to Magalu Bucket (Folder: `/results`).
8. **Completion:** - Generates Signed URL (or Public URL depending on bucket policy).
   - POSTs payload to API Webhook.

### Step 4: Notification
1. API receives Webhook (`POST /internal/webhook/callback`).
2. API validates `WEBHOOK_SECRET`.
3. API looks up `socket_id` using `protocol_id` from Redis.
4. API emits WS event `job_complete` with `{ download_url }`.
5. Frontend displays "Download Ready".
6. K8s Job completes (Pod terminates).

---

## 4. Technical Specifications & Contracts

### 4.1 Kubernetes Job Manifest Template (Conceptual)
The API must dynamically inject the `env` block.
```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: seekr-scan-${PROTOCOL_ID}
  namespace: seekr-workers
spec:
  ttlSecondsAfterFinished: 60 # Auto-cleanup
  template:
    spec:
      containers:
      - name: crawler
        image: [registry.gitlab.com/seekr/crawler:latest](https://registry.gitlab.com/seekr/crawler:latest)
        env:
        - name: JOB_PROTOCOL_ID
          value: "${PROTOCOL_ID}"
        - name: TARGET_DATE
          value: "${DATE}"
        resources:
          limits:
            memory: "1Gi"
            cpu: "1000m"
      restartPolicy: Never

```

### 4.2 API Endpoints

**POST /api/scan**

* **Request:**
```json
{
  "track_id": "interlagos-kart",
  "date": "2023-10-27",
  "time": "14:00"
}

```


* **Response:**
```json
{
  "protocol_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "processing"
}

```



**POST /internal/webhook (Crawler -> API)**

* **Headers:** `X-Internal-Secret: <hash>`
* **Request:**
```json
{
  "protocol_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "success", // or "error"
  "file_url": "https://magalu-bucket.s3.../file.zip",
  "error_message": null
}

```



### 4.3 WebSocket Events

* `connect`: Handshake.
* `status_update`: `{ step: "LOGIN" | "SCRAPING" | "UPLOADING" }`
* `result_ready`: `{ url: "..." }`
* `error`: `{ msg: "Race not found" }`

---

## 5. Security Constraints

1. **RBAC:** The API Pod ServiceAccount must only have permissions to `create`, `get`, `delete` Jobs in the `seekr-workers` namespace. It must NOT have cluster-admin.
2. **Isolation:** Worker pods run in a separate namespace with ResourceQuotas to prevent dDoS via resource exhaustion.
3. **Secrets:** Magalu Credentials and Target Website Credentials must be stored in K8s Secrets, never in code or Docker images.
4. **Input Sanitation:** The Date/Time inputs must be strictly regex validated in the API before being passed to the K8s Job to prevent Shell Injection attacks inside the container.

## 6. Development Tasks (Prompting Instructions)

When asking for code generation, please follow this order:

1. **Crawler:** Focus on Playwright logic and resilient selector strategies.
2. **API:** Focus on the K8s Client integration and WebSocket bridging.
3. **Infra:** Focus on the K8s YAML definitions and Dockerfile optimization for headless browsers.