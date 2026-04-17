#  Code Lab: Deploy Your AI Agent to Production

> **AICB-P1 · VinUniversity 2026**  
> Thời gian: 3-4 giờ | Độ khó: Intermediate

##  Mục Tiêu

Sau khi hoàn thành lab này, bạn sẽ:
- Hiểu sự khác biệt giữa development và production
- Containerize một AI agent với Docker
- Deploy agent lên cloud platform
- Bảo mật API với authentication và rate limiting
- Thiết kế hệ thống có khả năng scale và reliable

---

##  Yêu Cầu

```bash
 Python 3.11+
 Docker & Docker Compose
 Git
 Text editor (VS Code khuyến nghị)
 Terminal/Command line
```

**Không cần:**
-  OpenAI API key (dùng mock LLM)
-  Credit card
-  Kinh nghiệm DevOps trước đó

---

##  Lộ Trình Lab

| Phần | Thời gian | Nội dung |
|------|-----------|----------|
| **Part 1** | 30 phút | Localhost vs Production |
| **Part 2** | 45 phút | Docker Containerization |
| **Part 3** | 45 phút | Cloud Deployment |
| **Part 4** | 40 phút | API Security |
| **Part 5** | 40 phút | Scaling & Reliability |
| **Part 6** | 60 phút | Final Project |

---

## Part 1: Localhost vs Production (30 phút)

###  Concepts

**Vấn đề:** "It works on my machine" — code chạy tốt trên laptop nhưng fail khi deploy.

**Nguyên nhân:**
- Hardcoded secrets
- Khác biệt về environment (Python version, OS, dependencies)
- Không có health checks
- Config không linh hoạt

**Giải pháp:** 12-Factor App principles

###  Exercise 1.1: Phát hiện anti-patterns

```bash
cd 01-localhost-vs-production/develop
```

**Nhiệm vụ:** Đọc `app.py` và tìm ít nhất 5 vấn đề.

<details>
<summary> Gợi ý</summary>

Tìm:
- API key hardcode
- Port cố định
- Debug mode
- Không có health check
- Không xử lý shutdown

</details>

###  Exercise 1.2: Chạy basic version

```bash
pip install -r requirements.txt
python app.py
```

Test:
```bash
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

**Quan sát:** Nó chạy! Nhưng có production-ready không?

###  Exercise 1.3: So sánh với advanced version

```bash
cd ../production
cp .env.example .env
pip install -r requirements.txt
python app.py
```

**Nhiệm vụ:** So sánh 2 files `app.py`. Điền vào bảng:

| Feature | Basic | Advanced | Tại sao quan trọng? |
|---------|-------|----------|---------------------|
| Config | Hardcode trong code | Từ environment variables (PORT, DEBUG, OPENAI_API_KEY) | Tránh commit secrets, dễ đổi giữa dev/prod, an toàn với Git |
| Health check | Không có | `@app.get("/health")` + `@app.get("/ready")` | Platform (Railway, Kubernetes) biết khi nào restart, readiness đảm bảo connection ready |
| Logging | print() — khó parse | JSON structured logging | Log aggregators (Datadog, Loki, ELK) có thể parse, dễ search errors |
| Shutdown | Đột ngột (Ctrl+C) | Lifespan context manager + SIGTERM handler | In-flight requests hoàn thành, sạch sẽ không lose data |
| CORS | N/A | Configured via ALLOWED_ORIGINS | Bảo vệ từ cross-origin attacks, whitelist origins |
| Host binding | localhost (chỉ local) | 0.0.0.0 (mọi interface) | Hoạt động trong container, nhận traffic từ bên ngoài |
| Secrets | Hardcoded sk-key, password123 | Từ env var, không visible trong code | Nếu push lên GitHub → key bị lộ ngay lập tức |
| Port | Cứng port 8000 | Từ PORT env var | Railway/Render inject PORT tự động, flexible |

###  Checkpoint 1

- [ ] Hiểu tại sao hardcode secrets là nguy hiểm
- [ ] Biết cách dùng environment variables
- [ ] Hiểu vai trò của health check endpoint
- [ ] Biết graceful shutdown là gì

---

## Part 2: Docker Containerization (45 phút)

###  Concepts

**Vấn đề:** "Works on my machine" part 2 — Python version khác, dependencies conflict.

**Giải pháp:** Docker — đóng gói app + dependencies vào container.

**Benefits:**
- Consistent environment
- Dễ deploy
- Isolation
- Reproducible builds

###  Exercise 2.1: Dockerfile cơ bản

```bash
cd ../../02-docker/develop
```

**Nhiệm vụ:** Đọc `Dockerfile` và trả lời:

1. Base image là gì?  
   **Trả lời:** `python:3.11` — Full Python distribution (~1GB), bao gồm cả pip, libraries phát triển

2. Working directory là gì?  
   **Trả lời:** `/app` — Nơi code sẽ được copy vào, tất cả commands sau đó chạy trong thư mục này

3. Tại sao COPY requirements.txt trước?  
   **Trả lời:** Docker layer cache — Nếu code thay đổi nhưng requirements.txt không đổi, layer pip cache được reuse thay vì cài lại tất cả (tiết kiệm thời gian build)

4. CMD vs ENTRYPOINT khác nhau thế nào?  
   **Trả lời:**
   - `CMD`: Lệnh mặc định khi container start, có thể được override (`docker run app.py` sẽ override CMD)
   - `ENTRYPOINT`: Lệnh luôn chạy, CMD là arguments cho ENTRYPOINT. Trong file này dùng `CMD ["python", "app.py"]` là đủ

###  Exercise 2.2: Build và run

```bash
# Build image
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .

