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

