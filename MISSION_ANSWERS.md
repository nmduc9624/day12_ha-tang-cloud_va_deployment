# Day 12 Lab - Mission Answers

## Part 1: Localhost vs Production

### Exercise 1.1: Anti-patterns found

I reviewed `01-localhost-vs-production/develop/app.py` and found these production anti-patterns:

1. The OpenAI API key is hardcoded directly in source code as `OPENAI_API_KEY = "sk-hardcoded-fake-key-never-do-this"`. If this code is pushed to GitHub, the secret is exposed.
2. The database URL is also hardcoded and contains username/password information: `postgresql://admin:password123@localhost:5432/mydb`.
3. There is no proper configuration management. Values like `DEBUG`, `MAX_TOKENS`, host, and port are fixed in code instead of being loaded from environment variables.
4. The app logs sensitive information with `print(f"[DEBUG] Using key: {OPENAI_API_KEY}")`, which can leak secrets into terminal logs or cloud logs.
5. There is no `/health` endpoint, so a cloud platform or load balancer cannot check whether the app is alive.
6. The server binds to `localhost`, which only accepts local connections and is not suitable inside a container or cloud environment.
7. The port is hardcoded to `8000`. Platforms like Railway and Render inject the port through the `PORT` environment variable.
8. `reload=True` is enabled, which is useful during development but not appropriate for production.
9. There is no readiness endpoint to tell a load balancer whether the app is ready to receive traffic.
10. There is no graceful shutdown handling, so in-flight requests could be interrupted when the platform stops or restarts the service.

### Exercise 1.2: Run basic version

The basic localhost version was run from `01-localhost-vs-production/develop` using:

```bash
python app.py
```

The `/ask` endpoint works when the question is passed as a query parameter.

Test commands:

```bash
curl -X POST "http://localhost:8000/ask?question=Hello"
curl -X POST "http://localhost:8000/ask?question=What%20is%20deployment"
```

Observed results:

```json
{"answer":"Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ OpenAI/Anthropic."}
{"answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được."}
```

I also tested the health endpoint:

```bash
curl http://localhost:8000/health
```

Observed result:

```json
{"detail":"Not Found"}
```

Conclusion: the basic app can answer questions locally, but it is not production-ready because it does not provide a `/health` endpoint for monitoring or cloud platform health checks.

### Exercise 1.3: Comparison table

I compared `01-localhost-vs-production/develop/app.py` with `01-localhost-vs-production/production/app.py` and `production/config.py`.

| Feature | Basic | Advanced | Why Important? |
|---|---|---|---|
| Config | Hardcoded values in `app.py` | Centralized config in `config.py`, loaded from environment variables | Makes the app portable across local, staging, and production without changing source code |
| Secrets | API key and database URL are written directly in code | Secrets are read from environment variables such as `OPENAI_API_KEY` and `AGENT_API_KEY` | Prevents leaking credentials in GitHub, logs, or shared source code |
| Host binding | Uses `host="localhost"` | Uses `settings.host`, default `0.0.0.0` | Containers and cloud platforms need the app to accept external connections |
| Port | Fixed to `8000` | Uses `settings.port`, loaded from the `PORT` environment variable | Platforms like Railway/Render provide the runtime port dynamically |
| Health check | Missing `/health` endpoint | Provides `/health` with status, uptime, version, environment, and timestamp | Cloud platforms use health checks to detect crashes and restart unhealthy services |
| Readiness check | Missing `/ready` endpoint | Provides `/ready` and returns 503 when the app is not ready | Load balancers should only send traffic to instances that are ready |
| Logging | Uses `print()` and logs secrets | Uses structured JSON logging and avoids logging secrets | Production logs must be searchable, parseable, and safe |
| Debug/reload | Always runs with `reload=True` | Reload is controlled by `settings.debug` | Auto-reload is useful locally but should be disabled in production |
| Shutdown | No explicit lifecycle/shutdown logic | Uses FastAPI lifespan and SIGTERM handling for graceful shutdown | Helps finish in-flight requests and clean up resources during deploy/restart |
| CORS | No CORS configuration | Adds CORS middleware with origins from config | Production APIs often need controlled browser access |
| Error handling | Relies on default behavior | Returns clear 422 when `question` is missing | Clear errors make APIs easier to test and debug |

Overall conclusion: the basic version proves the agent can run locally, while the advanced version follows production practices: 12-factor configuration, safer secrets handling, health/readiness endpoints, structured logging, cloud-compatible host/port settings, and graceful shutdown behavior.
## Part 2: Docker

### Exercise 2.1: Dockerfile questions

I reviewed `02-docker/develop/Dockerfile`.

1. Base image: `python:3.11`. This image contains a full Python 3.11 runtime and OS userspace. It is easy to use for learning, but it is large.
2. Working directory: `/app`, defined by `WORKDIR /app`. All following commands such as `COPY`, `RUN`, and `CMD` run relative to this directory.
3. `COPY requirements.txt` is done before copying the application source code to take advantage of Docker layer caching. If only `app.py` changes but `requirements.txt` stays the same, Docker can reuse the dependency installation layer instead of running `pip install` again.
4. `CMD` provides the default command when a container starts and can be overridden at `docker run` time. `ENTRYPOINT` defines the main executable for the container and is usually harder to override. In this lab, `CMD ["python", "app.py"]` is enough because the container simply starts the Python app.

### Exercise 2.2: Build and run basic container

I built the develop Docker image from the project root:

```bash
docker build -f 02-docker/develop/Dockerfile -t my-agent:develop .
```

Then I ran the container:

```bash
docker run -p 8000:8000 my-agent:develop
```

The container was confirmed running:

```text
CONTAINER ID   IMAGE              PORTS                                         STATUS       NAMES
c67dab706738   my-agent:develop   0.0.0.0:8000->8000/tcp, [::]:8000->8000/tcp   Up 5 min     intelligent_edison
```

Test command:

```bash
curl -X POST "http://localhost:8000/ask?question=What%20is%20Docker"
```

Observed result:

```json
{"answer":"Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!"}
```

Image size:

```text
IMAGE              ID             DISK USAGE   CONTENT SIZE
my-agent:develop   9fec2236d2be       1.66GB          424MB
```

Conclusion: the basic Docker image works, but it is large because it uses the full `python:3.11` base image and a single-stage build.

### Exercise 2.3: Multi-stage build

I reviewed `02-docker/production/Dockerfile`.

Stage 1 is the `builder` stage:

- Uses `python:3.11-slim AS builder`.
- Installs build tools such as `gcc` and `libpq-dev`.
- Installs Python dependencies with `pip install --user` into `/root/.local`.
- This stage is only used for building dependencies and is not shipped as the final runtime image.

Stage 2 is the `runtime` stage:

- Uses `python:3.11-slim AS runtime`.
- Creates a non-root `appuser` for better security.
- Copies only installed packages from the builder stage.
- Copies only the application code and `utils/mock_llm.py`.
- Exposes port `8000` and defines a Docker `HEALTHCHECK`.
- Starts the app with `uvicorn`.

Build command:

```bash
docker build -f 02-docker/production/Dockerfile -t my-agent:advanced .
```

Image size comparison:

```text
IMAGE               DISK USAGE   CONTENT SIZE
my-agent:develop        1.66GB          424MB
my-agent:advanced        236MB         56.6MB
```

The advanced image is much smaller. Disk usage was reduced from about `1.66GB` to `236MB`, roughly an 86% reduction. Content size was reduced from `424MB` to `56.6MB`, also roughly an 87% reduction.

Why it is smaller:

- It uses `python:3.11-slim` instead of full `python:3.11`.
- Build tools are kept in the builder stage and are not copied into the final runtime image.
- Only the required runtime packages and application files are included.

### Exercise 2.4: Docker Compose stack

I reviewed `02-docker/production/docker-compose.yml` and ran the stack from `02-docker/production`:

```bash
docker compose up -d
```

The compose stack starts these services:

| Service | Purpose |
|---|---|
| `agent` | FastAPI AI agent running on internal port `8000` |
| `redis` | Cache/session/rate-limit backend |
| `qdrant` | Vector database service for RAG-style architecture |
| `nginx` | Reverse proxy and load balancer exposed on host ports `80` and `443` |

Observed compose status:

```text
NAME                  IMAGE                  SERVICE   STATUS
production-agent-1    production-agent       agent     Up (healthy)
production-nginx-1    nginx:alpine           nginx     Up, 0.0.0.0:80->80/tcp, 0.0.0.0:443->443/tcp
production-qdrant-1   qdrant/qdrant:v1.9.0   qdrant    Up (healthy)
production-redis-1    redis:7-alpine         redis     Up (healthy)
```

Architecture diagram:

```text
Client
  |
  v
Nginx reverse proxy / load balancer (host port 80)
  |
  v
Agent service (FastAPI, internal port 8000)
  |\
  | \__ Redis (cache/session/rate limiting)
  |
  \____ Qdrant (vector database)
```

Health check through Nginx:

```bash
curl http://localhost/health
```

Observed result:

```json
{"status":"ok","uptime_seconds":8.5,"version":"2.0.0","timestamp":"2026-06-12T08:45:18.355442"}
```

Agent endpoint through Nginx requires JSON body in the production version:

```bash
curl -X POST http://localhost/ask \
  -H "Content-Type: application/json" \
  -d '{"question":"Explain microservices"}'
```

Observed result:

```json
{"answer":"Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."}
```

Conclusion: Docker Compose successfully orchestrates the full production stack. The client talks to Nginx, Nginx forwards requests to the agent, and the agent can use internal services such as Redis and Qdrant through the Docker network.
## Part 3: Cloud Deployment

### Exercise 3.1: Railway deployment

I deployed the Railway example from `03-cloud-deployment/railway`.

Deployment commands used:

```bash
railway login
railway init
railway variables set PORT=8000
railway variables set AGENT_API_KEY=my-secret-key
railway up
railway domain
```

Railway build and deployment completed successfully. The deployment log showed:

```text
Deploy complete
Uvicorn running on http://0.0.0.0:8080
GET /health HTTP/1.1 200 OK
Healthcheck succeeded
```

Public URL:

```text
https://day12-production-90b5.up.railway.app
```

Health check test:

```bash
curl https://day12-production-90b5.up.railway.app/health
```

Observed result:

```json
{"status":"ok","uptime_seconds":1081.3,"platform":"Railway","timestamp":"2026-06-12T09:26:23.750043+00:00"}
```

Agent endpoint test:

```powershell
$body = @{ question = "Am I on cloud?" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "https://day12-production-90b5.up.railway.app/ask" -ContentType "application/json" -Body $body
```

Observed result:

```json
{
  "question": "Am I on cloud?",
  "answer": "Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận.",
  "platform": "Railway"
}
```

Note: in PowerShell, Vietnamese text may appear with broken encoding such as `Agent Äang...`, but the API response is valid and the deployment works correctly.

Conclusion: Exercise 3.1 is complete. The app is deployed on Railway, has a working public URL, passes the `/health` check, and responds to `/ask` through the cloud endpoint.

### Exercise 3.2: Render deployment / config comparison

I compared `03-cloud-deployment/railway/railway.toml` with `03-cloud-deployment/render/render.yaml`.