# Run container
docker run -p 8000:8000 my-agent:develop

# Test
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "What is Docker?"}'
```

**Quan sát:** Image size là bao nhiêu?
```bash
docker images my-agent:develop
```

###  Exercise 2.3: Multi-stage build

```bash
cd ../production
```

**Nhiệm vụ:** Đọc `Dockerfile` và tìm:
- Stage 1 làm gì?  
  **Trả lời:** Builder stage — cài build tools (gcc, libpq-dev), copy requirements.txt, chạy `pip install --user` để cài tất cả dependencies vào `/root/.local`

- Stage 2 làm gì?  
  **Trả lời:** Runtime stage — chỉ copy `/root/.local` từ builder sang, copy source code, tạo non-root user, loại bỏ tất cả build tools → image nhỏ và an toàn

- Tại sao image nhỏ hơn?  
  **Trả lời:**
  - Builder stage có gcc, libpq-dev → ~300-500MB thêm
  - Runtime chỉ lấy `.local` (~200-300MB packages) → image cuối ~400-500MB thay vì 1GB+
  - Không còn build tools → image hoàn toàn stateless, không cách nào compile trong container (vừa an toàn vừa nhỏ)

Build và so sánh:
```bash
docker build -t my-agent:advanced .
docker images | grep my-agent
```

###  Exercise 2.4: Docker Compose stack

**Nhiệm vụ:** Đọc `docker-compose.yml` và vẽ architecture diagram.

```bash
cd ../production  # or wherever docker-compose.yml is
docker compose up
```

Services nào được start? Chúng communicate thế nào?  

**Mô tả Stack:**
```
┌──────────────────────────────────────────────┐
│          Docker Compose Services              │
├──────────────────────────────────────────────┤
│  nginx (port 80)                             │
│    ↓                                          │
│  agent service (port 8000, 1 instance)      │
│    ↓                                          │
│  Shared network: my-network                  │
└──────────────────────────────────────────────┘
```

- **Services:** nginx (reverse proxy) + agent
- **Communication:** Via Docker network bridge — nginx forward requests đến agent:8000
- **Volumes:** Optional — mount source code để development
- **Environment:** Port, API key, environment variables

Test:
```bash
# Health check (qua Nginx)
curl http://localhost/health

