<a id="bab-07"></a>

# Bab 7 — Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter, dan Grafana

Topik utama: monitoring, Prometheus, PromQL, Grafana, cAdvisor, Node Exporter.

## Tujuan Pembelajaran dan Kompetensi Utama

**Setelah menyelesaikan bab ini, pembaca diharapkan mampu:**

1. Membedakan monitoring metrics dan logging events.

2. Menjalankan Prometheus sebagai scraper dan time-series database.

3. Mengumpulkan metrik host dengan Node Exporter dan metrik container dengan cAdvisor.

4. Membuat dashboard Grafana untuk CPU, memory, disk, network, dan container usage.

5. Mengintegrasikan data log PostgreSQL dari bab sebelumnya ke Grafana.

## Konsep Inti dan Landasan Teori

Monitoring menjawab pertanyaan “apa yang sedang terjadi pada sistem” melalui angka time-series. Logging menjawab “mengapa hal itu terjadi” melalui event dan pesan detail. Keduanya harus digunakan bersama agar incident response tidak berhenti pada gejala.

Prometheus memakai model pull: server melakukan scrape endpoint /metrics dari target secara berkala. Grafana tidak menyimpan metrics utama, tetapi meng-query data source seperti Prometheus atau PostgreSQL untuk visualisasi dan alerting.

PromQL memungkinkan agregasi temporal seperti rate, avg, sum, dan histogram. Kesalahan umum pada pemula adalah membaca counter mentah tanpa rate, sehingga grafik terlihat naik terus tanpa merepresentasikan throughput aktual.

| Komponen | Fungsi | Port umum |
| --- | --- | --- |
| Prometheus | Scrape metrics dan menyimpan time-series | 9090 |
| Node Exporter | Expose metrics host: CPU, RAM, disk, network | 9100 |
| cAdvisor | Expose metrics container Docker | 8080/8081 |
| Grafana | Dashboard dan visualisasi data source | 3000 |
| PostgreSQL datasource | Menampilkan log query dari Bab 6 | 5432 |

| PromQL | Makna |
| --- | --- |
| up | Status scrape target Prometheus |
| rate(node_cpu_seconds_total[5m]) | Laju perubahan counter CPU per 5 menit |
| container_memory_usage_bytes | Pemakaian memori container |
| 100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100 | Estimasi CPU usage host |

### Monitoring sebagai Sistem Pengukuran

Monitoring menyediakan pengukuran berkala mengenai keadaan sistem, sedangkan observability adalah kemampuan menalar keadaan internal berdasarkan keluaran yang tersedia. Metric, log, dan trace saling melengkapi. Metric efisien untuk tren dan alert; log menyimpan detail event; trace menunjukkan lintasan transaksi terdistribusi. Prometheus mengumpulkan time series melalui model pull, menyimpan sample dengan timestamp dan label, serta menyediakan PromQL untuk agregasi dan analisis [6]. Grafana menyajikan data tersebut dalam dashboard, tetapi visualisasi tidak memperbaiki data yang salah, tidak lengkap, atau memiliki label tidak terkendali [7].

Setiap time series dibentuk oleh nama metric dan kombinasi label. Label seperti service, instance, method, status class, dan environment membantu segmentasi. Sebaliknya, memasukkan user ID, request ID, alamat acak, atau pesan error sebagai label menciptakan cardinality tinggi. Cardinality meningkatkan penggunaan memori dan storage serta dapat membuat query mahal. Nilai berdimensi tinggi sebaiknya disimpan pada log atau trace, sedangkan metric mempertahankan dimensi terbatas yang relevan untuk keputusan operasi.

### Jenis Metric dan Semantik

Counter hanya bertambah dan sesuai untuk jumlah request, error, atau byte total. Counter dianalisis sebagai rate, bukan nilai absolut, karena proses dapat restart. Gauge dapat naik dan turun, misalnya penggunaan memori, panjang antrean, atau jumlah koneksi. Histogram mengelompokkan observasi pada bucket dan memungkinkan estimasi quantile pada agregasi; summary menghitung quantile pada client dengan trade-off agregasi yang berbeda. Pemilihan tipe harus mengikuti semantik fenomena, bukan tampilan dashboard yang diinginkan.