| Aspect | Railway | Render |
|---|---|---|
| Config file | `railway.toml` | `render.yaml` |
| Format | TOML | YAML |
| Main purpose | Configure Railway build and deploy behavior | Define Render infrastructure as code using a Blueprint |
| Build system | Uses Nixpacks auto-detection with `builder = "NIXPACKS"` | Explicitly defines `buildCommand: pip install -r requirements.txt` |
| Start command | `uvicorn app:app --host 0.0.0.0 --port $PORT` | `uvicorn app:app --host 0.0.0.0 --port $PORT` |
| Health check | `healthcheckPath = "/health"` | `healthCheckPath: /health` |
| Runtime/service definition | Railway infers most service details | Render explicitly defines a web service with name, runtime, region, plan, build command, start command, health check, and auto-deploy |
| Environment variables | Usually set through Railway CLI or Dashboard, for example `railway variables set ...` | Declared in `envVars`; secrets can use `sync: false` or `generateValue: true` |
| Extra services | This `railway.toml` only configures the web service deploy | `render.yaml` also declares a Redis service named `agent-cache` |
| Deployment flow | CLI-first: `railway up` deploys directly from the project | GitHub-first: push code, create a Render Blueprint, connect the repo, and Render reads `render.yaml` |

Important details in `render.yaml`:

- The web service is named `ai-agent`.
- It uses the Python runtime.
- It deploys in the `singapore` region.
- It uses the `free` plan.
- It runs `pip install -r requirements.txt` during build.
- It starts with `uvicorn app:app --host 0.0.0.0 --port $PORT`.
- It checks `/health` for service health.
- It enables `autoDeploy: true`, so pushes to GitHub can trigger redeploys.
- It declares `ENVIRONMENT=production` and `PYTHON_VERSION=3.11.0`.
- It avoids committing real secrets: `OPENAI_API_KEY` uses `sync: false`, and `AGENT_API_KEY` uses `generateValue: true`.
- It declares a Redis service with `maxmemoryPolicy: allkeys-lru`.

Conclusion: Railway is simpler and faster for CLI-based deployment and prototyping. Render's Blueprint is more explicit and repeatable because it describes the web service, environment variables, health check, region, plan, auto-deploy behavior, and Redis service in one infrastructure-as-code file.

### Exercise 3.3: GCP Cloud Run CI/CD

I reviewed `03-cloud-deployment/production-cloud-run/cloudbuild.yaml` and `03-cloud-deployment/production-cloud-run/service.yaml`.

`cloudbuild.yaml` defines a CI/CD pipeline for Google Cloud Build:

1. Test step: uses `python:3.11-slim`, installs dependencies and `pytest`, then runs `pytest tests/ -v --tb=short`.
2. Build step: uses `gcr.io/cloud-builders/docker` to build a Docker image tagged as both `gcr.io/$PROJECT_ID/ai-agent:$COMMIT_SHA` and `gcr.io/$PROJECT_ID/ai-agent:latest`.
3. Push step: pushes all image tags to Google Container Registry.
4. Deploy step: uses `gcloud run deploy ai-agent` to deploy the commit-specific image to Cloud Run in `asia-southeast1`.
5. Runtime configuration: sets Cloud Run options such as public access, min/max instances, memory, CPU, timeout, environment variables, and secrets.

Important Cloud Build settings:

- `--allow-unauthenticated` makes the service publicly accessible.
- `--min-instances=1` reduces cold starts.
- `--max-instances=10` limits maximum scale and helps control cost.
- `--memory=512Mi` and `--cpu=1` define resource limits.
- `--set-env-vars=ENVIRONMENT=production` sets runtime config.
- `--set-secrets=OPENAI_API_KEY=openai-key:latest` reads secrets from Google Secret Manager instead of hardcoding them.
- `timeout: 1200s` gives the build up to 20 minutes.

`service.yaml` defines the Cloud Run service as infrastructure-as-code:

- Service name: `ai-agent`.
- Public ingress is enabled with `run.googleapis.com/ingress: all`.
- Autoscaling is configured with `minScale: "1"` and `maxScale: "10"`.
- Each instance can handle up to `containerConcurrency: 80` requests.
- Request timeout is `timeoutSeconds: 60`.
- The container image is `gcr.io/PROJECT_ID/ai-agent:latest`.
- The container exposes port `8000`.
- CPU/memory limits are set to `cpu: "1"` and `memory: 512Mi`.
- CPU/memory requests are set to `cpu: "0.5"` and `memory: 256Mi`.
- Environment variables include `ENVIRONMENT=production` and `PORT=8000`.
- Secrets such as `OPENAI_API_KEY` and `AGENT_API_KEY` are loaded from Secret Manager.
- A liveness probe checks `/health` on port `8000`.
- A startup probe checks `/ready` on port `8000`.

Comparison with Railway and Render:

- Railway is the fastest and simplest for this lab because `railway up` handles most build/deploy details.
- Render is more explicit than Railway through `render.yaml`, and it can define both the web service and Redis.
- Cloud Run is the most production-oriented option because it defines CI/CD, image versioning, autoscaling, resource limits, health probes, startup probes, and Secret Manager integration.

Conclusion: GCP Cloud Run requires more setup than Railway or Render, but it gives the most control over production deployment, scaling, resource usage, secrets, and CI/CD automation.
## Part 4: API Security

### Exercise 4.1: API Key authentication

I tested the basic API key authentication example in `04-api-gateway/develop/app.py`.

The app reads the expected API key from the `AGENT_API_KEY` environment variable:

```python
API_KEY = os.getenv("AGENT_API_KEY", "demo-key-change-in-production")
```

The API key is checked in the `verify_api_key()` dependency. It reads the `X-API-Key` request header using `APIKeyHeader`:

```python
api_key_header = APIKeyHeader(name="X-API-Key", auto_error=False)
```

Test setup:

```powershell
$env:AGENT_API_KEY="lab12-demo-key"
python -m uvicorn app:app --host 127.0.0.1 --port 8000
```

Health check:

```powershell
Invoke-RestMethod -Method Get -Uri "http://localhost:8000/health"
```

Observed result:

```json
{"status":"ok"}
```

Test 1: call `/ask` without an API key:

```powershell
$body = @{ question = "Hello" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "http://localhost:8000/ask" -ContentType "application/json" -Body $body
```

Observed result:

```json
{"StatusCode":401,"StatusDescription":"Unauthorized"}
```

This happens because the `X-API-Key` header is missing.

Test 2: call `/ask` with the wrong API key:

```powershell
$body = @{ question = "Hello" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "http://localhost:8000/ask" -Headers @{ "X-API-Key" = "wrong-key" } -ContentType "application/json" -Body $body
```

Observed result:

```json
{"StatusCode":403,"StatusDescription":"Forbidden"}
```

This happens because the provided key does not match `AGENT_API_KEY`.

Test 3: call `/ask` with the correct API key:

```powershell
$body = @{ question = "Hello" } | ConvertTo-Json
Invoke-RestMethod -Method Post -Uri "http://localhost:8000/ask" -Headers @{ "X-API-Key" = "lab12-demo-key" } -ContentType "application/json" -Body $body
```

Observed result:

```json
{
  "question": "Hello",
  "answer": "Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận."
}
```

Note: PowerShell may display Vietnamese text with broken encoding, but the API response itself is valid.

How to rotate the API key:

1. Generate or choose a new value for `AGENT_API_KEY`.
2. Update the environment variable locally or in the cloud platform dashboard.
3. Restart/redeploy the service.
4. Update clients to send the new value in the `X-API-Key` header.

Conclusion: Exercise 4.1 is complete. The `/ask` endpoint is protected by API key authentication: missing key returns `401`, invalid key returns `403`, and a valid key allows the request.
### Exercise 4.2: JWT authentication

I tested the advanced JWT authentication flow in `04-api-gateway/production`.

The app exposes a public token endpoint:

```text
POST /auth/token
```

This endpoint validates username/password and returns a JWT access token. I used the demo credentials from the app:

```text
student / demo123
```

Token request:

```powershell
$login = @{ username = "student"; password = "demo123" } | ConvertTo-Json
$tokenResponse = Invoke-RestMethod -Method Post -Uri "http://localhost:8000/auth/token" -ContentType "application/json" -Body $login
$token = $tokenResponse.access_token
$token
```

Observed result: a JWT token was returned successfully.

```text
eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

Then I used the token to call the protected `/ask` endpoint:

```powershell
$body = @{ question = "Explain JWT" } | ConvertTo-Json
Invoke-RestMethod -Method Post `
  -Uri "http://localhost:8000/ask" `
  -Headers @{ Authorization = "Bearer $token" } `
  -ContentType "application/json" `
  -Body $body
```

Observed result:

```text
question    answer                                          usage
--------    ------                                          -----
Explain JWT <mock agent answer>                             @{requests_remaining=9; ...}
```

This confirms that a valid JWT token allows access to the protected agent endpoint. The response also includes usage information, including remaining requests.

I also tested calling `/ask` without the token:

```powershell
$body = @{ question = "Explain JWT" } | ConvertTo-Json
Invoke-RestMethod -Method Post `
  -Uri "http://localhost:8000/ask" `
  -ContentType "application/json" `
  -Body $body
```

Observed result:

```json
{"detail":"Authentication required. Include: Authorization: Bearer "}
```

Conclusion: Exercise 4.2 is complete. The production app uses JWT authentication correctly: users first obtain a token from `/auth/token`, then call protected endpoints with `Authorization: Bearer <token>`. Requests without a token are rejected.
### Exercise 4.3: Rate limiting

I reviewed `04-api-gateway/production/rate_limiter.py` and tested rate limiting with the `student` user.

The rate limiter uses a Sliding Window Counter algorithm:

- Each user has a window of request timestamps.
- Old timestamps outside the 60-second window are removed.
- If the number of active timestamps reaches the limit, the request is rejected.
- The API returns `429 Too Many Requests` when the user exceeds the limit.

Configured limits:

```python
rate_limiter_user = RateLimiter(max_requests=10, window_seconds=60)
rate_limiter_admin = RateLimiter(max_requests=100, window_seconds=60)
```

So regular users are limited to 10 requests per minute, while admins are limited to 100 requests per minute.

Test setup:

```powershell
$login = @{ username = "student"; password = "demo123" } | ConvertTo-Json
$tokenResponse = Invoke-RestMethod -Method Post -Uri "http://localhost:8000/auth/token" -ContentType "application/json" -Body $login
$token = $tokenResponse.access_token
```

Test loop:

```powershell
for ($i = 1; $i -le 20; $i++) {
  Write-Host "Request $i"
  $body = @{ question = "Rate limit test $i" } | ConvertTo-Json

  try {
    $res = Invoke-RestMethod -Method Post `
      -Uri "http://localhost:8000/ask" `
      -Headers @{ Authorization = "Bearer $token" } `
      -ContentType "application/json" `
      -Body $body

    Write-Host "OK - remaining:" $res.usage.requests_remaining
  } catch {
    $status = $_.Exception.Response.StatusCode.value__
    Write-Host "BLOCKED - status:" $status
  }
}
```

Observed result:

```text
Request 1  -> OK - remaining: 9
Request 2  -> OK - remaining: 8
Request 3  -> OK - remaining: 7
Request 4  -> OK - remaining: 6
Request 5  -> OK - remaining: 5
Request 6  -> OK - remaining: 4
Request 7  -> OK - remaining: 3
Request 8  -> OK - remaining: 2
Request 9  -> OK - remaining: 1
Request 10 -> OK - remaining: 0
Request 11 -> BLOCKED - status: 429
Request 12 -> BLOCKED - status: 429
Request 13 -> BLOCKED - status: 429
Request 14 -> BLOCKED - status: 429
Request 15 -> BLOCKED - status: 429
Request 16 -> BLOCKED - status: 429
Request 17 -> BLOCKED - status: 429
Request 18 -> BLOCKED - status: 429
Request 19 -> BLOCKED - status: 429
Request 20 -> BLOCKED - status: 429
```

How admins bypass the normal user limit:

The app does not fully bypass rate limiting for admins, but it applies a higher-limit limiter. In `app.py`, the limiter is selected by role:

```python
limiter = rate_limiter_admin if role == "admin" else rate_limiter_user
```

Therefore, `student` uses the 10 requests/minute limit, while an admin role uses the 100 requests/minute limit.

Conclusion: Exercise 4.3 is complete. Rate limiting works as expected: the first 10 requests from the `student` user succeeded, and requests 11-20 were blocked with HTTP `429`.
### Exercise 4.4: Cost guard

I reviewed and tested `04-api-gateway/production/cost_guard.py`.

The cost guard protects the service from unexpected LLM spending by tracking token usage and estimated cost.

Important constants:

```python
PRICE_PER_1K_INPUT_TOKENS = 0.00015
PRICE_PER_1K_OUTPUT_TOKENS = 0.0006
```

Configured budget limits:

```python
cost_guard = CostGuard(daily_budget_usd=1.0, global_daily_budget_usd=10.0)
```

So the demo app has:

- Per-user daily budget: `$1.00`
- Global daily budget: `$10.00`
- Warning threshold: 80% of the user budget

How it works:

1. Before calling the LLM, `cost_guard.check_budget(username)` checks whether the user or global budget has already been exceeded.
2. If the per-user budget is exceeded, the API raises HTTP `402 Payment Required`.
3. If the global budget is exceeded, the API raises HTTP `503 Service Unavailable`.
4. After the LLM call, `cost_guard.record_usage(username, input_tokens, output_tokens)` records estimated token usage and adds the estimated cost.
5. Usage is available through the protected `/me/usage` endpoint.

Test setup:

```powershell
$login = @{ username = "student"; password = "demo123" } | ConvertTo-Json
$token = (Invoke-RestMethod -Method Post -Uri "http://localhost:8000/auth/token" -ContentType "application/json" -Body $login).access_token
```

Initial usage check:

```powershell
Invoke-RestMethod -Method Get -Uri "http://localhost:8000/me/usage" -Headers @{ Authorization = "Bearer $token" }
```

Observed result before any new request:

```json
{
  "user_id": "student",
  "date": "2026-06-12",
  "requests": 0,
  "input_tokens": 0,
  "output_tokens": 0,
  "cost_usd": 0.0,
  "budget_usd": 1.0,
  "budget_remaining_usd": 1.0,
  "budget_used_pct": 0.0
}
```

Then I sent one protected `/ask` request:

```powershell
$body = @{ question = "Cost guard test request" } | ConvertTo-Json
Invoke-RestMethod -Method Post `
  -Uri "http://localhost:8000/ask" `
  -Headers @{ Authorization = "Bearer $token" } `
  -ContentType "application/json" `
  -Body $body
```

Observed response:

```json
{
  "question": "Cost guard test request",
  "answer": "<mock agent answer>",
  "usage": {
    "requests_remaining": 9,
    "budget_remaining_usd": 0.000019
  }
}
```

Note: the `/ask` response field name `budget_remaining_usd` appears to contain the current request's accumulated cost (`usage.total_cost_usd`) rather than the actual remaining budget. The `/me/usage` endpoint returns the correct budget remaining value.

Usage after the request:

```json
{
  "user_id": "student",
  "date": "2026-06-12",
  "requests": 1,
  "input_tokens": 8,
  "output_tokens": 30,
  "cost_usd": 0.000019,
  "budget_usd": 1.0,
  "budget_remaining_usd": 0.999981,
  "budget_used_pct": 0.0
}
```

This confirms that cost guard records request count, input tokens, output tokens, estimated cost, budget, and remaining budget.

Production note:

This lab implementation stores usage in memory. That is acceptable for a demo, but in production it should be stored in Redis or a database so usage survives restarts and works correctly across multiple app instances.

Conclusion: Exercise 4.4 is complete. The app checks budget before calling the LLM, records estimated token usage after the call, exposes usage through `/me/usage`, and is designed to block users with HTTP `402` once their daily budget is exceeded.


## Part 5 - Scaling & Reliability

### Exercise 5.1: Health check and readiness

I tested the `05-scaling-reliability/develop` app locally on port `8010` to avoid conflict with the earlier Part 4 server.

Commands used:

```powershell
cd D:\AI20k\lab12\day12_ha-tang-cloud_va_deployment\05-scaling-reliability\develop
$env:PORT = "8010"
.\.venv\Scripts\python.exe app.py
```

Health check result:

```json
{
  "status": "ok",
  "uptime_seconds": 13.2,
  "version": "1.0.0",
  "environment": "development",
  "timestamp": "2026-06-12T10:31:23.228242+00:00",
  "checks": {
    "memory": {
      "status": "ok",
      "used_percent": 87.9
    }
  }
}
```

Readiness result:

```json
{
  "ready": true,
  "in_flight_requests": 2
}
```

