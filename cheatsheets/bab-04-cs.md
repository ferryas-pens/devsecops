# Cheatsheet Bab 4 — Nginx, Apache, Flask, dan TLS dengan Docker Compose

Cheatsheet ini menggunakan format praktis seperti Bab 3: satu direktori kerja, daftar file, konfigurasi yang harus ditulis, perintah validasi, pengujian, dan cleanup.

## 1. Direktori Kerja

```bash
mkdir -p ~/docker-lab/bab-4/{apache/sites,nginx/conf,certs,logs/nginx,app}
cd ~/docker-lab/bab-4
```

Struktur file:

```text
bab-4/
├── compose.yaml
├── nginx/conf/default.conf
├── apache/sites/index.html
├── app/Dockerfile
├── app/requirements.txt
├── app/app.py
├── certs/lab.crt
├── certs/lab.key
└── logs/nginx/
```

## 2. Sertifikat TLS Laboratorium

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout certs/lab.key \
  -out certs/lab.crt \
  -subj "/CN=localhost/O=DevSecOps Docker Lab" \
  -addext "subjectAltName=DNS:localhost,IP:127.0.0.1"

chmod 600 certs/lab.key
chmod 644 certs/lab.crt
```

## 3. File `compose.yaml`

```bash
nano compose.yaml
```

```yaml
services:
  proxy:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8080:80"
      - "127.0.0.1:8443:443"
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - ./logs/nginx:/var/log/nginx
    networks:
      - web-net
    depends_on:
      apache-web:
        condition: service_started
      flask-app:
        condition: service_healthy

  apache-web:
    image: httpd:2.4-alpine
    volumes:
      - ./apache/sites:/usr/local/apache2/htdocs:ro
    networks:
      - web-net

  flask-app:
    build:
      context: ./app
    networks:
      - web-net
    healthcheck:
      test:
        - CMD
        - python
        - -c
        - "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health')"
      interval: 5s
      timeout: 3s
      retries: 5
      start_period: 5s

networks:
  web-net:
    driver: bridge
```

## 4. File `nginx/conf/default.conf`

```bash
nano nginx/conf/default.conf
```

```nginx
upstream apache_backend {
    server apache-web:80;
}

upstream flask_backend {
    server flask-app:5000;
}

server {
    listen 80;
    server_name localhost;
    return 301 https://$host:8443$request_uri;
}

server {
    listen 443 ssl;
    server_name localhost;

    ssl_certificate     /etc/nginx/certs/lab.crt;
    ssl_certificate_key /etc/nginx/certs/lab.key;
    ssl_protocols TLSv1.2 TLSv1.3;

    add_header X-Content-Type-Options "nosniff" always;
    add_header X-Frame-Options "DENY" always;
    add_header Referrer-Policy "no-referrer" always;

    location / {
        proxy_pass http://apache_backend;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /api/ {
        proxy_pass http://flask_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

## 5. File `apache/sites/index.html`

```bash
nano apache/sites/index.html
```

```html
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>DevSecOps Bab 4</title>
</head>
<body>
    <h1>Apache di Belakang Nginx</h1>
    <p>Halaman Bab 4 berhasil dilayani melalui reverse proxy.</p>
    <p><a href="/api/">Uji Flask API</a></p>
</body>
</html>
```

## 6. File `app/requirements.txt`

```bash
nano app/requirements.txt
```

```text
Flask==3.1.2
gunicorn==23.0.0
```

## 7. File `app/app.py`

```bash
nano app/app.py
```

```python
from flask import Flask, jsonify, request

app = Flask(__name__)


@app.get("/")
def index():
    return jsonify(
        status="ok",
        service="flask-app",
        message="API Bab 4 berhasil diakses melalui Nginx",
        forwarded_proto=request.headers.get("X-Forwarded-Proto"),
    )


@app.get("/health")
def health():
    return jsonify(status="healthy"), 200
```

## 8. File `app/Dockerfile`

```bash
nano app/Dockerfile
```

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd --system --uid 10001 --no-create-home appuser
USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

## 9. Validasi

```bash
cd ~/docker-lab/bab-4
find . -maxdepth 3 -type f | sort
docker compose config
docker compose config --services
```

Service yang harus muncul:

```text
proxy
apache-web
flask-app
```

## 10. Build dan Menjalankan Stack

```bash
docker compose up -d --build
docker compose ps
docker compose logs --tail 100
```

## 11. Pengujian Cepat

```bash
# HTTP harus redirect ke HTTPS
curl -I http://localhost:8080/

# Halaman Apache melalui Nginx
curl -k -i https://localhost:8443/

# Flask API melalui Nginx
curl -k https://localhost:8443/api/

# Health endpoint Flask
curl -k -i https://localhost:8443/api/health

# Informasi TLS
openssl s_client \
  -connect localhost:8443 \
  -servername localhost \
  -brief </dev/null
```

`curl -k` hanya digunakan untuk sertifikat self-signed laboratorium.

## 12. Pemeriksaan Isolasi

```bash
docker compose ps
docker network inspect bab-4_web-net
docker compose exec proxy wget -qO- http://apache-web/
docker compose exec proxy wget -qO- http://flask-app:5000/health
```

Hanya `proxy` yang boleh memiliki published port.

## 13. Log

```bash
docker compose logs proxy
docker compose logs apache-web
docker compose logs flask-app
tail -n 20 logs/nginx/access.log
tail -n 20 logs/nginx/error.log
```

## 14. Troubleshooting Cepat

| Gejala | Pemeriksaan | Perbaikan utama |
|---|---|---|
| Compose tidak ditemukan | `pwd` dan `ls` | Masuk ke `~/docker-lab/bab-4` |
| `502 Bad Gateway` | `docker compose logs flask-app apache-web` | Periksa upstream, port, dan status backend |
| HTTPS gagal | Log proxy dan file sertifikat | Pastikan `listen 443 ssl` dan mount benar |
| Flask unhealthy | `docker compose logs flask-app` | Periksa dependency dan `/health` |
| Port digunakan | `ss -lntp` | Ganti host port 8080/8443 |
| Log permission denied | `ls -ld logs/nginx` | Koreksi ownership/permission minimum |

## 15. Cleanup

```bash
cd ~/docker-lab/bab-4
docker compose down
```

Hapus image lokal hasil build bila diperlukan:

```bash
docker compose down --rmi local
```

## Checklist PASS

- [ ] `docker compose config` tidak menghasilkan galat.
- [ ] `flask-app` berstatus `healthy`.
- [ ] HTTP port 8080 menghasilkan redirect 301.
- [ ] HTTPS port 8443 menampilkan halaman Apache.
- [ ] `/api/` menghasilkan JSON Flask.
- [ ] TLS 1.2 atau TLS 1.3 berhasil dinegosiasikan.
- [ ] Apache dan Flask tidak memiliki published port.
- [ ] Request tercatat pada access log Nginx.
