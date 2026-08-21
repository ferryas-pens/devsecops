<a id="bab-06"></a>

# Bab 6 — Centralized Logging dengan Fluent Bit dan PostgreSQL

Topik utama: centralized logging, Fluent Bit, Docker logging driver, PostgreSQL JSONB, retention.

## Tujuan Pembelajaran dan Kompetensi Utama

**Setelah menyelesaikan bab ini, pembaca diharapkan mampu:**

1. Menjelaskan kebutuhan centralized logging pada lingkungan container yang ephemeral.

2. Mengirim log stdout/stderr container ke Fluent Bit menggunakan Docker fluentd logging driver.

3. Menyimpan log ke PostgreSQL dalam kolom JSONB dan membuat view analisis.

4. Membuat log generator untuk mensimulasikan INFO, DEBUG, WARN, ERROR, dan CRITICAL.

5. Menerapkan query pencarian log dan retention sederhana.

## Konsep Inti dan Landasan Teori

Container sering dibuat dan dihapus secara cepat. Bila log hanya tersimpan di container lokal, debugging dan audit menjadi sulit. Centralized logging memindahkan log ke satu storage sehingga dapat dicari, dianalisis, dan diaudit lebih konsisten.

Fluent Bit ringan dan cocok sebagai collector di edge atau lab. Docker logging driver fluentd mengirim log dari stdout/stderr container ke Fluent Bit/Fluentd, lalu output plugin dapat menyimpan data ke storage seperti PostgreSQL.

Menyimpan log sebagai JSONB memungkinkan query fleksibel terhadap field log terstruktur. Namun, database relasional bukan selalu pilihan terbaik untuk volume log sangat besar; untuk produksi, evaluasi Loki, OpenSearch, ClickHouse, atau pipeline observability yang lebih scalable.

| Field | Makna |
| --- | --- |
| tag | Klasifikasi sumber log, misalnya docker.nginx atau docker.flask |
| time | Timestamp log diterima collector |
| data | Record log penuh dalam format JSON/JSONB |
| container_name | Nama container yang menghasilkan log |
| source | stdout atau stderr |

### Centralized Logging dan Observability

Log adalah rekaman diskret mengenai peristiwa yang terjadi pada aplikasi, container, host, maupun kontrol keamanan. Centralized logging mengumpulkan rekaman tersebut ke lokasi yang dapat dicari dan dianalisis lintas service. Tujuannya bukan sekadar memindahkan teks dari `stdout`; sistem harus mempertahankan konteks, urutan waktu, integritas field, serta kemampuan menghubungkan satu request dengan komponen lain. Fluent Bit berperan sebagai agen atau router telemetry yang membaca input, melakukan parsing dan filtering, lalu mengirimkan record ke output yang ditetapkan [5].

Dalam lingkungan container, proses idealnya menulis log aplikasi ke `stdout` dan `stderr`, kemudian runtime atau collector meneruskannya. Pola ini memisahkan aplikasi dari detail backend logging. Namun, log driver yang blocking atau buffer yang tidak terkendali dapat memengaruhi availability aplikasi. Desain harus menetapkan apa yang terjadi ketika tujuan log lambat atau tidak tersedia: apakah record dibuffer, dibuang, atau membuat producer tertahan. Tidak ada jawaban universal; keputusan bergantung pada kritikalitas evidence, toleransi kehilangan, kapasitas disk, dan kebutuhan latency.

### Struktur Event dan Kualitas Data

Structured logging menggunakan field yang konsisten, misalnya timestamp, severity, service, environment, event type, request ID, trace ID, principal, status, duration, dan outcome. JSON sering digunakan karena mudah diparse, tetapi JSON yang valid belum menjamin semantik yang konsisten. Nama field, satuan durasi, zona waktu, dan kategori error harus distandardisasi. Timestamp sebaiknya memakai UTC dan format yang tidak ambigu; sinkronisasi waktu host penting agar korelasi lintas komponen tidak menyesatkan.

Correlation ID memungkinkan penelusuran satu transaksi melalui reverse proxy, API, database, dan pipeline asynchronous. ID tersebut harus diteruskan atau dibuat pada trust boundary yang jelas. Nilai dari client tidak boleh langsung dianggap tepercaya untuk kebutuhan audit tanpa validasi. Log juga perlu membedakan kejadian bisnis, kejadian operasional, dan kejadian keamanan. Error stack membantu debugging, tetapi security event memerlukan konteks keputusan seperti subject, resource, policy, action, dan alasan penolakan.