The `/health` endpoint is the liveness probe. It tells the platform whether the process is alive and healthy. The `/ready` endpoint is the readiness probe. It tells the platform whether the app is ready to receive traffic.

Conclusion: Health check and readiness work correctly. The app can report both process health and traffic readiness.

### Exercise 5.2: Graceful shutdown

I reviewed the graceful shutdown implementation in `05-scaling-reliability/develop/app.py`.

The app uses these main parts:

- `lifespan()` controls startup and shutdown.
- `_is_ready` becomes `True` after startup is complete.
- `track_requests` middleware increments `_in_flight_requests` when a request starts and decrements it when the request finishes.
- During shutdown, `_is_ready` is set to `False`, so `/ready` can stop new traffic.
- The shutdown flow waits until `_in_flight_requests == 0`, up to 30 seconds.
- Signal handlers log `SIGTERM` and `SIGINT`, then uvicorn handles the shutdown lifecycle.

Flow:

```text
Platform sends SIGTERM
-> app marks itself not ready
-> load balancer should stop routing new traffic
-> app waits for in-flight requests to finish
-> app exits only after requests complete or timeout is reached
```

Conclusion: The graceful shutdown design prevents in-flight requests from being cut off immediately when the platform stops or replaces an instance.

### Exercise 5.3: Stateless design with Redis

I reviewed the production implementation in `05-scaling-reliability/production/app.py`.

The production app stores session history outside the app process:

- Redis is configured through `REDIS_URL`.
- Session history is saved with `setex(..., ttl=3600)`.
- Session history is read back with `get(...)`.
- If Redis is unavailable, the app can fall back to in-memory storage, but the intended production path is Redis.

This solves the stateful scaling problem:

```text
Request 1 -> instance A -> save session to Redis
Request 2 -> instance B -> read same session from Redis
```

Conclusion: The app is stateless from the container perspective because conversation state is not tied to one instance. Any instance can serve the next request for the same session.

### Exercise 5.4: Production scaling with Docker Compose and Nginx

I started the production stack with 3 agent instances behind Nginx:

```powershell
cd D:\AI20k\lab12\day12_ha-tang-cloud_va_deployment\05-scaling-reliability\production
docker compose up -d --build --force-recreate --scale agent=3
```

The running services were:

```text
production-agent-1   healthy   8000/tcp
production-agent-2   healthy   8000/tcp
production-agent-3   healthy   8000/tcp
production-nginx-1   running   0.0.0.0:8080->80/tcp
production-redis-1   healthy   6379/tcp
```

Production health check through Nginx:

```json
{
  "status": "ok",
  "instance_id": "instance-1292f7",
  "uptime_seconds": 26.0,
  "storage": "redis",
  "redis_connected": true
}
```

Production readiness check through Nginx:

```json
{
  "ready": true,
  "instance": "instance-57eaa8"
}
```

The two checks were served by different instance IDs, which shows that Nginx is routing traffic across multiple app containers.

Conclusion: The production stack runs 3 agent instances behind Nginx, with Redis shared across all instances.

### Exercise 5.5: Stateless scaling test

I ran the lab test script:

```powershell
$env:PYTHONIOENCODING = "utf-8"
.\..\develop\.venv\Scripts\python.exe test_stateless.py
```

Observed output:

```text
Session ID: 4a2b0707-94ea-4a16-97c0-4a4751485a65

Request 1: [instance-1292f7]
  Q: What is Docker?
  A: Container là cách đóng gói app để chạy ở mọi nơi. Build once, run anywhere!...

Request 2: [instance-57eaa8]
  Q: Why do we need containers?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....

Request 3: [instance-68e83e]
  Q: What is Kubernetes?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....

Request 4: [instance-1292f7]
  Q: How does load balancing work?
  A: Tôi là AI agent được deploy lên cloud. Câu hỏi của bạn đã được nhận....

Request 5: [instance-57eaa8]
  Q: What is Redis used for?
  A: Đây là câu trả lời từ AI agent (mock). Trong production, đây sẽ là response từ O...

Total requests: 5
Instances used: {'instance-1292f7', 'instance-68e83e', 'instance-57eaa8'}
All requests served despite different instances.

Conversation History:
Total messages: 10
Session history preserved across all instances via Redis.
```

The same session was served by three different instances, but the conversation history still contained all 10 messages. This proves that the app can scale horizontally without losing session state.

Conclusion: Part 5 is complete. Health checks, readiness checks, graceful shutdown design, Redis-backed stateless sessions, Nginx load balancing, and 3-instance scaling were all verified successfully.

## Part 6 - Final Project: Complete Production Agent

### Final Project Objective

Part 6 combines all production deployment concepts from the lab into one production-ready AI agent project in `06-lab-complete`.

The project includes:

- Multi-stage Dockerfile
- Docker Compose stack with agent and Redis
- Environment-based configuration
- API key authentication
- Rate limiting
- Cost guard
- Health check endpoint
- Readiness endpoint
- Structured JSON logging
- Graceful shutdown signal handling
- Railway and Render deployment config

### Fixes required before runtime test

The static readiness checker initially passed, but the Docker runtime test exposed two real issues.

Issue 1: Docker build failed because the Dockerfile copied `utils/`, but `06-lab-complete/utils` did not exist.

Fix:

```text
Added: 06-lab-complete/utils/mock_llm.py
```

Issue 2: the container started but crashed with:

```text
ModuleNotFoundError: No module named 'uvicorn'
```

Reason: dependencies were installed with `pip install --user` under `/root/.local` in the builder stage, then copied to `/home/agent/.local` in the runtime stage. The runtime Python path did not include that site-packages directory.

Fix in `06-lab-complete/Dockerfile`:

```dockerfile
ENV PYTHONPATH=/app:/home/agent/.local/lib/python3.11/site-packages
```

Issue 3: `/health` and `/ready` returned HTTP 500 because Starlette `MutableHeaders` does not support `.pop()`.

Fix in `06-lab-complete/app/main.py`:

```python
if "server" in response.headers:
    del response.headers["server"]
```

### Production readiness checker

Command used:

```powershell
cd D:\AI20k\lab12\day12_ha-tang-cloud_va_deployment\06-lab-complete
$env:PYTHONIOENCODING = "utf-8"
$env:PYTHONUTF8 = "1"
C:\Users\nguye\AppData\Local\Programs\Python\Python313\python.exe check_production_ready.py
```

Result:

```text
Result: 20/20 checks passed (100%)
PRODUCTION READY
```

The checker confirmed:

- Required files exist
- `.env` is ignored by git
- No hardcoded secrets found
- `/health` and `/ready` are defined
- Authentication is implemented
- Rate limiting is implemented
- Graceful shutdown uses `SIGTERM`
- Structured JSON logging is present
- Dockerfile uses multi-stage build
- Dockerfile uses non-root user
- Dockerfile has a healthcheck
- Dockerfile uses a slim base image
- `.dockerignore` excludes `.env` and `__pycache__`

### Docker build and stack test

Command used:

```powershell
docker compose up -d --build --force-recreate
```

Final running services:

```text
06-lab-complete-agent-1   healthy   0.0.0.0:8000->8000/tcp
06-lab-complete-redis-1   healthy   6379/tcp
```

Docker image size:

```text
06-lab-complete-agent:latest   247MB
```

This satisfies the lab target of a multi-stage Docker image under 500MB.

### Runtime endpoint tests

Health check:

```powershell
curl.exe http://localhost:8000/health
```

Observed result:

```json
{
  "status": "ok",
  "version": "1.0.0",
  "environment": "staging",
  "uptime_seconds": 9.5,
  "total_requests": 2,
  "checks": {
    "llm": "mock"
  },
  "timestamp": "2026-06-12T10:56:11.679516+00:00"
}
```

Readiness check:

```powershell
curl.exe http://localhost:8000/ready
```

Observed result:

```json
{
  "ready": true
}
```

Authenticated `/ask` request:

```powershell
$body = @{ question = "What is deployment?" } | ConvertTo-Json -Compress
Invoke-RestMethod -Method Post `
  -Uri "http://localhost:8000/ask" `
  -Headers @{ "X-API-Key" = "part6-demo-key" } `
  -ContentType "application/json" `
  -Body $body
```

Observed result:

```json
{
  "question": "What is deployment?",
  "answer": "Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.",
  "model": "gpt-4o-mini",
  "timestamp": "2026-06-12T10:56:11.856680+00:00"
}
```

Authentication test without API key:

```text
HTTP 401
{"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
```

Metrics endpoint with API key:

```json
{
  "uptime_seconds": 21.3,
  "total_requests": 3,
  "error_count": 0,
  "daily_cost_usd": 0.0,
  "daily_budget_usd": 5.0,
  "budget_used_pct": 0.0
}
```

Rate limit runtime test:

```text
Request 1 OK
Request 2 OK
...
Request 19 OK
Request 20 BLOCKED 429
Request 21 BLOCKED 429
Request 22 BLOCKED 429
Request 23 BLOCKED 429
Request 24 BLOCKED 429
Request 25 BLOCKED 429
```

The first blocked request appeared at request 20 because the same API key already had previous successful requests in the current 60-second rate-limit window.

### Deployment readiness

The project includes both deployment configuration files:

- `railway.toml` for Railway
- `render.yaml` for Render

For Railway, the required variables are:

```text
AGENT_API_KEY=<secret value>
JWT_SECRET=<secret value>
OPENAI_API_KEY=<optional real LLM key>
```

The app works without `OPENAI_API_KEY` because it falls back to the mock LLM for this lab.

### Final conclusion

Part 6 is complete. The final project now passes the production readiness checker, builds successfully as a Docker image, runs through Docker Compose, exposes working health/readiness endpoints, enforces API key authentication, applies rate limiting, exposes metrics/cost guard information, and is ready to deploy through Railway or Render config.

### Part 6 Railway deployment verification

I deployed the final project in `06-lab-complete` to a new Railway project named `day12-part6-production`.

Public URL:

```text
https://day12-part6-production-production.up.railway.app
```

Railway deployment status:

```text
Service: day12-part6-production
Status: SUCCESS
```

A runtime issue appeared during the first Railway deploy: `railway.toml` used `--port $PORT`, and Railway passed `$PORT` as a literal string to uvicorn. I fixed this by changing the Railway start command to:

```toml
startCommand = "python -m app.main"
```

This lets `app.config` read the real `PORT` environment variable and start uvicorn with the correct integer port.

Public health check test:

```text
GET https://day12-part6-production-production.up.railway.app/health
HTTP 200
{"status":"ok","version":"1.0.0","environment":"production","uptime_seconds":94.2,"total_requests":4,"checks":{"llm":"mock"},"timestamp":"2026-06-12T11:12:41.718505+00:00"}
```

Public readiness test:

```text
GET https://day12-part6-production-production.up.railway.app/ready
HTTP 200
{"ready":true}
```

Public authenticated `/ask` test:

```text
POST https://day12-part6-production-production.up.railway.app/ask
Header: X-API-Key: part6-demo-key
HTTP 200
{"question":"What is deployment?","answer":"Deployment là quá trình đưa code từ máy bạn lên server để người khác dùng được.","model":"gpt-4o-mini","timestamp":"2026-06-12T11:12:41.839404+00:00"}
```

