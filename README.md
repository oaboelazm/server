# 🚀 SaaS Server Infrastructure Setup

**Target OS:** Ubuntu Linux

This document serves as the architectural blueprint and step-by-step deployment guide for a production-ready SaaS infrastructure. It implements strict network segmentation, a Zero-Trust security model, and heavily isolated Docker networks to ensure no backend systems are ever exposed directly to the public internet.

---

## 🏢 SaaS Zero-Trust Architecture

The network topology is structurally divided into four strictly isolated layers:

### 1. The Edge Layer (`edge_net`)
- **Purpose:** Public Internet gateway.
- **Services:** `nginx-proxy-manager` ONLY.
- **Rule:** This is the only network that binds to the host's public ports (`80`/`443`).

### 2. The Infrastructure Layer (`infra_net`)
- **Purpose:** Proxy configuration and state isolation.
- **Services:** `nginx-proxy-manager`, `npm_db`.
- **Rule:** A private, air-gapped network so the proxy can communicate with its internal MariaDB configuration storage without exposing it to the main application layers.

### 3. The Application Layer (`app_net`)
- **Purpose:** Frontend routing and SaaS backend execution.
- **Services:** `nginx-proxy-manager`, `landing`, `app`, `api`.
- **Rule:** Proxies public traffic exclusively to backend containers. Frontend applications here (`landing`, `app`) **cannot** reach the database layer natively. 

### 4. The Data Layer (`data_net`)
- **Purpose:** Business data isolation and secure administration.
- **Services:** `api`, `saas_db`, `phpmyadmin`.
- **Rule:** This vault houses the main `saas_db` (MySQL). It is heavily restricted: **only** the `api` service can access it to fetch/write business data. `phpmyadmin` manages it internally via secure localhost tunneling.

---

## 🔒 Security Posture & VPN (Tailscale)
All admin dashboards (NPM Admin and phpMyAdmin) are bound **exclusively** to localhost (`127.0.0.1`), meaning they are completely inaccessible from outside the server. 
**Access requires internal routing via Tailscale Subnet Routing or SSH tunneling.**

---

## 🚀 Deployment Instructions

### Step 1: System Update & Essential Utilities
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl wget git ufw nano jq openssl
```

### Step 2: Configure UFW Firewall
Restrict all incoming traffic except SSH, HTTP, HTTPS, and your VPN layer.
```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow in on tailscale0
sudo ufw enable
```

### Step 3: Install & Configure Tailscale (VPN)
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up

# Extract the Tailscale IP and save it to your bash profile
export TAILSCALE_IP=$(tailscale ip -4)
echo "export TAILSCALE_IP=$TAILSCALE_IP" >> ~/.bashrc
source ~/.bashrc
```

### Step 4: Install Docker & Docker Compose
```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo systemctl enable docker
sudo systemctl start docker

# Add your user to the docker group
sudo usermod -aG docker $USER
```

### Step 5: Setup Project & Environment Variables
Generate secure, randomized passwords for the deployment.
```bash
mkdir -p ~/saas-infra/data
cd ~/saas-infra

# Copy the example config
cp .env.example .env

# Open .env and populate the variables with secure strings
nano .env
chmod 600 .env
```

### Step 6: Create External Docker Networks
Because we declared `edge_net` and `infra_net` as `external: true`, they must be created before running Docker Compose.
```bash
docker network create edge_net
docker network create infra_net
```

### Step 7: Deploy the Stack
Deploy the infrastructure in the background. The native `/bin/check-health` and socket scripts will block downstream containers until their dependencies are strictly ready.
```bash
docker compose up -d
docker compose ps
```

---

## ⚙️ Post-Deployment Configuration & Domain Routing

The architecture is structurally primed to support horizontal SaaS scaling via Nginx Proxy Manager's UI, utilizing domain-based routing.

When future microservice containers (like `frontend` or `backend`) are added to `app_net`, you will manage them exclusively through the NPM web interface **without hardcoding ports** into the `docker-compose.yml`:

- **Landing Page:** `landing.yourdomain.com` → Routes to `landing:80`
- **Frontend App:** `app.yourdomain.com` → Routes to `app:80`
- **Backend API:** `api.yourdomain.com` → Routes to `api:80`

### Workflow for Expanding Your SaaS:
1. Access the NPM Admin UI via your browser using your secure SSH tunnel or Tailscale IP: `http://127.0.0.1:81`.
2. **Default Login:** `admin@example.com` / `changeme` (Change this immediately!).
3. Deploy the new application container (e.g., `app`).
4. **Do not** map its ports to the host machine. Connect it exclusively to `app_net`.
5. In Nginx Proxy Manager, click "Proxy Hosts" -> "Add Proxy Host".
6. Enter the domain (`app.yourdomain.com`) and point it to the internal Docker hostname (`app`) and internal port.
7. Go to the **SSL** tab, request a new certificate via Let's Encrypt, and enable "Force SSL".
8. Save.

This guarantees a seamless, fully encrypted, highly scalable, zero-downtime routing infrastructure!
