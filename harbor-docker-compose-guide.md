
# Harbor Docker Compose Setup Guide

This document provides step-by-step instructions for running **Harbor** using **Docker Compose**.

---

## Prerequisites

- **Docker** (20.x or later)
- **Docker Compose** (1.29 or later)
- A Linux-based system (Ubuntu, CentOS, etc.)
- Internet access (for online installation)

---

## 1. Download Harbor

```bash
wget https://github.com/goharbor/harbor/releases/download/v2.10.0/harbor-online-installer-v2.10.0.tgz
```

> Check for the latest versions at: [https://github.com/goharbor/harbor/releases](https://github.com/goharbor/harbor/releases)

---

## 2. Extract the Archive

```bash
tar xzvf harbor-online-installer-v2.10.0.tgz
cd harbor
```

---

## 3. Configure `harbor.yml`

```bash
cp harbor.yml.tmpl harbor.yml
```

### Example content for `harbor.yml`:

```yaml
hostname: localhost  # or your IP/domain ex: reg.mydomain.com

http:
  port: 8080

# Enable HTTPS (optional):
# https:
#   port: 443
#   certificate: /your/cert/path
#   private_key: /your/key/path

harbor_admin_password: Harbor12345

data_volume: /data

log:
  level: info
```

> If using HTTPS, provide the paths for your certificate and private key.

---

## 4. Run the Installer

```bash
./install.sh
```

This generates the `docker-compose.yml` file and starts the Harbor services.

---

## 5. Access the Web UI

Open in your browser:

```
http://localhost:8080
```

Login credentials:

- **Username:** `admin`
- **Password:** `Harbor12345` (or whatever you set in `harbor.yml`)

---

## 6. Manage with Docker Compose

```bash
# Start
docker-compose up -d

# Stop
docker-compose down

# Logs
docker-compose logs -f
```

---

## Optional: Enable HTTPS

1. Uncomment the `https:` section in `harbor.yml`.
2. Provide your certificate and key file paths.
3. Run `./install.sh` again.

Alternatively, use a reverse proxy like **Nginx** or **Traefik** for SSL termination.

---

## Optional: Systemd Service

You can create a `systemd` service to auto-start Harbor on boot (optional).

---
```bash
curl localhost
docker pull busybox
docker images
docker tag busybox localhost/busybox
docker login localhost
docker push locahost/busybox

vi Dockerfile
docker build -t localhost/web/sleep:1 .
docker images
docker push localhost/web/busybox
```




## Resources

- [Harbor Official Documentation](https://goharbor.io/docs)
- [Harbor GitHub Repository](https://github.com/goharbor/harbor)

---
