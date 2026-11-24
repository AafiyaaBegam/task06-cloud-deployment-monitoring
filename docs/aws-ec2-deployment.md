# AWS EC2 Deployment Runbook

This is a deployment procedure for the Task 06 stack. It does not create AWS
resources or claim a live deployment. Provisioning an EC2 instance can incur
charges; review the AWS estimate and account limits before launching.

## Architecture

Internet -> EC2 security group (22 from trusted IP, 80/443 public) -> host Nginx
-> application bound to 127.0.0.1:8000. Prometheus, Grafana, Blackbox Exporter,
and Alertmanager communicate on a private Docker network. Grafana is bound to
127.0.0.1:3000 and should be accessed through an SSH tunnel or an authenticated
HTTPS proxy, never by opening port 3000 publicly.

## 1. Create the EC2 host

In AWS Console, launch Ubuntu Server 24.04 LTS on an instance size suitable for
the application plus four monitoring containers. Configure:
- A dedicated SSH key pair.
- Security group inbound: TCP 22 from your current trusted public IP only;
  TCP 80 and 443 from the internet.
- Do not open 8000, 3000, 9090, 9093, or 9115 publicly.
- Attach an appropriate IAM role only if the deployment needs AWS API access.

Record the public IPv4 address and instance ID. Review pricing before launch.

## 2. Connect and install Docker

```bash
ssh -i YOUR_KEY.pem ubuntu@EC2_PUBLIC_IP
sudo apt update
sudo apt install -y ca-certificates curl git nginx
sudo install -m 0755 -d /etc/apt/keyrings
sudo curl -fsSL https://download.docker.com/linux/ubuntu/gpg -o /etc/apt/keyrings/docker.asc
sudo chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/ubuntu $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin
sudo systemctl enable --now docker
sudo docker version
sudo docker compose version
```

## 3. Deploy the stack

```bash
git clone https://github.com/AafiyaaBegam/task06-cloud-deployment-monitoring.git
cd task06-cloud-deployment-monitoring
cp .env.example .env
nano .env
```

Set a unique long Grafana password and a real HTTPS incident webhook URL. Keep
the populated `.env` file private; never commit it.

```bash
sudo docker compose config
sudo docker compose pull
sudo docker compose up -d
sudo docker compose ps
sudo docker compose logs --tail=100
curl -i http://127.0.0.1:8000/health
```

## 4. Configure host Nginx

Create an Nginx site that proxies the public domain to `http://127.0.0.1:8000`.
Use the hardened configuration in the Task 05 repository, changing
`server_name` to your real domain. Validate with `sudo nginx -t` before
reloading. Point the domain's DNS A record to the EC2 public IP.

## 5. Enable HTTPS

```bash
sudo apt install -y certbot python3-certbot-nginx
sudo certbot --nginx -d YOUR_DOMAIN
sudo certbot renew --dry-run
```

Only claim a live HTTPS endpoint after DNS resolves and an external HTTPS
request returns the expected application response.

## 6. Verify monitoring

```bash
sudo docker compose ps
curl -fsS http://127.0.0.1:8000/health
curl -fsS http://127.0.0.1:9090/-/healthy
curl -I http://127.0.0.1:3000
```

Access Grafana safely with an SSH tunnel from your local machine:

```bash
ssh -i YOUR_KEY.pem -L 3000:127.0.0.1:3000 ubuntu@EC2_PUBLIC_IP
```

Open `http://localhost:3000`, sign in using the credentials from `.env`,
and open the provisioned **Cloud Capstone Monitoring** dashboard. In
Prometheus, confirm the `cloud-app-health` probe target is UP and
`probe_success` is 1. Confirm Alertmanager is receiving the configured route.
A notification test requires a valid webhook endpoint.

## 7. Evidence to capture

Save genuine, redacted evidence in `docs/evidence/`:
- EC2 instance state and security-group rules.
- `docker compose ps` and health endpoint response.
- Nginx `nginx -t` output and external HTTPS response.
- Prometheus target page showing the blackbox probe.
- Grafana dashboard showing availability and response time.
- Alert firing/resolution and delivery confirmation from the configured webhook.

Do not commit AWS credentials, SSH private keys, populated `.env`, or webhook
secrets. Delete the EC2 instance and associated billable resources when the
exercise is finished if they are no longer needed.
