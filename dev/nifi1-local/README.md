# Local NiFi 1.x dev environment

A single-command local stand-up of **Apache NiFi 1.28.1** and **NiFi Registry 1.28.1** for building and testing the migration tool. Runs entirely on your laptop, no cloud, no cost.

**Scope:** dev only. Single-user auth, self-signed TLS, non-secure Registry. Do not expose beyond `localhost`.

---

## Prerequisites

- Docker Desktop (macOS/Windows) or Docker Engine + `docker compose` v2 (Linux).
- ~3 GB free RAM for the containers. ~2 GB disk for the initial image pulls.

Verify:

```bash
docker --version
docker compose version
```

---

## Start

From this directory:

```bash
docker compose up -d
```

First run pulls the images (~2 GB) and NiFi itself takes about 2 minutes to finish starting. Watch progress:

```bash
docker compose ps        # both services should reach status "healthy"
docker compose logs -f nifi
```

The NiFi container is ready when `docker compose ps` shows it as `healthy` — that's the point at which the REST API responds.

---

## Access

| Service | URL | Credentials |
|---|---|---|
| NiFi UI | https://localhost:8443/nifi | `admin` / `changeme123456` |
| NiFi REST API | https://localhost:8443/nifi-api | Same, via `curl -k` |
| Registry UI | http://localhost:18080/nifi-registry | None (non-secure, dev only) |
| Registry REST API | http://localhost:18080/nifi-registry-api | None |

Your browser will warn about the self-signed cert — click through.

---

## Smoke-test the REST API

Get an auth token (used as a `Bearer` header for subsequent calls):

```bash
TOKEN=$(curl -k -s -X POST https://localhost:8443/nifi-api/access/token \
  -H 'Content-Type: application/x-www-form-urlencoded' \
  --data-urlencode 'username=admin' \
  --data-urlencode 'password=changeme123456')
echo "$TOKEN" | head -c 40; echo …
```

Fetch the root process group (the entry point the migration tool will use):

```bash
curl -k -s -H "Authorization: Bearer $TOKEN" \
  https://localhost:8443/nifi-api/flow/process-groups/root | jq '.processGroupFlow.id'
```

System diagnostics (JVM heap, disk, threads):

```bash
curl -k -s -H "Authorization: Bearer $TOKEN" \
  https://localhost:8443/nifi-api/system-diagnostics | jq '.systemDiagnostics.aggregateSnapshot.uptime'
```

Registry buckets (no auth needed in dev mode):

```bash
curl -s http://localhost:18080/nifi-registry-api/buckets | jq
```

---

## Add a demo flow (optional, ~5 min)

Two options:

1. **UI**: open https://localhost:8443/nifi, drag a `GenerateFlowFile` processor and a `LogAttribute` processor onto the canvas, connect them, start the flow. You now have a running flow producing metrics against which to test the ingest layer.
2. **Later**: we can add a small seed script (`seed_demo_flows.py`) that provisions a set of representative flows via the REST API. Ask for it when you're ready to test the ingest layer end-to-end.

---

## Stop and clean up

```bash
docker compose down                     # stop containers, keep data volumes
docker compose down --volumes           # stop AND wipe all NiFi state (fresh start)
```

---

## Bumping the NiFi version

The Apache NiFi 1.x line is in maintenance while 2.x is mainline. Check Docker Hub tags at https://hub.docker.com/r/apache/nifi/tags for the latest 1.x patch, then edit both image tags in `docker-compose.yml` (NiFi and Registry should match).

---

## Common issues

| Symptom | Fix |
|---|---|
| `nifi` stuck on `starting`, health check flapping | Give it 2–3 min on first start. If still stuck, `docker compose logs nifi` — look for OOM or port binding errors. |
| Browser gets `ERR_CERT_AUTHORITY_INVALID` | Expected — self-signed. Click through. |
| `curl` returns `SSL certificate problem` | Pass `-k` to skip TLS verification (dev only). |
| Login fails with "Invalid username / password" | Confirm the password matches `SINGLE_USER_CREDENTIALS_PASSWORD` in `docker-compose.yml`. If you changed it after the first start, wipe volumes with `docker compose down --volumes` and restart — the credential is baked in at first-boot. |
| Registry UI shows blank/empty | Non-secure dev mode; try http://localhost:18080/nifi-registry/#/explorer. |
| Docker Desktop RAM pressure | Bump Docker Desktop to ≥ 6 GB RAM, or drop `NIFI_JVM_HEAP_MAX` to `1g`. |
