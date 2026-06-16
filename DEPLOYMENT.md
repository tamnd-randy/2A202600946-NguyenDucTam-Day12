# Deployment Information

## Deployed Service Overview
- **Public URL:** `https://ai-agent-production.up.railway.app`
- **Platform:** Railway (Docker deployment via GitHub Integration)

---

## 🛠️ Environment Variables Set

The following environment variables have been securely configured on the cloud dashboard:
- `PORT`: `8000` (Injected dynamically by Railway)
- `ENVIRONMENT`: `production`
- `AGENT_API_KEY`: `dev-key-change-me` (Or a securely generated API Key)
- `JWT_SECRET`: `secure-jwt-secret-key-placeholder`
- `DAILY_BUDGET_USD`: `5.0`
- `RATE_LIMIT_PER_MINUTE`: `20`

---

## 🧪 Verification & Test Commands

### 1. Health Check (Liveness Probe)
Verify the agent service status and system uptime:
```bash
curl https://ai-agent-production.up.railway.app/health
```
**Expected Output:**
```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "production",
  "uptime_seconds": 15.4,
  "checks": {
    "llm": "mock"
  }
}
```

### 2. Readiness Check (Readiness Probe)
Ensure the agent is ready to process incoming calls:
```bash
curl https://ai-agent-production.up.railway.app/ready
```
**Expected Output:**
```json
{
  "ready": true
}
```

### 3. Ask Endpoint Test (Authentication Verification)

#### A. Unauthorized Request (Should return 401)
```bash
curl -X POST https://ai-agent-production.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
**Expected Response:**
```json
{
  "detail": "Invalid or missing API key. Include header: X-API-Key: <key>"
}
```

#### B. Authorized Request (Should return 200)
```bash
curl -X POST https://ai-agent-production.up.railway.app/ask \
  -H "X-API-Key: dev-key-change-me" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```
**Expected Response:**
```json
{
  "question": "What is Docker?",
  "answer": "Mock response to: What is Docker?",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-16T09:10:00Z"
}
```

---

## 📸 Screenshots
- **Railway Deployment Dashboard:** `screenshots/dashboard.png`
- **Service Running Logs:** `screenshots/running.png`
- **Successful cURL Test Results:** `screenshots/test.png`
