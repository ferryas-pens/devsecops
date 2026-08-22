File Compose pada Bab 3 harus disimpan sebagai `compose.yaml` di direktori utama praktikum. Namun, file Compose saja belum cukup karena service `app` membutuhkan Dockerfile dan source Flask, sedangkan service `web` membutuhkan `nginx.conf`.

## 1. Membuat direktori praktikum

Pada Linux atau Ubuntu WSL di Windows:

```bash
mkdir -p ~/docker-lab/bab-3/app
mkdir -p ~/docker-lab/bab-3/html
cd ~/docker-lab/bab-3
```

Struktur akhirnya:

```text
bab-3/
├── compose.yaml
├── nginx.conf
├── html/
│   └── index.html
└── app/
    ├── Dockerfile
    ├── requirements.txt
    └── app.py
```

## 2. Menulis `compose.yaml`

Buka editor Nano:

```bash
nano compose.yaml
```

Pada Windows dengan Visual Studio Code dan WSL, dapat menggunakan:

```bash
code compose.yaml
```

Tuliskan isi berikut. Gunakan spasi, bukan tab.

```yaml
services:
  web:
    image: nginx:alpine
    ports:
      - "127.0.0.1:8080:80"
    volumes:
      - ./html:/usr/share/nginx/html:ro
      - ./nginx.conf:/etc/nginx/conf.d/default.conf:ro
    networks:
      - frontend
    depends_on:
      - app

  app:
    build:
      context: ./app
    environment:
      DB_HOST: db
      DB_PORT: "5432"
      DB_NAME: labdb
      DB_USER: labuser
      DB_PASS: labpass123
    networks:
      - frontend
      - backend
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: labdb
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: labpass123
    volumes:
      - pg-data:/var/lib/postgresql/data
    networks:
      - backend
    healthcheck:
      test:
        - CMD-SHELL
        - pg_isready -U labuser -d labdb
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  pg-data:

networks:
  frontend:
  backend:
```

Pemetaan port menggunakan:

```yaml
- "127.0.0.1:8080:80"
```

Konfigurasi tersebut membatasi akses Nginx hanya dari laptop lokal. Bentuk pada buku, `"8080:80"`, dapat membuka layanan pada seluruh interface jaringan host.

Jika menggunakan Nano, simpan dengan:

1. `Ctrl+O`
2. Tekan `Enter`
3. `Ctrl+X`

## 3. Membuat aplikasi Flask

Buat `app/requirements.txt`:

```bash
nano app/requirements.txt
```

Isi:

```text
Flask==3.1.2
psycopg[binary]==3.2.9
gunicorn==23.0.0
```

Buat `app/app.py`:

```bash
nano app/app.py
```

Isi:

```python
import os

import psycopg
from flask import Flask, jsonify

app = Flask(__name__)


def database_connection():
    return psycopg.connect(
        host=os.environ["DB_HOST"],
        port=os.getenv("DB_PORT", "5432"),
        dbname=os.environ["DB_NAME"],
        user=os.environ["DB_USER"],
        password=os.environ["DB_PASS"],
    )


@app.get("/")
def index():
    try:
        with database_connection() as connection:
            with connection.cursor() as cursor:
                cursor.execute("SELECT version();")
                database_version = cursor.fetchone()[0]

        return jsonify(
            status="ok",
            message="Nginx, Flask, dan PostgreSQL berhasil terhubung",
            database=database_version,
        )
    except Exception as error:
        return jsonify(
            status="error",
            message="Koneksi database gagal",
            detail=str(error),
        ), 500


@app.get("/health")
def health():
    try:
        with database_connection() as connection:
            with connection.cursor() as cursor:
                cursor.execute("SELECT 1;")
                cursor.fetchone()

        return jsonify(status="healthy"), 200
    except Exception:
        return jsonify(status="unhealthy"), 503
```

## 4. Membuat Dockerfile aplikasi

Buat `app/Dockerfile`:

```bash
nano app/Dockerfile
```

Isi:

```dockerfile
FROM python:3.12-slim

WORKDIR /app

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

COPY app.py .

RUN useradd --system --uid 10001 appuser
USER appuser

EXPOSE 5000

CMD ["gunicorn", "--bind", "0.0.0.0:5000", "--workers", "2", "app:app"]
```

`USER appuser` memastikan aplikasi tidak berjalan sebagai root.

## 5. Membuat konfigurasi Nginx

Buat `nginx.conf`:

```bash
nano nginx.conf
```

Isi:

```nginx
upstream flask_backend {
    server app:5000;
}

server {
    listen 80;
    server_name localhost;

    location / {
        proxy_pass http://flask_backend;
        proxy_http_version 1.1;

        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 5s;
        proxy_read_timeout 30s;
    }

    location = /static.html {
        root /usr/share/nginx/html;
    }
}
```

Nama `app` pada `server app:5000` berasal dari nama service Compose. Docker menyediakan resolusi DNS internal sehingga alamat IP container tidak perlu ditulis manual.

## 6. Membuat halaman statis opsional

```bash
nano html/index.html
```

Isi:

```html
<!DOCTYPE html>
<html lang="id">
<head>
    <meta charset="UTF-8">
    <title>Laboratorium Docker Compose</title>
</head>
<body>
    <h1>Docker Compose Bab 3</h1>
    <p>Bind mount Nginx berhasil digunakan.</p>
</body>
</html>
```

File tersebut dapat diakses melalui:

```text
http://localhost:8080/static.html
```

## 7. Memvalidasi Compose

Sebelum menjalankan container, periksa struktur YAML:

```bash
docker compose config
```

PASS ditandai dengan tampilnya konfigurasi akhir tanpa pesan galat. Jika terdapat kesalahan seperti berikut:

```text
yaml: line 12: mapping values are not allowed
```

biasanya penyebabnya adalah:

* penggunaan tab;
* indentasi tidak konsisten;
* tanda titik dua hilang;
* isi konfigurasi ditempatkan pada level yang salah.

Periksa juga service yang akan dibuat:

```bash
docker compose config --services
```

Keluaran yang diharapkan:

```text
web
app
db
```

## 8. Menjalankan stack

```bash
docker compose up -d --build
```

Periksa status:

```bash
docker compose ps
```

Keluaran konseptualnya:

```text
NAME          SERVICE   STATUS
bab-3-web-1   web       Up
bab-3-app-1   app       Up
bab-3-db-1    db        Up (healthy)
```

Nama container dapat berbeda karena Compose menambahkan nama project.

## 9. Menguji aplikasi

```bash
curl http://localhost:8080/
```

Keluaran yang diharapkan:

```json
{
  "database": "PostgreSQL 16...",
  "message": "Nginx, Flask, dan PostgreSQL berhasil terhubung",
  "status": "ok"
}
```

Uji health endpoint:

```bash
curl -i http://localhost:8080/health
```

Uji halaman statis:

```bash
curl http://localhost:8080/static.html
```

## 10. Memeriksa log

```bash
docker compose logs --tail 100
```

Untuk mengikuti log secara real-time:

```bash
docker compose logs -f
```

Log service tertentu:

```bash
docker compose logs app
docker compose logs db
docker compose logs web
```

## 11. Menghentikan lingkungan

Menghapus container dan network, tetapi mempertahankan data PostgreSQL:

```bash
docker compose down
```

Menghapus container, network, sekaligus volume PostgreSQL:

```bash
docker compose down -v
```

Perintah kedua bersifat destruktif karena data pada `pg-data` akan dihapus.

## Kesalahan yang sering terjadi

| Gejala                              | Penyebab                                      | Penyelesaian                                                         |
| ----------------------------------- | --------------------------------------------- | -------------------------------------------------------------------- |
| `no configuration file provided`    | Perintah dijalankan dari direktori yang salah | Jalankan `cd ~/docker-lab/bab-3`                                     |
| `app: no such file or directory`    | Folder `app` belum dibuat                     | Periksa dengan `find . -maxdepth 2 -type f`                          |
| `port is already allocated`         | Port 8080 digunakan aplikasi lain             | Ubah menjadi `127.0.0.1:8081:80`                                     |
| Database `unhealthy`                | Password, database, atau healthcheck salah    | Periksa `docker compose logs db`                                     |
| Nginx menampilkan `502 Bad Gateway` | Aplikasi Flask belum siap atau gagal          | Periksa `docker compose logs app`                                    |
| Perubahan password tidak berlaku    | Volume lama menyimpan database sebelumnya     | Gunakan password lama atau reset lab dengan `docker compose down -v` |
| YAML gagal diproses                 | Tab atau indentasi salah                      | Jalankan `docker compose config`                                     |

**[High confidence]** Struktur Compose dan file pendukung di atas konsisten dengan arsitektur Bab 3. Risiko utama pada contoh buku adalah file Compose tidak dapat berjalan sendiri tanpa `app/Dockerfile`, source Flask, dan `nginx.conf`; komponen tersebut telah dilengkapi dalam tutorial ini.
