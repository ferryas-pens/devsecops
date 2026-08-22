# Cheatsheet Bab 5 — PostgreSQL, pgAdmin, Volume, Backup, dan Restore

Cheatsheet ini menggunakan pola yang sama dengan Bab 3 dan Bab 4. Setiap file yang harus dibuat ditempatkan dalam **Script**. Perintah terminal untuk setup, validasi, pengujian, backup, restore, dan cleanup ditempatkan dalam **Perintah** terpisah.

| Kotak | Nama file | Fungsi |
|---:|---|---|
| 1 | `.env.example` | Template konfigurasi laboratorium |
| 2 | `.gitignore` | Mencegah credential dan dump masuk repository |
| 3 | `compose.yaml` | PostgreSQL, pgAdmin, volume, healthcheck, dan network |
| 4 | `init/01-schema.sql` | Membuat tabel dan data awal |
| 5 | `pgadmin/servers.json` | Mendaftarkan server PostgreSQL pada pgAdmin |
| 6 | `scripts/backup.sh` | Membuat logical backup dan checksum |
| 7 | `scripts/restore-test.sh` | Menguji restore pada database terpisah |

## Kotak Perintah A — Menyiapkan Direktori Bab 5

```bash
mkdir -p ~/docker-lab/bab-5/{init,backup,pgadmin,scripts}
cd ~/docker-lab/bab-5
```

Struktur yang akan dibuat:

```text
bab-5/
├── .env.example
├── .env
├── .gitignore
├── compose.yaml
├── init/
│   └── 01-schema.sql
├── pgadmin/
│   └── servers.json
├── scripts/
│   ├── backup.sh
│   └── restore-test.sh
└── backup/
```

## Script 1 — File `.env.example`

> **Nama file:** `.env.example`  
> **Lokasi:** `~/docker-lab/bab-5/.env.example`  
> **Buka editor:** `nano .env.example`  
> **Petunjuk:** nilai berikut hanya credential sintetis untuk laptop laboratorium.

```dotenv
POSTGRES_DB=labdb
POSTGRES_USER=labuser
POSTGRES_PASSWORD=labpass123
PGADMIN_DEFAULT_EMAIL=admin@example.local
PGADMIN_DEFAULT_PASSWORD=admin123
```

Salin template menjadi konfigurasi aktif:

```bash
cp .env.example .env
chmod 600 .env
```

File `.env` memudahkan interpolasi Compose, tetapi bukan secret manager. Jangan gunakan nilai laboratorium ini pada sistem produksi.

## Script 2 — File `.gitignore`

> **Nama file:** `.gitignore`  
> **Lokasi:** `~/docker-lab/bab-5/.gitignore`  
> **Buka editor:** `nano .gitignore`  
> **Petunjuk:** file ini mengurangi risiko credential dan dump terkirim ke Git.

```gitignore
.env
backup/*.dump
backup/*.sha256
backup/*.log
```

## Script 3 — File `compose.yaml`

> **Nama file:** `compose.yaml`  
> **Lokasi:** `~/docker-lab/bab-5/compose.yaml`  
> **Buka editor:** `nano compose.yaml`  
> **Petunjuk:** gunakan spasi, bukan tab.

```yaml
services:
  postgres-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: ${POSTGRES_DB}
      POSTGRES_USER: ${POSTGRES_USER}
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
    ports:
      - "127.0.0.1:5432:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
    networks:
      - data-net
    healthcheck:
      test:
        - CMD-SHELL
        - "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"
      interval: 5s
      timeout: 5s
      retries: 10
      start_period: 10s
    restart: unless-stopped

  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: ${PGADMIN_DEFAULT_EMAIL}
      PGADMIN_DEFAULT_PASSWORD: ${PGADMIN_DEFAULT_PASSWORD}
    ports:
      - "127.0.0.1:5050:80"
    volumes:
      - pgadmin-data:/var/lib/pgadmin
      - ./pgadmin/servers.json:/pgadmin4/servers.json:ro
    networks:
      - data-net
    depends_on:
      postgres-db:
        condition: service_healthy
    restart: unless-stopped

volumes:
  pg-data:
  pgadmin-data:

networks:
  data-net:
    driver: bridge
```

Port PostgreSQL dan pgAdmin dibatasi ke loopback. Untuk eksperimen yang hanya menggunakan pgAdmin di dalam Compose, published port PostgreSQL dapat dihapus sehingga database sama sekali tidak terbuka ke host.

## Script 4 — File `init/01-schema.sql`

> **Nama file:** `01-schema.sql`  
> **Lokasi:** `~/docker-lab/bab-5/init/01-schema.sql`  
> **Buka editor:** `nano init/01-schema.sql`  
> **Petunjuk:** init script hanya dijalankan ketika direktori data PostgreSQL masih kosong.

