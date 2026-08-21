<a id="bab-05"></a>

# Bab 5 — Database Service di Docker: PostgreSQL

Topik utama: PostgreSQL, pgAdmin, init script, volume, backup restore.

## Tujuan Pembelajaran dan Kompetensi Utama

**Setelah menyelesaikan bab ini, pembaca diharapkan mampu:**

1. Menjalankan PostgreSQL dengan konfigurasi berbasis environment variable dan volume persisten.

2. Menginisialisasi database menggunakan file SQL pada docker-entrypoint-initdb.d.

3. Mengakses database melalui psql dan pgAdmin4.

4. Melakukan backup dan restore menggunakan pg_dump dan pg_restore.

5. Menerapkan healthcheck pada service PostgreSQL.

## Konsep Inti dan Landasan Teori

Database container berbeda dari stateless service. Data harus diperlakukan sebagai aset utama, sehingga volume, backup, restore, permission, dan lifecycle container harus dirancang dengan hati-hati.

Image PostgreSQL resmi mengeksekusi file .sql dan .sh di /docker-entrypoint-initdb.d hanya saat volume data masih kosong. Ini sering menjadi sumber kebingungan: mengubah init script tidak akan berpengaruh bila volume lama masih dipakai.

Untuk produksi, pertimbangkan eksternal database managed service, backup berkala, monitoring WAL, strategi restore yang diuji, dan secret management yang lebih baik daripada environment variable plaintext.

| Variabel | Fungsi |
| --- | --- |
| POSTGRES_DB | Nama database awal yang dibuat ketika inisialisasi |
| POSTGRES_USER | User awal/superuser container |
| POSTGRES_PASSWORD | Password awal; wajib diisi untuk lab sederhana |
| PGDATA | Direktori data PostgreSQL di dalam container |
| POSTGRES_INITDB_ARGS | Argumen tambahan initdb, misalnya encoding atau locale |

### Database sebagai Sistem Persisten

PostgreSQL di dalam container tetap merupakan sistem database relasional penuh; container hanya mengubah cara proses dan dependensinya dikemas. Data tidak boleh dianggap melekat pada umur container. Writable layer sesuai untuk perubahan sementara, sedangkan data database memerlukan volume persisten yang dapat dipasang kembali ketika container diganti. Dokumentasi Docker membedakan volume yang dikelola engine, bind mount yang menunjuk path host, dan tmpfs yang bersifat sementara [50-53]. Untuk laboratorium PostgreSQL, named volume memberikan pemisahan yang lebih aman daripada menyimpan data pada writable layer, tetapi volume tetap bukan backup.

Persistensi mempunyai empat dimensi: durability transaksi, keberlangsungan media, kemampuan backup, dan kemampuan restore. PostgreSQL menjaga konsistensi transaksi melalui write-ahead logging dan mekanisme recovery, tetapi kerusakan storage, kesalahan operator, atau penghapusan volume masih dapat menghilangkan data. Backup baru bernilai apabila dapat dipulihkan. Praktikum karena itu harus menyertakan uji restore ke instance terpisah, pemeriksaan jumlah baris atau checksum logis, dan pencatatan versi PostgreSQL. Dokumentasi PostgreSQL menjadi rujukan utama untuk kompatibilitas format, `pg_dump`, `pg_restore`, dan prosedur pemulihan [4].

### Transaksi, Integritas, dan Konkurensi

Model ACID menjelaskan properti yang diharapkan dari transaksi. Atomicity memastikan seluruh perubahan berhasil atau dibatalkan sebagai satu unit; consistency menjaga aturan integritas; isolation mengatur interaksi transaksi serentak; dan durability mempertahankan commit setelah kegagalan. Constraint seperti primary key, foreign key, unique, check, dan not-null bukan sekadar dokumentasi skema. Constraint adalah kontrol yang menolak state tidak valid meskipun aplikasi memiliki bug atau terdapat beberapa jalur penulisan data.

Isolation level memengaruhi keseimbangan konsistensi dan konkurensi. Pada beban serentak, aplikasi dapat mengalami lost update, non-repeatable read, atau serialization failure bergantung pada pola transaksi. Solusinya bukan selalu meningkatkan isolation ke tingkat tertinggi. Transaksi harus dibuat sesingkat mungkin, query diberi indeks yang tepat, konflik ditangani secara eksplisit, dan retry hanya dilakukan untuk operasi yang aman diulang. Analisis database harus membedakan error koneksi, pelanggaran constraint, deadlock, dan kekurangan resource karena tindakan korektifnya berbeda.

### Identitas, Hak Akses, dan Secret