Public authentication failure test:

```text
POST https://day12-part6-production-production.up.railway.app/ask
Without X-API-Key
HTTP 401
{"detail":"Invalid or missing API key. Include header: X-API-Key: <key>"}
```

Conclusion: the Part 6 final project is deployed publicly on Railway and the deployment requirement is complete.
## Discussion Questions

### Part 1 - Localhost vs Production

#### 1. What happens if you push code with a hardcoded API key to a public GitHub repository?

If an API key is hardcoded and pushed to a public repository, anyone can copy and use it. This can lead to unauthorized API usage, unexpected billing, data exposure, abuse of the service, and account suspension.

The correct response is to revoke the leaked key immediately, create a new key, move secrets into environment variables or a secret manager, and check git history because deleting the key from the latest commit is not enough if it still exists in previous commits.

#### 2. Why is stateless important when scaling?

Stateless design is important because multiple instances can handle requests independently. If state is stored only in one instance's memory, a later request routed to another instance may lose session history or user context.

With stateless app instances, shared state is moved to Redis, a database, or another external store. This allows horizontal scaling, load balancing, rolling deploys, and container restarts without losing user data.

#### 3. What does 12-factor "dev/prod parity" mean in practice?

Dev/prod parity means the development, staging, and production environments should be as similar as possible. The same app should use similar dependencies, configuration style, backing services, and startup commands across environments.

In practice, this means using environment variables for config, Docker for consistent runtime, the same FastAPI app structure locally and in production, and similar health checks/logging behavior. This reduces bugs that only appear after deployment.

### Part 2 - Docker

#### 1. Why copy `requirements.txt` and run `pip install` before `COPY . .`?

Docker builds images layer by layer and caches unchanged layers. `requirements.txt` changes less often than application code, so copying it first lets Docker cache the dependency installation layer.

If the Dockerfile copies all source code before installing dependencies, every code change invalidates the cache and forces `pip install` to run again. That makes builds slower.

#### 2. What should `.dockerignore` contain, and why are `venv/` and `.env` important?

`.dockerignore` should exclude files that are not needed inside the image, such as:

- `.git/`
- `.env` and `.env.*`
- `venv/` and `.venv/`
- `__pycache__/`
- `*.pyc`
- local logs, test caches, and editor files

`venv/` is important because virtual environments are large and platform-specific; dependencies should be installed inside the image instead. `.env` is important because it may contain secrets such as API keys, JWT secrets, database URLs, or cloud credentials.

#### 3. If the agent needs to read files from disk, how do you mount a volume into the container?

Use a Docker bind mount or named volume.

Example with `docker run`:

```powershell
docker run -p 8000:8000 -v D:\AI20k\data:/app/data my-agent:latest
```

Example with Docker Compose:

```yaml
services:
  agent:
    volumes:
      - ./data:/app/data
```

The app can then read files from `/app/data` inside the container while the real files live on the host machine.

### Part 3 - Cloud Deployment

#### 1. Why is serverless, such as Lambda, not always good for an AI agent?

Serverless is useful for short stateless tasks, but AI agents often need longer request time, streaming responses, warm model clients, background work, vector database connections, and predictable latency.

Serverless platforms may have cold starts, execution time limits, memory limits, connection limits, and less control over long-running processes. For many AI agents, container platforms such as Railway, Render, or Cloud Run are easier and more reliable.

#### 2. What is a cold start, and how does it affect UX?

A cold start happens when the platform starts a new instance from zero before serving a request. The app must boot, load dependencies, initialize clients, and become ready before it can respond.

For users, this means the first request can be slow or may time out. In AI apps, cold starts feel especially bad because the LLM call already adds latency. Keeping minimum instances warm or using health checks and readiness probes can reduce the impact.

#### 3. When should you upgrade from Railway to Cloud Run?

Railway is good for quick deployment, demos, small products, and learning because it is simple and fast. Cloud Run is a better fit when the project needs more production control.

Upgrade to Cloud Run when you need stronger IAM/security integration, VPC networking, detailed autoscaling control, regional deployment strategy, CI/CD through Google Cloud Build or GitHub Actions, better observability, or tighter integration with other GCP services.

### Part 4 - API Gateway & Security

#### 1. When should you use API Key vs JWT vs OAuth2?

API keys are best for simple server-to-server access, internal tools, demos, and cases where one key represents one client or service.

JWT is better when users log in and the API needs to know identity, role, expiration time, and claims without checking a database on every request.

OAuth2 is best when users authorize third-party applications or when the system needs standard login/authorization flows with providers such as Google, GitHub, Microsoft, or an enterprise identity provider.

#### 2. How many requests per minute should an AI agent allow?

There is no single correct number. The limit depends on model cost, latency, user plan, infrastructure capacity, and abuse risk.

A reasonable demo limit is 10 to 20 requests per minute per user. A production app can use tiered limits, for example:

- Free users: 5-20 requests/minute
- Paid users: 60-300 requests/minute
- Admin/internal services: higher limits

The limit should be paired with cost guard because AI requests can be expensive even if request count is not high.

#### 3. If an API key is leaked, how do you detect and handle it?

Detection methods include monitoring unusual traffic spikes, high error rates, unexpected geographic access, abnormal cost increase, and logs showing unknown clients using the key.

Response steps:

1. Revoke or rotate the leaked key immediately.
2. Create a new key and update trusted clients.
3. Check logs to estimate impact and abuse window.
4. Reduce blast radius with per-key rate limits and budgets.
5. Remove the secret from code, `.env` files, screenshots, and git history if needed.
6. Add alerting for future spikes in traffic or cost.

Conclusion: the discussion questions for Parts 1-4 are complete.