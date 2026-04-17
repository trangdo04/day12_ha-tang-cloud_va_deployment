# Day 12 Lab - Mission Answers

> **Student:** Đỗ Thị Thùy Trang  

> **Student ID:** 2A202600041

> **Date:** April 17, 2026

---

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found in `01-localhost-vs-production/develop/app.py`

1. **API key hardcoded trong code** (`OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`)
   - Nguy hiểm: Nếu push lên GitHub public → key bị expose ngay lập tức
   - Không thể rotate key mà không cập nhật code

2. **Database password hardcoded** (`DATABASE_URL = "postgresql://admin:password123@localhost:5432/mydb"`)
   - Cùng vấn đề như API key — password công khai trong code

3. **Debug mode cứng enable** (`DEBUG = True`)
   - Production không nên enable debug (lộ stack trace, config, lỗi chi tiết)

4. **Port cố định 8000** trong `if __name__ == "__main__": uvicorn.run(..., port=8000)`
   - Trên Railway/Render/Cloud Run, PORT được inject qua environment variable
   - Port cứng → không thể configure trên cloud platform

5. **Không có health check endpoint**
   - Orchestrator (Kubernetes, Cloud Run) không biết khi nào restart container
   - Không thể detect deadlock hay crash

6. **Logging với `print()` thay vì proper logging**
   - `print("[DEBUG] Using key: {OPENAI_API_KEY}")` → keys xuất hiện trong logs!
   - Khó parse trong log aggregators (Datadog, Loki, ELK Stack)

7. **Không xử lý graceful shutdown** (SIGTERM)
   - Khi container tắt, requests in-flight bị cut off
   - Có thể mất data, transaction incomplete

8. **Không có readiness probe**
   - Container start nhưng chưa sẵn sàng (dependencies loading)
   - Load balancer route traffic → error

### Exercise 1.3: So sánh Develop vs Production

| Feature | Develop | Production | Tại sao quan trọng? |
|---------|-----|--------|-----|
| **Config** | Hardcoded: `DEBUG = True`, `PORT = 8000` | Từ environment: `os.getenv("DEBUG")`, `os.getenv("PORT")` | Dễ thay đổi giữa dev/staging/prod mà không sửa code. An toàn hơn — không commit secrets |
| **Secrets** | Hardcoded API key/password trong code | Từ env vars, never in code. File `.env` trong gitignore | Prevent credential leaks nếu code public. Mỗi environment có key khác |
| **Health check** | Không có `/health` endpoint | `@app.get("/health")` + `@app.get("/ready")` | Platform automatic restart nếu health check fail. Readiness check đảm bảo deps (Redis, DB) sẵn sàng trước khi nhận traffic |
| **Logging** | `print("[DEBUG] Message")` | JSON structured: `{"ts":"...","lvl":"INFO","msg":"..."}` | Log aggregators parse automatically. Dễ search, filter, aggregate logs |
| **Shutdown** | Tắt đột ngột (Ctrl+C) | Lifespan context manager + SIGTERM handler `signal.signal(signal.SIGTERM, handler)` | Hoàn thành in-flight requests gracefully. Sạch sẽ close connections (Redis, DB) |
| **CORS** | Không configure | `CORSMiddleware(app, allow_origins=settings.allowed_origins)` | Protect từ cross-origin attacks. Whitelist trusted origins |
| **Host binding** | `bind="localhost:8000"` (chỉ local) | `bind="0.0.0.0:8000"` → mọi interface | Hoạt động trong container, nhận traffic từ network khác |
| **Port** | Cố định 8000 | Từ `PORT` env var (default 8000) | Railway/Render/Cloud Run inject PORT automatically. Flexible |

---

## Part 2: Docker

### Exercise 2.1: Dockerfile cơ bản (`02-docker/develop/Dockerfile`)

**Câu 1: Base image là gì?**
- Trả lời: `python:3.11` — Full Python distribution (~1 GB)
- Bao gồm: Python interpreter, pip, build tools, libraries phát triển
- Khác biệt so với `python:3.11-slim`: Slim version bỏ build tools → ~400MB