```sql
CREATE TABLE IF NOT EXISTS students (
    id BIGSERIAL PRIMARY KEY,
    nrp VARCHAR(20) UNIQUE NOT NULL,
    name TEXT NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO students (nrp, name)
VALUES
    ('31230001', 'Mahasiswa Satu'),
    ('31230002', 'Mahasiswa Dua')
ON CONFLICT (nrp) DO NOTHING;

CREATE INDEX IF NOT EXISTS idx_students_name
    ON students (name);
```

## Script 5 — File `pgadmin/servers.json`

> **Nama file:** `servers.json`  
> **Lokasi:** `~/docker-lab/bab-5/pgadmin/servers.json`  
> **Buka editor:** `nano pgadmin/servers.json`  
> **Petunjuk:** password tidak ditulis pada file ini; pgAdmin akan memintanya ketika koneksi dibuat.

```json
{
  "Servers": {
    "1": {
      "Name": "PostgreSQL Bab 5",
      "Group": "Laboratorium DevSecOps",
      "Host": "postgres-db",
      "Port": 5432,
      "MaintenanceDB": "labdb",
      "Username": "labuser",
      "SSLMode": "prefer"
    }
  }
}
```

Hostname pgAdmin harus menggunakan `postgres-db`, bukan `localhost`, karena pgAdmin berjalan dalam container berbeda. Di dalam container pgAdmin, `localhost` berarti container pgAdmin itu sendiri.

## Script 6 — File `scripts/backup.sh`

> **Nama file:** `backup.sh`  
> **Lokasi:** `~/docker-lab/bab-5/scripts/backup.sh`  
> **Buka editor:** `nano scripts/backup.sh`  
> **Petunjuk:** script menggunakan `docker compose exec`, sehingga tidak bergantung pada nama container dinamis.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

mkdir -p backup

timestamp="$(date -u +%Y%m%dT%H%M%SZ)"
dump_file="backup/${POSTGRES_DB}-${timestamp}.dump"
checksum_file="${dump_file}.sha256"

docker compose exec -T postgres-db \
    pg_dump \
    --username "$POSTGRES_USER" \
    --dbname "$POSTGRES_DB" \
    --format custom \
    --no-owner \
    --no-privileges \
    > "$dump_file"

test -s "$dump_file"
sha256sum "$dump_file" > "$checksum_file"

echo "[PASS] Backup: $dump_file"
echo "[PASS] Checksum: $checksum_file"
```

## Script 7 — File `scripts/restore-test.sh`

> **Nama file:** `restore-test.sh`  
> **Lokasi:** `~/docker-lab/bab-5/scripts/restore-test.sh`  
> **Buka editor:** `nano scripts/restore-test.sh`  
> **Petunjuk:** script tidak menimpa `labdb`; restore dilakukan ke database `labdb_restore_test`.

```bash
#!/usr/bin/env bash
set -Eeuo pipefail

cd "$(dirname "$0")/.."

if [[ ! -f .env ]]; then
    echo "[FAIL] File .env tidak ditemukan." >&2
    exit 1
fi

set -a
source .env
set +a

dump_file="${1:-}"
restore_database="${POSTGRES_DB}_restore_test"

if [[ -z "$dump_file" || ! -s "$dump_file" ]]; then
    echo "Penggunaan: $0 backup/nama-file.dump" >&2
    exit 1
fi

if [[ -f "${dump_file}.sha256" ]]; then
    sha256sum --check "${dump_file}.sha256"
fi

docker compose exec -T postgres-db \
    dropdb --username "$POSTGRES_USER" --if-exists "$restore_database"

docker compose exec -T postgres-db \
    createdb --username "$POSTGRES_USER" "$restore_database"

docker compose exec -T postgres-db \
    pg_restore \
    --username "$POSTGRES_USER" \
    --dbname "$restore_database" \
    --no-owner \
    --no-privileges \
    < "$dump_file"

docker compose exec -T postgres-db \
    psql \
    --username "$POSTGRES_USER" \
    --dbname "$restore_database" \
    --command "SELECT COUNT(*) AS total_students FROM students;"

echo "[PASS] Restore berhasil ke database: $restore_database"
```

## Perintah B — Permission dan Validasi File

```bash
cd ~/docker-lab/bab-5
chmod +x scripts/backup.sh scripts/restore-test.sh

