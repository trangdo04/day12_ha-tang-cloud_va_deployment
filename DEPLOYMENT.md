# Deployment Information

## Public URL

**Railway**:  https://agent-production-3fc5.up.railway.app

**Render**: https://day12-ha-tang-cloud-va-deployment-crv4.onrender.com/

## Platform Information

**Platform:** Railway  / Render


## Test Commands

### Health Check
```bash
curl https://agent-production-3fc5.up.railway.app/health
# Expected: {"status": "ok"}
```

```bash
curl https://day12-ha-tang-cloud-va-deployment-crv4.onrender.com/health
# Expected: {"status": "ok"}
```

## Environment Variables Set
- **PORT**: 8000 (Railway/Render inject automatically)
- **ENVIRONMENT**: production
- **REDIS_URL**: redis://redis:6379 (internal)
- **AGENT_API_KEY**: demo-key-for-testing-secure-key-123
- **LOG_LEVEL**: INFO
- **DEBUG**: false
- **RATE_LIMIT_PER_MINUTE**: 20
- **DAILY_BUDGET_USD**: 10.0

## screenshot

- `screenshot/03_railway_production_docs.png` — Railway deployment dashboard
- `screenshot/06_lab_production.png` — Lab 06 running 
- `screenshot/06_lab_production_docs.png` — Render deployment dashboard

---

## Deployment Steps

### Deploy Railway

```bash
# 1. Cài Railway CLI
npm i -g @railway/cli

# 2. Login
railway login

# 3. Initialize (tạo project mới)
railway init

# 4. Add service (nếu cần)
railway add

# 5. Set environment variables
railway variables set OPENAI_API_KEY=sk-... 
railway variables set AGENT_API_KEY=your-secret-key

# 6. Deploy!
railway up

# 7. Nhận public URL
railway domain
# Output: https://agent-production-3fc5.up.railway.app/
```

### Deploy Render (Alternative)

1. Push repo lên GitHub
2. Render Dashboard → New → Blueprint
3. Connect repo → Render đọc `render.yaml`
4. Set secrets in Render dashboard:
   - `ENVIRONMENT` = production
   - `AGENT_API_KEY` = demo-key-for-testing-secure-key-123
   - `RATE_LIMIT_PER_MINUTE` = 20
   - `DAILY_BUDGET_USD` = 10.0
   - `DEBUG` = false
   - `LOG_LEVEL` = INFO
   - `REDIS_URL` = Auto-provided by Render
5. Deploy → Nhận URL!

---

## Chạy Local trước khi Deploy

```bash
# 1. Setup
cp .env.example .env

# 2. Chạy với Docker Compose
docker compose up --build

# 3. Test health check
curl http://localhost:8000/health

# 4. Lấy API key từ .env
# Windows PowerShell:
$API_KEY = (Get-Content .env | Select-String '^AGENT_API_KEY=').ToString().Split('=')[1]
echo $API_KEY

# Mac/Linux:
export API_KEY=$(grep '^AGENT_API_KEY=' .env | cut -d'=' -f2)
echo $API_KEY

# 5. Test endpoint
# Windows PowerShell:
$headers = @{
  "X-API-Key" = $API_KEY
  "Content-Type" = "application/json"
}
$body = '{"question":"What is deployment?"}'

Invoke-RestMethod `
  -Uri "http://localhost:8000/ask" `
  -Method POST `
  -Headers $headers `
  -Body $body

# Mac/Linux/WSL:
curl -X POST http://localhost:8000/ask \
  -H "X-API-Key: $API_KEY" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is deployment?"}'
```

## Kiểm Tra Production Readiness

```bash
# Chạy script kiểm tra trước khi deploy
python check_production_ready.py

# Output: Result: 20/20 checks passed (100%)
```