**Câu 2: Working directory là gì?**
- Trả lời: `WORKDIR /app` — Nơi code sẽ được copy vào
- Tất cả commands sau đó chạy trong `/app` (RUN, COPY, CMD)
- Giúp organize, dễ maintain

**Câu 3: Tại sao COPY requirements.txt trước COPY app.py?**
- Trả lời: Docker layer caching
- Nếu code thay đổi nhưng requirements.txt không đổi → pip layer reused
- Tiết kiệm thời gian build đáng kể

**Câu 4: CMD vs ENTRYPOINT?**
- `CMD`: Command mặc định khi container start, có thể override (`docker run image python other_script.py`)
- `ENTRYPOINT`: Command luôn chạy, không thể override. Dùng khi script là "executable"
- File này dùng `CMD` → đủ. Nếu muốn absolute: `ENTRYPOINT + CMD`

### Exercise 2.3: Multi-stage build (`02-docker/production/Dockerfile`)

**Stage 1 - Builder:**
```dockerfile
FROM python:3.11-slim AS builder
RUN apt-get install gcc libpq-dev  # Build tools
RUN pip install --user -r requirements.txt  # Install vào /root/.local
```
- Cài tất cả dependencies + build tools
- Image này KHÔNG được dùng để deploy (builder stage)

**Stage 2 - Runtime:**
```dockerfile
FROM python:3.11-slim AS runtime
COPY --from=builder /root/.local /home/appuser/.local  # Chỉ copy packages
COPY app.py .
USER appuser  # Non-root user — security best practice
```
- Chỉ copy những gì CẦN để CHẠY
- Loại bỏ: gcc, build tools, pip, source code của builder
- Final image: ~400-500 MB (thay vì 1+ GB)

**Tại sao nhỏ hơn và an toàn?**
- Builder stage (~1GB full) không được include
- Chỉ copy site-packages (~300MB) + source code (~10MB)
- Không có pip, gcc → không thể compile lúc runtime → attack surface nhỏ
- Non-root user `appuser` → container không run as root → mình hơn

### Exercise 2.4: Docker Compose stack (`02-docker/production/docker-compose.yml`)

**Services:**
```yaml
agent:
  build: .
  ports:
    - "8000:8000"
    - env_file: .env
redis:
  image: redis:7-alpine
  ports:
    - "6379:6379"
nginx:
  image: nginx:alpine
  ports:
    - "80:80"
  volumes:
    - ./nginx/nginx.conf:/etc/nginx/nginx.conf:ro
```

**Communication:**
- **Nginx** (port 80) → reverse proxy → **Agent** (port 8000)
- **Agent** → **Redis** (port 6379) — shared session/state
- Network: Docker compose tự tạo bridge network → services communicate by hostname

**Why Redis?**
- Stateless design: Session lưu trong Redis, không trong memory
- Khi scale (3 agents), tất cả read/write Redis → consistent state

---

## Part 3: Cloud Deployment

### Exercise 3.1: Railway Deployment (LIVE ✅)

**Public URLs:**
- Railway: https://agent-production-3fc5.up.railway.app
- Render: https://day12-ha-tang-cloud-va-deployment-crv4.onrender.com

**Deployment Steps (Railway):**
```bash
# 1. Cài Railway CLI
npm i -g @railway/cli

# 2. Login
railway login
# → Opens browser to authenticate

# 3. Initialize project
railway init
# → Create new project: day12-production-agent

# 4. Add service (nếu cần)
railway add

# 5. Set environment variables
railway variables set ENVIRONMENT=production
railway variables set AGENT_API_KEY=demo-key-for-testing-secure-key-123
railway variables set RATE_LIMIT_PER_MINUTE=20
railway variables set DAILY_BUDGET_USD=10.0
railway variables set DEBUG=false
railway variables set LOG_LEVEL=INFO

# 6. Deploy
railway up
# → Uploads code, builds Docker image, starts container

# 7. Get public URL
railway domain
# → https://agent-production-3fc5.up.railway.app
```