### Pipeline Input, Parser, Filter, dan Output

Pipeline Fluent Bit terdiri atas input, parser, filter, buffer, dan output. Input menentukan sumber record; parser mengubah representasi teks menjadi field; filter menambah, menghapus, atau mentransformasi field; output mengirim data ke PostgreSQL atau sistem observability lain. Kegagalan parsing tidak boleh diam-diam menghasilkan data tanpa struktur. Record yang gagal perlu diberi tag, dikirim ke dead-letter destination, atau dihitung sebagai metric agar penurunan kualitas data terdeteksi.

Filter dapat menambah metadata container dan environment, tetapi enrichment mempunyai biaya CPU, memori, dan lookup. Metadata cardinality tinggi meningkatkan storage dan memperlambat query. Prinsip minimisasi data berlaku: hanya field yang diperlukan untuk operasi, keamanan, kepatuhan, atau analisis yang disimpan. Secret, access token, password, session identifier, dan data pribadi tidak boleh dicatat secara default. Redaction sebaiknya dilakukan sedekat mungkin dengan producer, kemudian diverifikasi kembali pada collector.

### PostgreSQL sebagai Backend Laboratorium

PostgreSQL memudahkan mahasiswa mempelajari schema, query, indeks, retensi, dan akses terkontrol. Walaupun demikian, database relasional bukan selalu backend log skala besar yang paling efisien. Volume event tinggi, ingest kontinu, retensi panjang, serta pencarian teks dapat memerlukan sistem khusus. Dalam laboratorium, tabel perlu membatasi panjang message, menetapkan tipe timestamp, menggunakan indeks pada waktu, service, severity, dan correlation ID, serta memisahkan credential writer dari credential analyst [4].

Retensi harus menjadi kebijakan eksplisit. Menyimpan seluruh log tanpa batas meningkatkan biaya dan risiko privasi; menghapus terlalu cepat menghilangkan evidence insiden. Partisi berdasarkan waktu, job penghapusan, archive, dan backup perlu disesuaikan dengan kebutuhan. Integritas log juga tidak identik dengan keberadaan row. Untuk kebutuhan forensik yang lebih tinggi, akses perubahan dibatasi, audit perubahan diaktifkan, dan hasil ekspor dapat diberi checksum atau disimpan pada media immutable.

### Reliability, Backpressure, dan Pengujian

Backpressure terjadi ketika kecepatan output lebih rendah daripada input. Buffer memori cepat tetapi hilang ketika proses berhenti; buffer disk lebih tahan tetapi dapat memenuhi filesystem. Batas retry tanpa kebijakan dapat menghasilkan loop tak berujung. Desain harus mendokumentasikan kapasitas buffer, retry, timeout, dan perilaku drop. Metric collector seperti jumlah record masuk, record gagal parse, retry, dropped record, penggunaan buffer, dan latency output diperlukan untuk membuktikan pipeline logging sehat.

Pengujian tidak cukup dengan menemukan satu baris log pada database. Skenario harus mencakup event normal, error aplikasi, data multiline, karakter khusus, output sementara tidak tersedia, dan record yang mengandung nilai sensitif. PASS berarti field penting dapat dicari, waktu konsisten, secret tidak muncul, serta kegagalan delivery terlihat melalui metric atau log internal. Analisis kemudian membandingkan completeness, timeliness, correctness, dan confidentiality. Centralized logging yang aman adalah pipeline data terkelola, bukan sekadar koleksi pesan.

### Log sebagai Evidence dan Produk Data

Log perlu mempunyai pemilik dan kontrak data. Tim aplikasi bertanggung jawab atas makna event, tim platform menjaga delivery, sedangkan tim keamanan menetapkan event yang diperlukan untuk deteksi dan investigasi. Perubahan schema log dapat merusak parser, dashboard, atau alert; karena itu perubahan field perlu diuji seperti perubahan API. Versi schema atau mekanisme backward compatibility membantu consumer beradaptasi.