# Agent endpoint  
curl http://localhost/ask -X POST \
  -H "X-API-Key: my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain microservices"}'

# Hoặc direct đến agent
curl http://localhost:8000/health
```

**Debug:**
```bash
# View logs
docker compose logs -f agent

# Access shell trong container
docker compose exec agent /bin/bash

# Check network
docker network ls
docker network inspect <network_name>
```

###  Checkpoint 2

- [ ] Hiểu cấu trúc Dockerfile
- [ ] Biết lợi ích của multi-stage builds
- [ ] Hiểu Docker Compose orchestration
- [ ] Biết cách debug container (`docker logs`, `docker exec`)

---

## Part 3: Cloud Deployment (45 phút)

###  Concepts

**Vấn đề:** Laptop không thể chạy 24/7, không có public IP.

**Giải pháp:** Cloud platforms — Railway, Render, GCP Cloud Run.

**So sánh:**

| Platform | Độ khó | Free tier | Best for |
|----------|--------|-----------|----------|
| Railway | ⭐ | $5 credit | Prototypes |
| Render | ⭐⭐ | 750h/month | Side projects |
| Cloud Run | ⭐⭐⭐ | 2M requests | Production |

###  Exercise 3.1: Deploy Railway (15 phút)

```bash
cd ../../03-cloud-deployment/railway
```

**Steps:**

1. Install Railway CLI:
```bash
npm i -g @railway/cli
```

2. Login:
```bash
railway login
```

3. Initialize project:
```bash
railway init
```

4. Set environment variables:
```bash
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
```

5. Deploy:
```bash
railway up
```

6. Get public URL:
```bash
railway domain
```

**Nhiệm vụ:** Test public URL với curl hoặc Postman.

Test:
```bash
# Health check
curl http://student-agent-domain/health

# Agent endpoint
curl http://studen-agent-domain/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": ""}'
```

###  Exercise 3.2: Deploy Render (15 phút)

```bash
cd ../render
```

**Steps:**

1. Push code lên GitHub (nếu chưa có)
2. Vào [render.com](https://render.com) → Sign up
3. New → Blueprint
4. Connect GitHub repo
5. Render tự động đọc `render.yaml`
6. Set environment variables trong dashboard
7. Deploy!

**Nhiệm vụ:** So sánh `render.yaml` với `railway.toml`. Khác nhau gì?  

**Trả lời:**
- **render.yaml**: YAML format, IaC style — define services, environment, ports
- **railway.toml**: TOML format, simpler — focus on build + start commands
- **Similarities**: Cả hai define PORT, environment variables, phải config secrets
- **Render advantage**: Better free tier (750h/month), more config options
- **Railway advantage**: Simpler, faster to deploy, có CLI tool

**Test Public URL:**
```bash
curl https://your-render-domain/health
curl -H "X-API-Key: your-secret-key" https://your-render-domain/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello from Render"}'
```

###  Exercise 3.3: (Optional) GCP Cloud Run (15 phút)

```bash
cd ../production-cloud-run
```

**Yêu cầu:** GCP account (có free tier).

**Nhiệm vụ:** Đọc `cloudbuild.yaml` và `service.yaml`. Hiểu CI/CD pipeline.

###  Checkpoint 3

- [ ] Deploy thành công lên ít nhất 1 platform
- [ ] Có public URL hoạt động
- [ ] Hiểu cách set environment variables trên cloud
- [ ] Biết cách xem logs

---

## Part 4: API Security (40 phút)

###  Concepts

**Vấn đề:** Public URL = ai cũng gọi được = hết tiền OpenAI.

**Giải pháp:**
1. **Authentication** — Chỉ user hợp lệ mới gọi được
2. **Rate Limiting** — Giới hạn số request/phút
3. **Cost Guard** — Dừng khi vượt budget

###  Exercise 4.1: API Key authentication

```bash
cd ../../04-api-gateway/develop
```

**Nhiệm vụ:** Đọc `app.py` và tìm:
- API key được check ở đâu?  
  **Trả lời:** 
  - Config: `API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")`
  - Dependency function: `verify_api_key()` — kiểm tra header `X-API-Key`
  - Endpoint: `@app.post("/ask")` sử dụng `_key: str = Depends(verify_api_key)` để require auth