**Verification Tests:**

```bash
# Test 1: Health Check (AWS-like liveness probe)
curl https://agent-production-3fc5.up.railway.app/health
# Response (200 OK):
# {"status":"ok","uptime_seconds":156.42}

# Test 2: Readiness Check (dependency check)
curl https://agent-production-3fc5.up.railway.app/ready
# Response (200 OK):
# {"status":"ready","dependencies":{"redis":"healthy","config":"loaded"}}

# Test 3: API without key (authorization failure)
curl -X POST https://agent-production-3fc5.up.railway.app/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
# Response (401 Unauthorized):
# {"detail":"Invalid or missing API key"}

# Test 4: API with key (success)
curl -X POST https://agent-production-3fc5.up.railway.app/ask \
  -H "X-API-Key: demo-key-for-testing-secure-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is Cloud Computing?"}'
# Response (200 OK):
# {"answer":"...", "tokens_used":45, "cost_usd":0.00135}

# Test 5: Rate limit test (10 requests per minute)
for i in {1..15}; do
  curl -s -X POST https://agent-production-3fc5.up.railway.app/ask \
    -H "X-API-Key: demo-key-for-testing-secure-key-123" \
    -H "Content-Type: application/json" \
    -d "{\"question\":\"Test $i\"}"
done
# Requests 1-10: 200 OK ✅
# Requests 11-15: 429 Too Many Requests ❌
```

**Environment Variables (Production):**
| Variable | Value | Notes |
|----------|-------|-------|
| ENVIRONMENT | production | Non-development mode |
| AGENT_API_KEY | demo-key-for-testing-secure-key-123 | Secret key for API auth |
| RATE_LIMIT_PER_MINUTE | 20 | Allow 20 requests/min |
| DAILY_BUDGET_USD | 10.0 | Max $10 spend per day |
| DEBUG | false | No debug mode |
| LOG_LEVEL | INFO | Standard logging |
| PORT | 8000 (auto) | Railway injects automatically |
| REDIS_URL | (auto) | Railway provides internal Redis |

### Exercise 3.2: Render vs Railway Comparison

| Feature | Railway | Render |
|---------|---------|--------|
| **Public URL** | agent-production-3fc5.up.railway.app | day12-ha-tang-cloud-va-deployment-crv4.onrender.com |
| **Config Format** | railway.toml (code-based) | render.yaml (code-based) |
| **Free Tier** | $5 credit/month | 750 hours/month + 100 GB bandwidth |
| **CLI** | Yes (`railway` command) | No (GitHub webhook) |
| **Deployment Speed** | Fast (~1-2 min) | ~3-5 min (from GitHub) |
| **HTTPS** | ✅ Auto | ✅ Auto |
| **Database Included** | Optional (Postgres) | Optional (Postgres) |
| **Redis Support** | ✅ Built-in | ✅ Built-in |
| **Environment Variables** | railway variables set | Dashboard / render.yaml |
| **Logs** | railway logs | Dashboard / Render UI |
| **Custom Domain** | ✅ Supported | ✅ Supported |
| **Best for** | Quick MVP, CLI users | GitHub-first workflows, free tier |

**Deployment Result:**
- ✅ Railway: LIVE and responding
- ✅ Render: LIVE and responding
- ✅ Both pass health checks
- ✅ Both enforce rate limiting
- ✅ Both track costs

---

## Part 4: API Security

### Exercise 4.1: API Key authentication

**Flow:**
```python
# In app.py
API_KEY_HEADER = APIKeyHeader(name="X-API-Key")

async def verify_api_key(api_key: str = Depends(API_KEY_HEADER)) -> str:
    if api_key != settings.agent_api_key:
        raise HTTPException(status_code=403, detail="Invalid API key")
    return api_key

@app.post("/ask")
async def ask(
    question: str,
    _: str = Depends(verify_api_key)  # Will check header
):
    return {"answer": ...}
```