Evidence dari logging harus dapat menjawab siapa melakukan apa, pada resource mana, kapan, dari konteks apa, dan dengan hasil apa, tanpa mengumpulkan data berlebihan. Akses analyst dibuat read-only dan query administratif diaudit. Saat insiden, ekspor log disertai rentang waktu, query, zona waktu, dan checksum agar analisis dapat diulang. Pendekatan ini menempatkan centralized logging sebagai produk data yang memiliki kualitas, keamanan, retensi, dan service level, bukan repositori teks tanpa struktur.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan. Jalankan perintah cleanup setelah praktikum selesai, terutama jika port yang sama digunakan pada bab berikutnya.

```bash
# Struktur direktori umum Bab 6
mkdir -p ~/docker-lab/bab-6
cd ~/docker-lab/bab-6
# Simpan file compose, Dockerfile, konfigurasi, dan log di direktori ini.
```

## Langkah Praktikum Eksploratif

### Schema PostgreSQL untuk log

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```dockerfile
CREATE SCHEMA IF NOT EXISTS logs;
CREATE TABLE logs.fluentbit (
  id BIGSERIAL PRIMARY KEY,
  tag VARCHAR(200),
  time TIMESTAMP,
  data JSONB
);
CREATE INDEX idx_fb_time ON logs.fluentbit(time);
CREATE INDEX idx_fb_tag ON logs.fluentbit(tag);
CREATE INDEX idx_fb_data ON logs.fluentbit USING GIN(data);
CREATE OR REPLACE VIEW logs.recent_logs AS
SELECT id, time, tag,
       REPLACE(data->>'container_name', '/', '') AS container,
       data->>'source' AS source,
       LEFT(data->>'log', 200) AS preview
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 100;
```

### Konfigurasi Fluent Bit

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
[SERVICE]
  Flush 5
  Daemon Off
  Log_Level info
[INPUT]
  Name forward
  Listen 0.0.0.0
  Port 24224
[OUTPUT]
  Name pgsql
  Match *
  Host postgres-db
  Port 5432
  User labuser
  Password labpass123
  Database labdb
  Table fluentbit
  Schema logs
  Timestamp_Key time
[OUTPUT]
  Name stdout
  Match *
  Format json_lines
```

### Compose logging driver

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
logging:
  driver: fluentd
  options:
    fluentd-address: "localhost:24224"
    fluentd-async: "true"
    tag: "docker.generator"
```

### Query analisis log

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```dockerfile
SELECT tag, COUNT(*) AS total
FROM logs.fluentbit
GROUP BY tag
ORDER BY total DESC;
SELECT time, tag, LEFT(data->>'log', 120) AS log_preview
FROM logs.fluentbit
ORDER BY time DESC
LIMIT 10;
```

## Verifikasi dan Skenario Pengujian

[ ] Fluent Bit menerima log dari minimal tiga container.

[ ] Tabel logs.fluentbit berisi data dengan tag, time, dan data JSONB.

[ ] View recent_logs menampilkan log terbaru.

[ ] Query distribusi per tag dan pencarian keyword berjalan.

[ ] Mahasiswa memahami apa yang terjadi jika collector mati sementara.

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

1. Apa risiko kehilangan log ketika Fluent Bit tidak tersedia?

2. Mengapa structured logging lebih mudah dianalisis dibanding plain text logging?

3. Kapan PostgreSQL tidak lagi cocok sebagai log storage utama?

4. Apa perbedaan log untuk observability dan log untuk forensik?

5. Buat query SQL untuk menampilkan jumlah log per menit dalam 10 menit terakhir.

## Format Laporan Praktikum

Laporan Bab 6 dikumpulkan dalam PDF maksimum 5 halaman. Isi laporan harus menunjukkan bukti eksekusi, analisis, dan refleksi keamanan/operasional.

Bukti minimum:

- Screenshot docker compose ps atau docker ps yang menunjukkan service berjalan.

- Screenshot hasil curl/browser/API/dashboard sesuai target bab.

- Cuplikan log atau query yang membuktikan sistem bekerja.

Analisis wajib:

- Jelaskan satu masalah yang muncul dan cara Anda mendiagnosisnya.

- Jelaskan risiko keamanan atau operasional yang relevan pada bab ini.

- Berikan rekomendasi perbaikan bila lab ini akan dibawa ke production-like environment.
