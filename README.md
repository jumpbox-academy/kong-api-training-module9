# 🚀 API Traffic Management and Load Balancing with Kong

This section will:
- Create an **Upstream + Targets** (round-robin)
- Create a **Service + Route** (`/whoami`)
- Verify baseline load balancing (hostname alternates between `whoami-a` / `whoami-b`)
- Stop one target and confirm traffic still flows (with **active health checks**)
- Apply **rate-limiting** and **request-termination** plugins on the route

## 📁 Prerequisites

Make sure you have installed:

- Docker
- Docker Compose
- Git

---

You can verify your Docker installation with:
```bash
docker --version

docker compose version
```

## ✅ Getting Started
### 1.Create a Docker Network

Create a dedicated network for Kong:
```bash
docker network create kong-net
```

To check existing Docker networks, run:
```bash
docker network ls
```

You should see kong-net listed.

---

### 2.Start Kong Gateway

Start all services using Docker Compose:
```bash
docker compose up -d
```

**Create Upstream + Targets**
Create an Upstream:
```bash
# Create Upstream
curl -sS -i -X POST "http://localhost:8001/upstreams" \
  --data "name=whoami_upstream" \
  --data "algorithm=round-robin"
```
Add Targets:
```bash
# Add Targets
curl -sS -i -X POST "http://localhost:8001/upstreams/whoami_upstream/targets" \
  --data "target=whoami-a:80" --data "weight=100"

curl -sS -i -X POST "http://localhost:8001/upstreams/whoami_upstream/targets" \
  --data "target=whoami-b:80" --data "weight=100"
```

Confirm targets:
```bash
curl -sS "http://localhost:8001/upstreams/whoami_upstream/targets" | jq -r '.data[].target'
```

**Create Service + Route (/whoami) and test via proxy :8000**

Create Service that points to the upstream by host:
```bash
curl -sS -i -X POST "http://localhost:8001/services" \
  --data "name=whoami" \
  --data "protocol=http" \
  --data "host=whoami_upstream" \
  --data "port=80"
```

Create Route:
```bash
curl -sS -i -X POST "http://localhost:8001/services/whoami/routes" \
  --data "name=whoami-route" \
  --data "paths[]=/whoami" \
  --data "strip_path=true"
```

Test:
```bash
curl -sS "http://localhost:8000/whoami" | head -n 12
```

**Baseline distribution (hostname should alternate a/b)**
```bash
for i in {1..10}; do
  curl -sS "http://localhost:8000/whoami" | grep -i Hostname
done
```

**Stop one target and confirm traffic continues (with active health checks)**

Important: If active health checks are not enabled, Kong may still try to send traffic to a dead target for a while.
To see results quickly, enable fast active checks before stopping a target.

Enable fast active checks:
```bash
curl -sS -i -X PATCH "http://localhost:8001/upstreams/whoami_upstream" \
  -H "Content-Type: application/json" \
  -d '{
    "healthchecks": {
      "active": {
        "type": "http",
        "http_path": "/",
        "timeout": 1,
        "concurrency": 5,
        "healthy":   { "interval": 5 },
        "unhealthy": { "interval": 5 }
      }
    }
  }'
```

Stop one target container:
```bash
docker stop whoami-b
sleep 6
```

Re-test (should mostly return whoami-a only):
```bash
for i in {1..10}; do
  curl -sS "http://localhost:8000/whoami" | grep -i Hostname
done
```

Bring it back:
```bash
docker start whoami-b
sleep 2
```

**Rate limiting (5 req/min) on the route**

Enable rate limiting:
```bash
curl -sS -i -X POST "http://localhost:8001/routes/whoami-route/plugins" \
  --data "name=rate-limiting" \
  --data "config.minute=5" \
  --data "config.policy=local"
```

Test (you should see 429 after exceeding the limit):
```bash
for i in {1..8}; do
  curl -sS -o /dev/null -w "%{http_code}\n" "http://localhost:8000/whoami"
done
```

**Request termination (503 Maintenance mode)**

Enable maintenance response:
```bash
curl -sS -i -X POST "http://localhost:8001/routes/whoami-route/plugins" \
  --data "name=request-termination" \
  --data "config.status_code=503" \
  --data "config.message=Maintenance"
```

Test:
```bash
curl -sS -i "http://localhost:8000/whoami" | head -n 10
```

---
## 🧹 3.Cleanup (Stop and Remove Containers)
When you are done, you can stop and remove all running containers:
```bash
docker compose down
```

If you also want to remove the custom network:
```bash
docker network rm kong-net
```