find . -maxdepth 3 -type f | sort
docker compose config
docker compose config --services
```

Service yang diharapkan:

```text
postgres-db
pgadmin
```

## Perintah C — Menjalankan Stack

```bash
docker compose up -d
docker compose ps
docker compose logs --tail 100
```

Tunggu sampai PostgreSQL berstatus `healthy`. Akses pgAdmin melalui:

```text
http://localhost:5050
```

Login menggunakan nilai `PGADMIN_DEFAULT_EMAIL` dan `PGADMIN_DEFAULT_PASSWORD` pada `.env`. Server “PostgreSQL Bab 5” akan tersedia; masukkan password PostgreSQL ketika diminta.

## Perintah D — Menguji Schema dan Data Awal

```bash
docker compose exec -T postgres-db \
  psql -U labuser -d labdb \
  -c "\dt"

docker compose exec -T postgres-db \
  psql -U labuser -d labdb \
  -c "SELECT id, nrp, name, created_at FROM students ORDER BY id;"
```

Keluaran harus menampilkan tabel `students` dan sedikitnya dua baris data awal.

## Perintah E — Menguji Persistensi Volume

Tambahkan satu record:

```bash
docker compose exec -T postgres-db \
  psql -U labuser -d labdb \
  -c "INSERT INTO students(nrp, name) VALUES ('31230003', 'Mahasiswa Tiga') ON CONFLICT DO NOTHING;"
```

Buat ulang container tanpa menghapus volume:

```bash
docker compose down
docker compose up -d
docker compose exec -T postgres-db \
  psql -U labuser -d labdb \
  -c "SELECT nrp, name FROM students ORDER BY nrp;"
```

Record `31230003` harus tetap tersedia.

## Perintah F — Backup

```bash
./scripts/backup.sh
ls -lh backup/
sha256sum --check backup/*.dump.sha256
```

Jangan menyatakan backup berhasil hanya karena file ada. File harus berukuran lebih dari nol, checksum valid, dan restore harus diuji.

## Perintah G — Restore Test

Pilih dump terbaru:

```bash
latest_dump="$(find backup -maxdepth 1 -name '*.dump' -type f | sort | tail -n 1)"
./scripts/restore-test.sh "$latest_dump"
```

Periksa database hasil restore:

```bash
docker compose exec -T postgres-db \
  psql -U labuser -d labdb_restore_test \
  -c "SELECT nrp, name FROM students ORDER BY nrp;"
```

## Perintah H — Monitoring Dasar

```bash
docker compose ps
docker compose logs postgres-db --tail 100
docker compose exec -T postgres-db pg_isready -U labuser -d labdb
docker compose exec -T postgres-db psql -U labuser -d labdb -c "SELECT version();"
docker volume ls
docker system df -v
```

## Troubleshooting Cepat

| Gejala | Pemeriksaan | Perbaikan utama |
|---|---|---|
| Init SQL tidak dijalankan ulang | Periksa volume `pg-data` | Init script hanya berjalan pada volume kosong; gunakan migration untuk perubahan berikutnya |
| `relation students does not exist` | Log PostgreSQL dan volume | Periksa sintaks SQL; reset hanya jika data lab boleh dihapus |
| pgAdmin tidak terhubung | Host dan network | Gunakan hostname `postgres-db`, bukan `localhost` |
| Port 5432 digunakan | `ss -lntp` | Ubah host port atau hapus published port PostgreSQL |
| `permission denied` saat backup | Permission direktori host | Pastikan user dapat menulis ke `backup/` |
| Dump kosong | Status DB dan error `pg_dump` | Jalankan backup kembali; jangan gunakan file kosong |
| Restore gagal | Checksum, versi, atau role | Verifikasi checksum dan baca keluaran `pg_restore` |
| Password lama tetap berlaku | Volume sudah diinisialisasi | Variable init tidak mengubah database yang telah ada; ubah password melalui SQL |

## Perintah I — Cleanup Aman

Menghapus container dan network tetapi mempertahankan data:

```bash
cd ~/docker-lab/bab-5
docker compose down
```

Menghapus database hasil restore test:

```bash
docker compose up -d postgres-db
docker compose exec -T postgres-db \
  dropdb -U labuser --if-exists labdb_restore_test
```

Perintah berikut menghapus volume dan seluruh data PostgreSQL serta pgAdmin. Jalankan hanya jika data laboratorium tidak lagi diperlukan:

```bash
docker compose down -v
```

## Checklist PASS

- [ ] `docker compose config` tidak menghasilkan galat.
- [ ] PostgreSQL berstatus `healthy`.
- [ ] Tabel `students` dan data awal tersedia.
- [ ] pgAdmin dapat terhubung menggunakan hostname `postgres-db`.
- [ ] Data tetap tersedia setelah `docker compose down` dan `up`.
- [ ] Backup dump berukuran lebih dari nol.
- [ ] Checksum backup valid.
- [ ] Restore berhasil pada database `labdb_restore_test`.
- [ ] Jumlah dan isi record hasil restore sesuai database sumber.
- [ ] Credential dan dump tidak masuk repository Git.
