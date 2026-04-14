# Panduan Deploy DemoApp — Docker / Cloud Server

## Spesifikasi Baseline

| Parameter | Nilai |
|---|---|
| OS | Ubuntu 22.04 LTS (atau 20.04) |
| Server Role | Web server (single instance) |
| Container Runtime | Docker Engine 24+ & Docker Compose v2 |
| Port | 8080 (host) → 80 (container/nginx) |
| Minimum RAM | 512 MB |
| Minimum CPU | 1 vCPU |
| Storage | ~1 GB (OS + Docker image) |
| Jaringan | Akses publik ke port 8080 (atau 80/443 via reverse proxy) |

---

## Struktur File Aplikasi

```
demoapp/
├── index.html          ← Aplikasi web (login + dashboard)
├── Dockerfile          ← Build image Docker
├── nginx.conf          ← Konfigurasi web server Nginx
└── docker-compose.yml  ← Orkestrasi container
```

---

## Langkah-Langkah Deploy

### STEP 1 — Siapkan Server

Login ke server Ubuntu via SSH:

```bash
ssh user@IP_SERVER_ANDA
```

Update sistem:

```bash
sudo apt update && sudo apt upgrade -y
```

---

### STEP 2 — Install Docker Engine

```bash
# Install dependencies
sudo apt install -y ca-certificates curl gnupg lsb-release

# Tambah GPG key Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | \
  sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# Tambah repository Docker
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] \
  https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# Install Docker Engine + Compose plugin
sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Tambah user ke grup docker (agar tidak perlu sudo)
sudo usermod -aG docker $USER
newgrp docker

# Verifikasi
docker --version
docker compose version
```

---

### STEP 3 — Upload File Aplikasi ke Server

Dari komputer lokal kamu, jalankan:

```bash
# Buat folder di server
ssh user@IP_SERVER "mkdir -p ~/demoapp"

# Upload semua file
scp index.html Dockerfile nginx.conf docker-compose.yml \
  user@IP_SERVER:~/demoapp/
```

Atau clone dari Git jika disimpan di repository:

```bash
git clone https://github.com/REPO_KAMU/demoapp.git
cd demoapp
```

---

### STEP 4 — Build Docker Image

Masuk ke folder aplikasi di server:

```bash
cd ~/demoapp
```

Build image:

```bash
docker build -t demoapp:latest .
```

Verifikasi image berhasil dibuat:

```bash
docker images | grep demoapp
```

---

### STEP 5 — Jalankan Container

```bash
docker compose up -d
```

Flag `-d` = detached mode (berjalan di background.

Cek status container:

```bash
docker compose ps
```

Cek logs:

```bash
docker compose logs -f
```

---

### STEP 6 — Buka Firewall (Port 8080)

Untuk Ubuntu dengan UFW:

```bash
sudo ufw allow 8080/tcp
sudo ufw status
```

Untuk cloud provider (AWS, GCP, Azure, dsb): buka port 8080 di Security Group / Firewall Rules masing-masing provider.

---

### STEP 7 — Akses Aplikasi

Buka browser dan akses:

```
http://IP_SERVER_ANDA:8080
```

Login dengan akun test:
- **Username:** `admin`
- **Password:** `demo1234`

---

### STEP 8 — Verifikasi Health Check

```bash
# Cek health status container
docker inspect --format='{{.State.Health.Status}}' demoapp

# Atau test manual via curl
curl -I http://localhost:8080
```

Respon yang diharapkan: `HTTP/1.1 200 OK`

---

## Perintah Operasional Penting

| Aksi | Perintah |
|---|---|
| Start container | `docker compose up -d` |
| Stop container | `docker compose down` |
| Restart | `docker compose restart` |
| Lihat logs | `docker compose logs -f` |
| Lihat status | `docker compose ps` |
| Rebuild image | `docker compose up -d --build` |
| Hapus semua | `docker compose down --rmi all` |

---

## Arsitektur Deploy

```
Internet (Browser)
        │
        ▼ port 8080
  ┌─────────────────────┐
  │   Ubuntu 22.04 LTS  │
  │  ┌───────────────┐  │
  │  │  Docker       │  │
  │  │  Container    │  │
  │  │  ┌─────────┐  │  │
  │  │  │  Nginx  │  │  │
  │  │  │  :80    │  │  │
  │  │  └─────────┘  │  │
  │  └───────────────┘  │
  └─────────────────────┘
```

---

## Catatan Penting

- Ini adalah **demo deployment** — tidak menggunakan HTTPS (SSL), database, atau autentikasi produksi.
- Login dan user data bersifat **hardcoded** di `index.html` (hanya untuk keperluan demo).
- Tidak ada layanan tambahan berbayar yang diperlukan — cukup satu server dengan Docker.
- Untuk akses HTTPS (opsional, tanpa biaya tambahan), dapat ditambahkan Nginx reverse proxy + Certbot/Let's Encrypt di host.

---

## Troubleshooting

**Container tidak mau start:**
```bash
docker compose logs demoapp
```

**Port sudah digunakan:**
```bash
sudo lsof -i :8080
# Ganti port di docker-compose.yml jika bentrok
```

**Permission denied saat menjalankan docker:**
```bash
sudo usermod -aG docker $USER && newgrp docker
```
