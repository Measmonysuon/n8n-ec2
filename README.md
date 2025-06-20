
# n8n on EC2 with PostgreSQL and Cloudflare Tunnel

This project documents the deployment of [n8n](https://n8n.io/) — a fair-code automation tool — on an **AWS EC2 Ubuntu instance** using **Docker Compose**, **PostgreSQL**, and secured through **Cloudflare Tunnel**.

---

## ✅ Stack Overview

- **EC2 (Ubuntu)**: Base server for hosting containers
- **Docker Compose**: To run n8n and PostgreSQL containers
- **PostgreSQL**: External DB for n8n data persistence
- **Cloudflare Tunnel**: Secure public access without opening ports
- **Basic Auth**: Secures n8n UI access

---

## 📦 Setup Steps

### 1. Prepare Ubuntu EC2 Instance
```bash
sudo apt update && sudo apt upgrade -y
sudo apt install -y curl gnupg2 unzip docker.io docker-compose
sudo usermod -aG docker $USER && exit
# Reconnect to apply Docker group permissions
```

---

### 2. Create n8n Environment

```bash
mkdir ~/n8n && cd ~/n8n
```

#### Create `.env`
```env
N8N_BASIC_AUTH_ACTIVE=true
N8N_BASIC_AUTH_USER=admin
N8N_BASIC_AUTH_PASSWORD=your_secure_password

DB_TYPE=postgresdb
DB_POSTGRESDB_HOST=postgres
DB_POSTGRESDB_PORT=5432
DB_POSTGRESDB_DATABASE=n8n
DB_POSTGRESDB_USER=n8n_user
DB_POSTGRESDB_PASSWORD=postgres_password

TZ=Asia/Phnom_Penh
```

#### Create `docker-compose.yml`
```yaml
version: '3.8'
services:
  postgres:
    image: postgres:14
    environment:
      POSTGRES_USER: ${DB_POSTGRESDB_USER}
      POSTGRES_PASSWORD: ${DB_POSTGRESDB_PASSWORD}
      POSTGRES_DB: ${DB_POSTGRESDB_DATABASE}
    volumes:
      - pgdata:/var/lib/postgresql/data
    restart: always

  n8n:
    image: docker.n8n.io/n8nio/n8n
    ports:
      - "5678:5678"
    environment:
      - DB_TYPE=${DB_TYPE}
      - DB_POSTGRESDB_HOST=${DB_POSTGRESDB_HOST}
      - DB_POSTGRESDB_PORT=${DB_POSTGRESDB_PORT}
      - DB_POSTGRESDB_DATABASE=${DB_POSTGRESDB_DATABASE}
      - DB_POSTGRESDB_USER=${DB_POSTGRESDB_USER}
      - DB_POSTGRESDB_PASSWORD=${DB_POSTGRESDB_PASSWORD}
      - N8N_BASIC_AUTH_ACTIVE=${N8N_BASIC_AUTH_ACTIVE}
      - N8N_BASIC_AUTH_USER=${N8N_BASIC_AUTH_USER}
      - N8N_BASIC_AUTH_PASSWORD=${N8N_BASIC_AUTH_PASSWORD}
      - TZ=${TZ}
    volumes:
      - n8n_data:/home/node/.n8n
    depends_on:
      - postgres
    restart: always

volumes:
  pgdata:
  n8n_data:
```

#### Start Services
```bash
docker compose up -d
```

---

### 3. Configure Cloudflare Tunnel

#### Install `cloudflared`
```bash
wget -O cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
```

#### Authenticate & Create Tunnel
```bash
cloudflared tunnel login
cloudflared tunnel create n8n-tunnel
```

#### Add Configuration
```bash
mkdir -p ~/.cloudflared
nano ~/.cloudflared/config.yml
```

```yaml
tunnel: n8n-tunnel
credentials-file: /home/ubuntu/.cloudflared/n8n-tunnel.json

ingress:
  - hostname: n8n.yourdomain.com
    service: http://localhost:5678
  - service: http_status:404
```

#### Route DNS and Start Service
```bash
cloudflared tunnel route dns n8n-tunnel n8n.yourdomain.com
sudo mkdir -p /etc/cloudflared
sudo cp ~/.cloudflared/* /etc/cloudflared/
sudo cloudflared service install
sudo systemctl start cloudflared
```

---

## ✅ Access Your Instance

Visit:  
`https://n8n.yourdomain.com`  
Login: `admin / your_secure_password`

---

## 📂 Notes

- PostgreSQL is used instead of default SQLite for scalability.
- Cloudflare Tunnel ensures secure public access without opening ports in the firewall.
- Basic auth is enabled for admin protection.

---

## 📜 License

This setup uses n8n under the [Fair-code license](https://faircode.io/). Please respect n8n's licensing if used commercially.