- Điều gì xảy ra nếu sai key?  
  **Trả lời:** 
  - Không gửi key → 401 Unauthorized: "Missing API key"
  - Sai key → 403 Forbidden: "Invalid API key"

- Làm sao rotate key?  
  **Trả lời:** Đổi giá trị `AGENT_API_KEY` environment variable, deploy lại. Requests cũ sẽ bị 403 cho đến khi client update key

Test:
```bash
python app.py

#  Không có key → 401
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'

#  Có key → 200
curl http://localhost:8000/ask -X POST \
  -H "X-API-Key: my-secret-key" \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello"}'
```

###  Exercise 4.2: JWT authentication (Advanced)

```bash
cd ../production
```

**Nhiệm vụ:** 
1. Đọc `auth.py` — hiểu JWT flow  
   **Mô tả:**
   - JWT = JSON Web Token, format: `header.payload.signature`
   - Payload chứa: username, role, expiry time (iat, exp)
   - Server sign bằng SECRET_KEY → client không thể fake token
   - Flow: Login lấy token → gửi token mỗi request → server decode + verify signature

2. Lấy token:
```bash
# Terminal 1: Start server
python app.py

# Terminal 2: Login
curl -X POST http://localhost:8000/auth/token \
  -H "Content-Type: application/json" \
  -d '{"username": "student", "password": "demo123"}'

# Response: {"access_token": "eyJ0eXAiOiJKV1QiL...", "token_type": "bearer"}
```

3. Dùng token để gọi API:
```bash
TOKEN="<token_từ_bước_2>"
curl http://localhost:8000/ask -X POST \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"question": "Explain JWT"}'

# Nếu token expired → 401 Unauthorized: "Token expired"
# Nếu token invalid → 403 Forbidden: "Invalid token"
```

**So sánh API Key vs JWT:**
- API Key: Đơn giản, không expiry, dễ bị lộ nếu hardcode
- JWT: Stateless, expiry built-in, có role/claims, phải keep SECRET_KEY an toàn

###  Exercise 4.3: Rate limiting

**Nhiệm vụ:** Đọc `rate_limiter.py` và trả lời:
- Algorithm nào được dùng? (Token bucket? Sliding window?)  
  **Trả lời:** Sliding Window Counter
  - Mỗi user có 1 deque timestamps
  - Mỗi request thêm timestamp vào deque
  - Loại bỏ timestamps cũ (ngoài 60 giây)
  - Nếu deque size ≥ max_requests → 429 Too Many Requests

- Limit là bao nhiêu requests/minute?  
  **Trả lời:**
  - User: 10 requests/minute — `rate_limiter_user = RateLimiter(max_requests=10, window_seconds=60)`
  - Admin: 100 requests/minute — `rate_limiter_admin = RateLimiter(max_requests=100, window_seconds=60)`

- Làm sao bypass limit cho admin?  
  **Trả lời:** Check `role` trong JWT token, dùng `rate_limiter_admin` thay vì `rate_limiter_user` trong @Depends

Test:
```bash
# Gọi liên tục 15 lần (limit 10)
for i in {1..15}; do
  curl http://localhost:8000/ask -X POST \
    -H "Authorization: Bearer $TOKEN" \
    -H "Content-Type: application/json" \
    -d '{"question": "Test '$i'"}'
  echo ""
done

# Request 11-15 sẽ nhận 429 với header:
# X-RateLimit-Limit: 10
# X-RateLimit-Remaining: 0
# Retry-After: 60
```

###  Exercise 4.4: Cost guard

**Nhiệm vụ:** Đọc `cost_guard.py` và implement logic:

```python
def check_budget(user_id: str, estimated_cost: float) -> bool:
    """
    Return True nếu còn budget, False nếu vượt.
    
    Logic:
    - Mỗi user có budget $1/ngày (configurable)
    - Global budget: $10/ngày (stop khi vượt)
    - Track spending trong instance (in-memory)
    - Reset đầu ngày mới (check timestamp)
    - Warn khi dùng 80% budget
    """
    pass
```

**Giải thích code hiện tại:**
- `UsageRecord`: Dataclass track input_tokens, output_tokens, cost mỗi ngày
- `CostGuard` class:
  - `daily_budget_usd=$1.0` — per user limit
  - `global_daily_budget_usd=$10.0` — tổng cộng limit
  - `warn_at_pct=0.8` — cảnh báo khi dùng 80%
- `check_budget()`: 
  - Nếu vượt global → 503 Service Unavailable
  - Nếu vượt per-user → 402 Payment Required
  - Còn budget → log warning nếu ≥ 80%

**Tại sao quan trọng?**
- Tránh bill bất ngờ từ OpenAI (vd $1000/tháng)
- Cho partner/customer access nhưng budget controlled
- Disable service khi budget hết (safe fail)

###  Checkpoint 4

- [ ] Implement API key authentication
- [ ] Hiểu JWT flow + token expiry
- [ ] Implement rate limiting (Sliding Window)
- [ ] Implement cost guard (budget tracking)

---

## Part 5: Scaling & Reliability (40 phút)

###  Concepts

**Vấn đề:** 1 instance không đủ khi có nhiều users.

**Giải pháp:**
1. **Stateless design** — Không lưu state trong memory
2. **Health checks** — Platform biết khi nào restart
3. **Graceful shutdown** — Hoàn thành requests trước khi tắt
4. **Load balancing** — Phân tán traffic

###  Exercise 5.1: Health checks

```bash
cd ../../05-scaling-reliability/develop
```

**Nhiệm vụ:** Implement 2 endpoints:

```python
@app.get("/health")
def health():
    """Liveness probe — container còn sống không?"""
    # TODO: Return 200 nếu process OK
    pass

@app.get("/ready")
def ready():
    """Readiness probe — sẵn sàng nhận traffic không?"""
    # TODO: Check database connection, Redis, etc.
    # Return 200 nếu OK, 503 nếu chưa ready
    pass
```

<details>
<summary> Solution</summary>

```python
@app.get("/health")
def health():
    return {"status": "ok"}

@app.get("/ready")
def ready():
    try:
        # Check Redis
        r.ping()
        # Check database
        db.execute("SELECT 1")
        return {"status": "ready"}
    except:
        return JSONResponse(
            status_code=503,
            content={"status": "not ready"}
        )
```

</details>

###  Exercise 5.2: Graceful shutdown

**Nhiệm vụ:** Implement signal handler:

```python
import signal
import sys

def shutdown_handler(signum, frame):
    """Handle SIGTERM from container orchestrator"""
    global _is_ready
    
    # 1. Stop accepting new requests
    _is_ready = False
    logger.info("🔄 Graceful shutdown initiated...")
    
    # 2. Finish current requests (chờ in_flight_requests → 0)
    timeout = 30
    elapsed = 0
    while _in_flight_requests > 0 and elapsed < timeout:
        logger.info(f"Waiting for {_in_flight_requests} in-flight requests...")
        time.sleep(1)
        elapsed += 1
    
    # 3. Close connections (close Redis, DB, etc)
    try:
        r.close()  # Close Redis
        logger.info("✅ Redis connection closed")
    except:
        pass
    
    # 4. Exit gracefully
    logger.info("✅ Shutdown complete")
    sys.exit(0)

signal.signal(signal.SIGTERM, shutdown_handler)
```

**Giải thích:**
- `_is_ready = False` → /ready endpoint trả về 503, load balancer stop route traffic
- Chờ requests hiện tại hoàn thành (tối đa 30s)
- Close connections (Redis, database)
- Exit với code 0 (success exit)