Resource metric perlu dibaca bersama konteks. CPU usage tinggi dapat menunjukkan beban produktif atau loop; CPU throttling menunjukkan limit cgroup menahan workload. Memori working set berbeda dari cache; mendekati limit belum tentu error, tetapi OOM event dan restart merupakan evidence kuat. I/O latency, filesystem usage, packet drop, dan connection saturation dapat menjelaskan respons lambat ketika CPU tampak normal. cAdvisor mengekspos metric container, sedangkan Node Exporter mengekspos metric host. Keduanya menjawab level pertanyaan yang berbeda dan harus diberi label yang jelas.

### Golden Signals, SLI, dan SLO

Empat golden signals—latency, traffic, errors, dan saturation—membantu memilih pengukuran yang berorientasi layanan. Service level indicator adalah ukuran, misalnya proporsi request berhasil di bawah batas latency. Service level objective menetapkan target pada periode tertentu. Alert yang bermakna menghubungkan gejala dengan risiko terhadap SLO, bukan hanya threshold resource sesaat. CPU 85 persen selama satu menit mungkin normal pada batch job, sedangkan error rate kecil pada endpoint pembayaran dapat kritis.

Error budget menerjemahkan target reliability menjadi toleransi kegagalan. Jika budget cepat habis, organisasi mengutamakan stabilitas; jika sehat, perubahan dapat dilakukan dengan risiko terukur. Praktikum tidak perlu membangun program SRE lengkap, tetapi dashboard harus menjawab pertanyaan operasional: apakah pengguna terdampak, service mana yang menjadi bottleneck, sejak kapan, dan perubahan apa yang bertepatan. Panel tanpa pertanyaan yang jelas mudah menjadi dekorasi.

### Scraping, Service Discovery, dan Keamanan

Prometheus melakukan scrape terhadap endpoint metric. Target yang berstatus `up=1` hanya membuktikan endpoint berhasil dikunjungi, bukan bahwa aplikasi memenuhi fungsi bisnis. Interval scrape menentukan resolusi dan biaya. Timeout harus lebih pendek daripada interval. Label relabeling digunakan untuk menjaga identitas target konsisten. Pada Compose, nama service dapat digunakan melalui DNS internal, sehingga endpoint metric tidak harus dipublikasikan ke host.

Endpoint metric dapat membocorkan versi, nama host, route, pola beban, atau identifier. Aksesnya harus dibatasi oleh network, autentikasi, TLS, atau proxy sesuai risiko. Grafana juga memerlukan kontrol akses; akun anonim dan credential default tidak sesuai untuk produksi. Data source credential harus dikelola sebagai secret. Dashboard serta alert rule adalah configuration-as-code yang perlu direview, terversi, dan diuji.

### Alerting dan Kualitas Sinyal

Alert harus actionable: penerima memahami dampak, urgensi, konteks, dan langkah awal. Alert terlalu sensitif menghasilkan fatigue; alert terlalu lambat memperpanjang waktu pemulihan. Kondisi `for` membantu menahan spike singkat. Severity, owner, runbook, dan deduplication perlu konsisten. Alert juga harus diuji melalui failure injection yang aman, misalnya menghentikan service laboratorium, memberikan beban terkontrol, atau menurunkan limit resource. Keberhasilan alert dibuktikan oleh perubahan state, notifikasi, dan recovery, bukan hanya keberadaan rule.

Pengujian monitoring mencakup empat lapisan: target dapat discrape; query menghasilkan nilai benar; dashboard menyajikan unit dan rentang tepat; alert berubah state pada kondisi yang dirancang. Evidence perlu mencatat waktu, versi konfigurasi, query, dan event pemicu. Dengan cara ini, monitoring menjadi kontrol detektif yang dapat dipercaya dan menyediakan feedback bagi capacity planning, troubleshooting, serta keputusan keamanan.

### Monitoring sebagai Umpan Balik DevSecOps