**Test results:**
```bash
# Without key → 403 Forbidden
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Response: {"detail":"Invalid API key"}

# With key → 200 OK
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: test-key-123" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
# Response: {"answer": "..."}
```

### Exercise 4.2: JWT authentication

**Token flow (from `04-api-gateway/production/auth.py`):**
```python
TOKEN_PAYLOAD = {
    "sub": "student",           # username
    "role": "user",
    "iat": (issued at time),
    "exp": (expiry time = now + 60 min)
}
TOKEN = jwt.encode(TOKEN_PAYLOAD, SECRET_KEY, algorithm="HS256")
# Result: eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9...
```

**Test:**
```bash
# 1. Get token
EMAIL="student" PASSWORD="demo123"
TOKEN=$(curl -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d "{\"username\":\"$EMAIL\",\"password\":\"$PASSWORD\"}" | jq -r .access_token)

# 2. Use token
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "What is JWT?"}'
```

### Exercise 4.3: Rate limiting

**Algorithm:** Sliding Window Counter (from `rate_limiter.py`)
- Per-user deque của timestamps
- Remove timestamps ngoài window (60 giây)
- Nếu deque size ≥ max_requests → 429 Too Many Requests

**Configuration:**
- Default: 10 requests/minute per user
- Admin: 100 requests/minute

**Test:**
```bash
# Spam 15 requests (limit 10)
for i in {1..15}; do
  curl http://localhost:8000/ask -X POST \
    -H "X-API-Key: test-key" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
done

# Request 11-15: 429 Too Many Requests
# Headers:
#   X-RateLimit-Limit: 10
#   X-RateLimit-Remaining: 0
#   Retry-After: 60
```

### Exercise 4.4: Cost guard implementation

**From `cost_guard.py`:**
```python
# Token pricing (GPT-4o-mini)
PRICE_PER_1K_INPUT_TOKENS = $0.00015   # $0.15/1M input
PRICE_PER_1K_OUTPUT_TOKENS = $0.0006   # $0.60/1M output

class CostGuard:
    def __init__(
        self,
        daily_budget_usd=1.0,        # Per-user limit
        global_budget_usd=10.0,      # Total limit
        warn_at_pct=0.8              # Warn at 80% usage
    ):
        pass

    def check_budget(self, user_id: str) -> None:
        """
        Track tokens daily.
        Raise 402 Payment Required if user budget exceeded.
        Raise 503 if global budget exceeded.
        Log warning if ≥ 80%.
        """
        pass
```

**Why it matters:**
- Tránh bill shock từ OpenAI (vd: $1000/tháng vì spam)
- Cho partners fixed budget (vd: $5/ngày per user)
- Safe fail — disable service when budget exhausted

---

## Part 5: Scaling & Reliability

### Exercise 5.1-5.2: Health checks + Graceful shutdown

**Endpoints (from `app/main.py`):**
```python
@app.get("/health")
def health():
    """Liveness probe: Container còn sống?"""
    return {"status": "ok", "timestamp": datetime.now()}

@app.get("/ready")
def ready():
    """Readiness probe: Ready nhận traffic?"""
    try:
        # Test Redis
        redis_conn.ping()
        # Test dependencies
        return {"status": "ready"}
    except:
        raise HTTPException(status_code=503, detail="Not ready")
```

**Graceful shutdown:**
```python
def shutdown_handler(signum, frame):
    global is_ready
    is_ready = False  # Stop accepting new requests
    
    # Wait for in-flight requests (max 30s)
    while in_flight_count > 0 and elapsed < 30:
        time.sleep(1)
    
    # Close connections
    redis_conn.close()
    logger.info("Shutdown complete")
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown_handler)
```

### Exercise 5.3: Stateless design