Test:
```bash
python app.py &
PID=$!

# Gửi request
curl http://localhost:8000/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Long task"}' &

# Ngay lập tức kill
kill -TERM $PID

# Quan sát: Request có hoàn thành không?
```

###  Exercise 5.3: Stateless design

```bash
cd ../production
```

**Nhiệm vụ:** Refactor code để stateless.

**Anti-pattern:**
```python
#  State trong memory
conversation_history = {}

@app.post("/ask")
def ask(user_id: str, question: str):
    history = conversation_history.get(user_id, [])
    # ...
    conversation_history[user_id] = updated_history  # Lưu vào memory
    
# Vấn đề: Instance 1 lưu → Instance 2 không thấy!
```

**Correct:**
```python
#  State trong Redis (GOOD — scalable)
import redis
r = redis.from_url("redis://localhost:6379/0")

def save_session(session_id: str, data: dict):
    """Lưu session vào Redis."""
    serialized = json.dumps(data)
    r.setex(f"session:{session_id}", 3600, serialized)  # TTL 1 giờ

def load_session(session_id: str) -> dict:
    """Load session từ Redis."""
    data = r.get(f"session:{session_id}")
    return json.loads(data) if data else {}

@app.post("/ask")
def ask(user_id: str, question: str):
    history = r.lrange(f"history:{user_id}", 0, -1)
    # ...
```

Tại sao? Vì khi scale ra nhiều instances, mỗi instance có memory riêng.

###  Exercise 5.4: Load balancing

**Nhiệm vụ:** Chạy stack với Nginx load balancer:

```bash
docker compose up --scale agent=3
```

**Architecture:**
```
┌──────────┐
│  Client  │
└────┬─────┘
     │ HTTP requests
     ▼
┌───────────┐              ┌──────────┐
│   Nginx   │ ─reverse proxy─│ Agent 1  │ (port 8000)
│ port 8080 │              ├──────────┤
│ (LB)      │              │ Agent 2  │ (port 8000)
│           │              ├──────────┤
│ Round-    │              │ Agent 3  │ (port 8000)
│ robin     │              └──────────┘
│ algo      │                   ▲
│           │                   │
│upstream   │                   │
│agent:8000 ├───────────────────┘
└───────────┘
     │
     ├──────────────────┐
     ▼                  ▼
  ┌──────────┐    ┌──────────┐
  │  Redis   │    │ Network  │
  │ 6379     │    │ bridge   │
  └──────────┘    └──────────┘
```

**Observations:**
- **3 agent instances** được start (`--scale agent=3`)
- **Nginx** act as reverse proxy + load balancer
- **Round-robin**: Request 1→Agent1, Request 2→Agent2, Request 3→Agent3, Request 4→Agent1, ...
- **Docker Compose DNS**: `upstream agent:8000` → Docker DNS tự resolve sang tất cả agent instances
- **Healthcheck**: Nginx tự skip instances không healthy
- **Stateless**: Mỗi request có thể đi đến instance khác, nhưng session trong Redis → consistent

**Nginx configuration (từ nginx.conf):**
```nginx
upstream agent_cluster {
    server agent:8000;  # Docker DNS → resolve sang agent1, agent2, agent3
    keepalive 16;       # Connection pooling
}

server {
    listen 80;
    
    location / {
        proxy_pass http://agent_cluster;
        proxy_next_upstream error timeout http_503;  # Retry nếu fail
        proxy_next_upstream_tries 3;                 # Tối đa 3 tries
    }
}
```

Quan sát:
- **X-Served-By header**: Mỗi response có header `X-Served-By: <instance_ip>` → thấy rõ load phân tán
- **3 agent instances được start**, Nginx xử lý traffic, Redis shared state
- **Nếu 1 instance die**, Nginx tự skip → traffic vẫn tiếp tục đến 2 instances còn lại

Test (xem trong test section bên dưới)

###  Exercise 5.5: Test stateless

```bash
python test_stateless.py
```

Script này:
1. Gọi API để tạo conversation
2. Kill random instance
3. Gọi tiếp — conversation vẫn còn không?