Prinsip least privilege mengharuskan akun aplikasi hanya memperoleh hak yang diperlukan pada database, schema, table, sequence, atau function. Akun administrasi tidak boleh digunakan oleh aplikasi harian. Pemisahan role migrasi, runtime read-write, runtime read-only, backup, dan observability mengurangi dampak credential yang bocor. PostgreSQL mendukung role membership serta privilege granular; rancangan harus menguji bahwa operasi sah diterima dan operasi di luar kewenangan ditolak.

Credential tidak boleh ditanam pada Dockerfile, image layer, atau repository. Compose mendukung secret yang dipasang sebagai file runtime, sedangkan variable lingkungan lebih mudah terekspos melalui konfigurasi efektif, proses, atau log [58,59]. Dalam laboratorium, `.env` dapat membantu demonstrasi, tetapi file tersebut harus dikecualikan dari version control dan tidak diperlakukan sebagai secret manager produksi. Rotasi credential perlu mencakup pembuatan nilai baru, pembaruan consumer, pencabutan nilai lama, dan verifikasi bahwa koneksi aktif tidak mempertahankan akses tanpa batas.

### Network, Ketersediaan, dan Resource

PostgreSQL sebaiknya ditempatkan pada network internal dan tidak dipublikasikan ke seluruh interface host kecuali benar-benar diperlukan. `listen_addresses`, aturan autentikasi, firewall, dan kebijakan network harus membentuk lapisan yang konsisten. TLS database diperlukan ketika trafik melewati jaringan yang tidak sepenuhnya dipercaya. Enkripsi storage atau volume melindungi media, sedangkan TLS melindungi data dalam transit; keduanya menyelesaikan ancaman berbeda.

Database sensitif terhadap memori, I/O, jumlah koneksi, dan latency storage. Limit container mencegah satu workload menghabiskan resource host, tetapi limit yang terlalu kecil dapat memicu OOM atau penurunan performa. Connection pool membantu membatasi jumlah backend PostgreSQL, namun pool yang terlalu besar memindahkan masalah ke database. Health check `pg_isready` menunjukkan server menerima koneksi, bukan bahwa migrasi lengkap, query bisnis benar, atau replica mutakhir. Kriteria penerimaan perlu menguji query representatif, constraint, backup-restore, dan negative access.

### Skema, Migrasi, dan Evidence

Skema database adalah bagian dari artefak aplikasi dan harus terversi. Migrasi perlu memiliki urutan, pemilik, kondisi awal, serta strategi rollback atau forward-fix. Perubahan destruktif sebaiknya dilakukan bertahap: menambah struktur kompatibel, memigrasikan data, memindahkan aplikasi, lalu menghapus struktur lama setelah observasi memadai. Menjalankan migrasi otomatis pada startup setiap replica dapat menciptakan race condition; organisasi perlu menunjuk job atau tahap pipeline khusus.

Evidence praktikum yang sah meliputi digest image, status health, daftar role dan privilege yang relevan, hasil transaksi, bukti constraint menolak data invalid, checksum backup, dan keberhasilan restore. Dengan pendekatan ini, menjalankan PostgreSQL di Docker tidak berhenti pada keberhasilan `docker compose up`, tetapi mencakup integritas, confidentiality, availability, dan recoverability sebagai satu sistem.

### Tata Kelola Data dan Perubahan Operasional

Data memiliki klasifikasi, pemilik, masa retensi, dan kebutuhan pemulihan. Konfigurasi container tidak boleh mengaburkan tanggung jawab tersebut. Sebelum membuat backup, tim menentukan Recovery Point Objective dan Recovery Time Objective sebagai target kehilangan data dan waktu pemulihan yang dapat diterima. Backup periodik, enkripsi, pemisahan akses, serta salinan di lokasi berbeda disesuaikan dengan target itu. Restore exercise harus mengukur waktu aktual dan mendokumentasikan hambatan.

Perubahan versi PostgreSQL juga memerlukan perhatian. Tag image yang berubah dapat membawa perbedaan major version dan format data. Upgrade tidak dilakukan dengan mengganti tag secara buta; compatibility note, backup, metode migrasi, extension, dan rollback perlu diuji pada salinan data. Observability mencakup connection count, transaction rate, lock, replication lag bila ada, disk growth, dan query latency. Dengan demikian, operasional database menggabungkan kontrol teknis dan tata kelola: siapa boleh mengubah skema, bagaimana perubahan disetujui, evidence apa yang disimpan, serta siapa menerima residual risk setelah pengujian.

Setiap keputusan tersebut dicatat bersama versi image dan identitas dataset uji agar hasil dapat direproduksi.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan. Jalankan perintah cleanup setelah praktikum selesai, terutama jika port yang sama digunakan pada bab berikutnya.