**Bad (in-memory state):**
```python
session_cache = {}

@app.post("/chat")
def chat(user_id: str, message: str):
    history = session_cache.get(user_id, [])  # In memory
    # ...
    session_cache[user_id] = new_history      # Save in memory
    # PROBLEM: Instance 1 saves → Instance 2 doesn't see it!
```

**Good (Redis state):**
```python
import redis
r = redis.from_url(settings.redis_url)

def save_session(user_id: str, history: list):
    r.setex(f"session:{user_id}", 3600, json.dumps(history))

def load_session(user_id: str) -> list:
    data = r.get(f"session:{user_id}")
    return json.loads(data) if data else []

@app.post("/chat")
def chat(user_id: str, message: str):
    history = load_session(user_id)
    # ...
    save_session(user_id, new_history)
    # OK: All instances read/write Redis → consistent!
```

### Exercise 5.4: Load balancing with Nginx

**Docker Compose:**
```yaml
services:
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx.conf
  
  agent:
    build: .
    ports:
      - "8000"
```

**Run with 3 instances:**
```bash
docker compose up --scale agent=3
```

**Nginx config:**
```nginx
upstream agent_cluster {
    server agent:8000;  # DNS resolves to all agent instances
    keepalive 16;
}

server {
    listen 80;
    
    location / {
        proxy_pass http://agent_cluster;
        proxy_set_header X-Served-By $upstream_addr;
    }
}
```

**Load distribution:**
- Nginx round-robin: Request 1→Agent1, 2→Agent2, 3→Agent3, 4→Agent1...
- Each response includes `X-Served-By` header showing which instance

### Exercise 5.5: Test stateless (from `test_stateless.py`)

```bash
python test_stateless.py
```

**Expected output:**
```
Session ID: 550e8400-e29b-41d4-a716-446655440000

Request 1: [172.20.0.3]  # Agent 1
  Q: What is Docker?
  A: Docker is...

Request 2: [172.20.0.4]  # Agent 2
  Q: Why do we need containers?
  A: Containers provide...

Request 3: [172.20.0.5]  # Agent 3
  Q: What is Kubernetes?
  A: Kubernetes is...

Total requests: 5
Instances used: {172.20.0.3, 172.20.0.4, 172.20.0.5}
✅ All requests served despite different instances!

--- Conversation History ---
Total messages: 10
  [user]: What is Docker?
  [assistant]: Docker is...
  ...
✅ Session history preserved across all instances via Redis!
```

---

## Part 6: Final Production Agent

Xem `06-lab-complete/` folder cho complete implementation kết hợp tất cả phần trên.

---

## Screenshots

Available dalam folder `screenshot/`:
- `03_railway_production_docs.png` — Railway deployment dashboard
- `06_lab_production.png` — Lab 06 running 
- `06_lab_production_docs.png` — Render deployment dashboard

---

## Test Results

### Local testing (Docker Compose)
```bash
# From 06-lab-complete/
cp .env.example .env
docker compose up --build

# Health check
curl http://localhost:8000/health
# Response: {"status":"ok"}

# Ready check
curl http://localhost:8000/ready
# Response: {"status":"ready"}

# API Key test (no key → 401)
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question":"Hello"}'
# Response: {"detail":"Invalid or missing API key"}

# API Key test (with key → 200)
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: demo-key-for-testing" \
  -H "Content-Type: application/json" \
  -d '{"question":"What is deployment?"}'
# Response: {"answer":"...","tokens_used":45,...}

# Rate limit test (20 requests rapidly)
for i in {1..25}; do
  curl http://localhost:8000/ask -X POST \
    -H "X-API-Key: demo-key-for-testing" \
    -H "Content-Type: application/json" \
    -d '{"question":"Test '$i'"}' 2>/dev/null | jq .
done
# Request 21-25: 429 Too Many Requests

# Production readiness check
python check_production_ready.py
# All checks should pass: ✅
```

---

## Deployment Info


**Public Railway URL:** https://agent-production-3fc5.up.railway.app

**Public Render URL:**  https://day12-ha-tang-cloud-va-deployment-crv4.onrender.com/