###  Checkpoint 5

- [ ] Implement health và readiness checks
- [ ] Implement graceful shutdown
- [ ] Refactor code thành stateless
- [ ] Hiểu load balancing với Nginx
- [ ] Test stateless design

---

## Part 6: Final Project (60 phút)

###  Objective

Build một production-ready AI agent từ đầu, kết hợp TẤT CẢ concepts đã học.

###  Requirements

**Functional:**
- [ ] Agent trả lời câu hỏi qua REST API
- [ ] Support conversation history
- [ ] Streaming responses (optional)

**Non-functional:**
- [ ] Dockerized với multi-stage build
- [ ] Config từ environment variables
- [ ] API key authentication
- [ ] Rate limiting (10 req/min per user)
- [ ] Cost guard ($10/month per user)
- [ ] Health check endpoint
- [ ] Readiness check endpoint
- [ ] Graceful shutdown
- [ ] Stateless design (state trong Redis)
- [ ] Structured JSON logging
- [ ] Deploy lên Railway hoặc Render
- [ ] Public URL hoạt động

### 🏗 Architecture

```
┌─────────────┐
│   Client    │
└──────┬──────┘
       │
       ▼
┌─────────────────┐
│  Nginx (LB)     │
└──────┬──────────┘
       │
       ├─────────┬─────────┐
       ▼         ▼         ▼
   ┌──────┐  ┌──────┐  ┌──────┐
   │Agent1│  │Agent2│  │Agent3│
   └───┬──┘  └───┬──┘  └───┬──┘
       │         │         │
       └─────────┴─────────┘
                 │
                 ▼
           ┌──────────┐
           │  Redis   │
           └──────────┘
```

###  Step-by-step

#### Step 1: Project setup (5 phút)

```bash
mkdir my-production-agent
cd my-production-agent

# Tạo structure
mkdir -p app
touch app/__init__.py
touch app/main.py
touch app/config.py
touch app/auth.py
touch app/rate_limiter.py
touch app/cost_guard.py
touch Dockerfile
touch docker-compose.yml
touch requirements.txt
touch .env.example
touch .dockerignore
```

#### Step 2: Config management (10 phút)

**File:** `app/config.py`

```python
from pydantic_settings import BaseSettings

class Settings(BaseSettings):
    # TODO: Define all config
    # - PORT
    # - REDIS_URL
    # - AGENT_API_KEY
    # - LOG_LEVEL
    # - RATE_LIMIT_PER_MINUTE
    # - MONTHLY_BUDGET_USD
    pass

settings = Settings()
```

#### Step 3: Main application (15 phút)

**File:** `app/main.py`

```python
from fastapi import FastAPI, Depends, HTTPException
from .config import settings
from .auth import verify_api_key
from .rate_limiter import check_rate_limit
from .cost_guard import check_budget

app = FastAPI()

@app.get("/health")
def health():
    # TODO
    pass

@app.get("/ready")
def ready():
    # TODO: Check Redis connection
    pass

@app.post("/ask")
def ask(
    question: str,
    user_id: str = Depends(verify_api_key),
    _rate_limit: None = Depends(check_rate_limit),
    _budget: None = Depends(check_budget)
):
    # TODO: 
    # 1. Get conversation history from Redis
    # 2. Call LLM
    # 3. Save to Redis
    # 4. Return response
    pass
```

#### Step 4: Authentication (5 phút)

**File:** `app/auth.py`

```python
from fastapi import Header, HTTPException

def verify_api_key(x_api_key: str = Header(...)):
    # TODO: Verify against settings.AGENT_API_KEY
    # Return user_id if valid
    # Raise HTTPException(401) if invalid
    pass
```

#### Step 5: Rate limiting (10 phút)

**File:** `app/rate_limiter.py`

```python
import redis
from fastapi import HTTPException

r = redis.from_url(settings.REDIS_URL)

def check_rate_limit(user_id: str):
    # TODO: Implement sliding window
    # Raise HTTPException(429) if exceeded
    pass
```

