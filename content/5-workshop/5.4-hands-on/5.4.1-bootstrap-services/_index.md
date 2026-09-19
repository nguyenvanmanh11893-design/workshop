---
title: "Bootstrap and Verify Services"
date: 2026-09-18T23:00:00+07:00
weight: 1
chapter: false
pre: "<b>5.4.1 </b>"
---



---

### 1. Objective
* Concurrently start the Backend API and React Frontend development server using a single command.
* Run the standalone background job worker process (Durable Worker).
* Validate database connectivity and system health using the `/health/live` and `/health/ready` endpoints.
* Access the web user interface and the interactive Swagger API documentation.

---

### 2. Prerequisites
* Completed all steps in **[Module 5.3: Prerequisites and Environment Setup](../../5.3-prerequisites/)**.
* MySQL Server running on port `3306` with all migrations applied (`npm run db:migrate`).
* A valid `.env` file configured in the project root.

---

### 3. Execution Steps

#### Step 1: Start Web Application (Frontend & Backend API)
Open Terminal 1 in the project root directory and run:

```bash
npm run dev
```

*This command leverages npm workspaces to concurrently launch:*
* Express API backend listening on `http://localhost:3000`.
* Vite dev server listening on `http://localhost:5173` (with `/api` pre-configured to proxy to port `3000`).

#### Step 2: Start Durable Background Worker
Open Terminal 2 in the project root directory and run:

```bash
npm run worker
```

*The background process `worker.js` initializes periodic polling (default 3000ms), ready to claim and process tasks from the `Jobs` table.*

---

### 4. Expected Outcome

* **Terminal 1 (App Dev)**:
```text
[backend]  Server running in development mode on port 3000
[backend]  Database connected successfully.
[frontend] VITE v5.x.x  ready in 320 ms
[frontend] ➜  Local:   http://localhost:5173/
[frontend] ➜  Network: use --host to expose
```

* **Terminal 2 (Worker)**:
```text
[worker] Worker started. Polling interval: 3000ms
[worker] Heartbeat registered in database.
[worker] Polling for jobs... (0 active leases)
```

---

### 5. Verification Steps

#### Check 1: API Liveness Probe
Open a new terminal window and send an HTTP GET request to test liveness:

```bash
curl -i http://localhost:3000/health/live
```
*Expected Response:* HTTP `200 OK`
```json
{
  "status": "ok",
  "timestamp": "2026-09-18T23:00:00.000Z"
}
```

#### Check 2: API Readiness Probe & Database Health
Verify database connectivity through the readiness probe:

```bash
curl -i http://localhost:3000/health/ready
```
*Expected Response:* HTTP `200 OK`
```json
{
  "status": "ready",
  "database": "connected",
  "uptime_seconds": 45
}
```


#### Check 3: Browser UI and Documentation
Open your browser and navigate to:
* **Web Client**: `http://localhost:5173` — Cloud File Manager login interface rendered with Ant Design components.
* **Swagger API Docs**: `http://localhost:3000/api-docs` — Interactive OpenAPI Swagger UI displaying all available REST endpoints with request/response schemas.