```bash
# Struktur direktori umum Bab 5
mkdir -p ~/docker-lab/bab-5
cd ~/docker-lab/bab-5
# Simpan file compose, Dockerfile, konfigurasi, dan log di direktori ini.
```

## Langkah Praktikum Eksploratif

### Compose PostgreSQL dan pgAdmin

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```yaml
services:
  postgres-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: labdb
      POSTGRES_USER: labuser
      POSTGRES_PASSWORD: labpass123
    ports:
      - "5432:5432"
    volumes:
      - pg-data:/var/lib/postgresql/data
      - ./init:/docker-entrypoint-initdb.d:ro
      - ./backup:/backup
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U labuser -d labdb"]
      interval: 5s
      timeout: 5s
      retries: 5
  pgadmin:
    image: dpage/pgadmin4:latest
    environment:
      PGADMIN_DEFAULT_EMAIL: admin@pens.ac.id
      PGADMIN_DEFAULT_PASSWORD: admin123
    ports:
      - "5050:80"
volumes:
  pg-data:
```

### Inisialisasi schema

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```sql
mkdir -p init backup
cat > init/01-schema.sql << 'EOF'
CREATE TABLE IF NOT EXISTS students (
  id SERIAL PRIMARY KEY,
  nrp VARCHAR(20) UNIQUE NOT NULL,
  name TEXT NOT NULL,
  created_at TIMESTAMP DEFAULT now()
);
INSERT INTO students(nrp, name) VALUES
('31230001', 'Mahasiswa Satu'),
('31230002', 'Mahasiswa Dua')
ON CONFLICT DO NOTHING;
EOF
docker compose up -d
```

### Backup dan restore

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
docker exec postgres-db pg_dump -U labuser -d labdb -Fc -f /backup/labdb.dump
docker exec postgres-db pg_restore -U labuser -d labdb --clean --if-exists /backup/labdb.dump
docker exec -it postgres-db psql -U labuser -d labdb -c "SELECT * FROM students;"
```

## Verifikasi dan Skenario Pengujian

[ ] PostgreSQL berjalan dengan volume pg-data.

[ ] Tabel students dibuat otomatis dari init script.

[ ] pgAdmin dapat login dan terkoneksi ke database.

[ ] Backup menghasilkan file dump di direktori host.

[ ] Restore sudah diuji, bukan hanya diasumsikan berhasil.

| Prinsip troubleshooting: Mulai dari status container, baca logs, cek network, cek volume, lalu validasi konfigurasi. Jangan langsung menghapus volume sebelum memahami apakah data masih dibutuhkan. |
| --- |

```bash
docker compose ps
docker compose logs --tail 100
curl -v http://localhost:8080
docker network ls
docker volume ls
docker inspect <container-name>
```

## Troubleshooting dan Analisis Hasil

| Gejala | Penyebab yang mungkin | Tindakan korektif |
| --- | --- | --- |
| Gate berbeda antara lokal dan pipeline | Versi tool, input efektif, atau konfigurasi tidak sama | Pin versi; simpan konfigurasi efektif dan identitas artefak. |
| Service sehat tetapi security gate gagal | Healthcheck hanya memeriksa availability | Tinjau policy, scan, identity, signature, dan evidence secara terpisah. |
| Evidence tidak dapat ditelusuri | Commit, digest, waktu, atau owner tidak dicatat | Gunakan manifest evidence dan metadata yang konsisten. |
| Deployment gagal dipulihkan | Rollback, backup, atau credential rotation belum diuji | Lakukan recovery exercise dan dokumentasikan hasilnya. |

## Evaluasi dan Latihan Mandiri

1. Mengapa init script tidak dijalankan ulang saat volume lama masih ada?

2. Apa risiko menaruh password database pada docker-compose.yml?

3. Bagaimana cara membuktikan backup dapat dipulihkan?

4. Apa bedanya logical backup pg_dump dan backup filesystem volume mentah?

5. Apa dampak docker compose down -v terhadap database?

## Format Laporan Praktikum

Laporan Bab 5 dikumpulkan dalam PDF maksimum 5 halaman. Isi laporan harus menunjukkan bukti eksekusi, analisis, dan refleksi keamanan/operasional.

Bukti minimum:

- Screenshot docker compose ps atau docker ps yang menunjukkan service berjalan.

- Screenshot hasil curl/browser/API/dashboard sesuai target bab.

- Cuplikan log atau query yang membuktikan sistem bekerja.

Analisis wajib:

- Jelaskan satu masalah yang muncul dan cara Anda mendiagnosisnya.

- Jelaskan risiko keamanan atau operasional yang relevan pada bab ini.

- Berikan rekomendasi perbaikan bila lab ini akan dibawa ke production-like environment.