#### Step 6: Cost guard (10 phút)

**File:** `app/cost_guard.py`

```python
def check_budget(user_id: str):
    # TODO: Check monthly spending
    # Raise HTTPException(402) if exceeded
    pass
```

#### Step 7: Dockerfile (5 phút)

```dockerfile
# TODO: Multi-stage build
# Stage 1: Builder
# Stage 2: Runtime
```

#### Step 8: Docker Compose (5 phút)

```yaml
# TODO: Define services
# - agent (scale to 3)
# - redis
# - nginx (load balancer)
```

#### Step 9: Test locally (5 phút)

```bash
docker compose up --scale agent=3

# Test all endpoints
curl http://localhost/health
curl http://localhost/ready
curl -H "X-API-Key: secret" http://localhost/ask -X POST \
  -H "Content-Type: application/json" \
  -d '{"question": "Hello", "user_id": "user1"}'
```

#### Step 10: Deploy (10 phút)

```bash
# Railway
railway init
railway variables set REDIS_URL=...
railway variables set AGENT_API_KEY=...
railway up

# Hoặc Render
# Push lên GitHub → Connect Render → Deploy
```

###  Validation

Chạy script kiểm tra:

```bash
cd 06-lab-complete
python check_production_ready.py
```

Script sẽ kiểm tra:
-  Dockerfile exists và valid
-  Multi-stage build
-  .dockerignore exists
-  Health endpoint returns 200
-  Readiness endpoint returns 200
-  Auth required (401 without key)
-  Rate limiting works (429 after limit)
-  Cost guard works (402 when exceeded)
-  Graceful shutdown (SIGTERM handled)
-  Stateless (state trong Redis, không trong memory)
-  Structured logging (JSON format)

###  Grading Rubric

| Criteria | Points | Description |
|----------|--------|-------------|
| **Functionality** | 20 | Agent hoạt động đúng |
| **Docker** | 15 | Multi-stage, optimized |
| **Security** | 20 | Auth + rate limit + cost guard |
| **Reliability** | 20 | Health checks + graceful shutdown |
| **Scalability** | 15 | Stateless + load balanced |
| **Deployment** | 10 | Public URL hoạt động |
| **Total** | 100 | |

---

##  Hoàn Thành!

Bạn đã:
-  Hiểu sự khác biệt dev vs production
-  Containerize app với Docker
-  Deploy lên cloud platform
-  Bảo mật API
-  Thiết kế hệ thống scalable và reliable

###  Next Steps

1. **Monitoring:** Thêm Prometheus + Grafana
2. **CI/CD:** GitHub Actions auto-deploy
3. **Advanced scaling:** Kubernetes
4. **Observability:** Distributed tracing với OpenTelemetry
5. **Cost optimization:** Spot instances, auto-scaling

###  Resources

- [12-Factor App](https://12factor.net/)
- [Docker Best Practices](https://docs.docker.com/develop/dev-best-practices/)
- [FastAPI Deployment](https://fastapi.tiangolo.com/deployment/)
- [Railway Docs](https://docs.railway.app/)
- [Render Docs](https://render.com/docs)

---

##  Q&A

**Q: Tôi không có credit card, có thể deploy không?**  
A: Có! Railway cho $5 credit, Render có 750h free tier.

**Q: Mock LLM khác gì với OpenAI thật?**  
A: Mock trả về canned responses, không gọi API. Để dùng OpenAI thật, set `OPENAI_API_KEY` trong env.

**Q: Làm sao debug khi container fail?**  
A: `docker logs <container_id>` hoặc `docker exec -it <container_id> /bin/sh`

**Q: Redis data mất khi restart?**  
A: Dùng volume: `volumes: - redis-data:/data` trong docker-compose.

**Q: Làm sao scale trên Railway/Render?**  
A: Railway: `railway scale <replicas>`. Render: Dashboard → Settings → Instances.

---

**Happy Deploying! **