Monitoring menutup feedback loop dari operasi kembali ke pengembangan. Temuan latency, error, restart, atau resource saturation dapat menghasilkan test regresi, perubahan limit, optimasi query, dan pembaruan threat model. Dashboard karena itu perlu menghubungkan metric dengan versi rilis atau waktu deployment. Anotasi perubahan membantu membedakan degradasi yang disebabkan traffic dari degradasi setelah release.

Data monitoring juga memiliki batas. Scrape interval dapat melewatkan spike singkat, agregasi dapat menyembunyikan outlier, dan dashboard dapat menampilkan keadaan lama ketika query gagal. Mahasiswa harus menyatakan resolusi, periode, serta asumsi query ketika menarik kesimpulan. Evidence yang baik menyertakan ekspresi PromQL, waktu observasi, label target, dan event pemicu. Sikap kritis tersebut mencegah keputusan hanya berdasarkan warna panel dan menjadikan metric dasar analisis yang dapat diuji.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan. Jalankan perintah cleanup setelah praktikum selesai, terutama jika port yang sama digunakan pada bab berikutnya.

```bash
# Struktur direktori umum Bab 7
mkdir -p ~/docker-lab/bab-7
cd ~/docker-lab/bab-7
# Simpan file compose, Dockerfile, konfigurasi, dan log di direktori ini.
```

## Langkah Praktikum Eksploratif

### prometheus.yml dasar

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```yaml
global:
  scrape_interval: 15s
scrape_configs:
  - job_name: prometheus
    static_configs:
      - targets: ["prometheus:9090"]
  - job_name: node
    static_configs:
      - targets: ["node-exporter:9100"]
  - job_name: cadvisor
    static_configs:
      - targets: ["cadvisor:8080"]
```

### Compose monitoring stack

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```yaml
services:
  prometheus:
    image: prom/prometheus:latest
    ports: ["9090:9090"]
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prom-data:/prometheus
  grafana:
    image: grafana/grafana:latest
    ports: ["3000:3000"]
    volumes:
      - grafana-data:/var/lib/grafana
    depends_on: [prometheus]
  node-exporter:
    image: prom/node-exporter:latest
    pid: host
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    ports: ["8081:8080"]
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
volumes:
  prom-data:
  grafana-data:
```

### Query verifikasi Prometheus

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
# Buka http://localhost:9090 dan jalankan query:
up
rate(container_cpu_usage_seconds_total[5m])
container_memory_usage_bytes
node_filesystem_avail_bytes
```

## Verifikasi dan Skenario Pengujian

[ ] Semua target Prometheus berstatus up.

[ ] Grafana berhasil menambahkan Prometheus sebagai data source.

[ ] Dashboard menampilkan CPU, memory, dan container metrics.

[ ] Grafana dapat membaca data source PostgreSQL untuk log lab.

[ ] Mahasiswa mampu menjelaskan perbedaan rate dan counter mentah.

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

1. Mengapa Prometheus menggunakan scrape model dan bukan hanya push model?

2. Apa dampak scrape_interval terlalu kecil terhadap storage dan network?

3. Mengapa rate diperlukan untuk counter seperti node_cpu_seconds_total?

4. Apa indikator awal container mengalami memory pressure?

5. Bagaimana menghubungkan alert metrics dengan pencarian log?

## Format Laporan Praktikum

Laporan Bab 7 dikumpulkan dalam PDF maksimum 5 halaman. Isi laporan harus menunjukkan bukti eksekusi, analisis, dan refleksi keamanan/operasional.

Bukti minimum:

- Screenshot docker compose ps atau docker ps yang menunjukkan service berjalan.

- Screenshot hasil curl/browser/API/dashboard sesuai target bab.

- Cuplikan log atau query yang membuktikan sistem bekerja.

Analisis wajib:

- Jelaskan satu masalah yang muncul dan cara Anda mendiagnosisnya.

- Jelaskan risiko keamanan atau operasional yang relevan pada bab ini.

- Berikan rekomendasi perbaikan bila lab ini akan dibawa ke production-like environment.
