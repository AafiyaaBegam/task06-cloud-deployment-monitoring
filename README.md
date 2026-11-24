# Task 06 — Cloud Infrastructure Deployment & Monitoring Capstone

Containerized cloud deployment and availability monitoring for the Task 03 application.

## Architecture

- FastAPI application container from the Task 03 Docker Hub image.
- Docker Compose orchestration and persistent Prometheus/Grafana volumes.
- Prometheus blackbox exporter probes the application's `/health` endpoint.
- Prometheus alert rules detect failed and slow health checks.
- Alertmanager routes alerts to a configurable HTTPS webhook.
- Grafana is provisioned with Prometheus and a Cloud Capstone dashboard.
- App and Grafana host ports bind to localhost; monitoring services remain private to the Docker network.

## Repository contents

- `docker-compose.yml` — application, Blackbox Exporter, Prometheus, Alertmanager and Grafana.
- `prometheus.yml`, `blackbox.yml`, `prometheus/alerts.yml` — scraping, probe and alert rules.
- `alertmanager.yml` — webhook notification route.
- `grafana/provisioning/` — preconfigured datasource and dashboard.
- `.env.example` — environment variable template (copy to `.env`; never commit secrets).
- `docs/aws-ec2-deployment.md` — AWS EC2 deployment, HTTPS, monitoring and evidence runbook.

## Local validation

Requires Docker Engine and Docker Compose:

```bash
cp .env.example .env
# Edit .env and set a strong Grafana password plus a real alert webhook.
docker compose config
docker compose pull
docker compose up -d
docker compose ps
curl -i http://127.0.0.1:8000/health
```

Grafana is available locally at `http://127.0.0.1:3000`. Prometheus,
Alertmanager and Blackbox Exporter are not published to the host network.

## Cloud deployment status

The repository is cloud-deployment-ready and includes an AWS EC2 runbook.
**No EC2 instance, DNS name, TLS certificate, live public URL, or monitoring
evidence is claimed yet.** Provisioning a cloud server can incur charges and
requires an AWS account plus explicit launch approval. Follow the runbook and
capture genuine evidence before marking the live deployment complete.

## Security

Do not expose Docker daemon, application port 8000, Grafana port 3000,
Prometheus port 9090, Alertmanager port 9093, or Blackbox port 9115 publicly.
Use a host reverse proxy with HTTPS, strong credentials, least-privilege
security-group rules, and secret values stored outside Git.
