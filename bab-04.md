<a id="bab-04"></a>

# Bab 4 — Web Service Container: Apache, Nginx, Reverse Proxy, dan TLS

Topik utama: Apache httpd, Nginx, virtual host, reverse proxy, TLS.

## Tujuan Pembelajaran dan Kompetensi Utama

**Setelah menyelesaikan bab ini, pembaca diharapkan mampu:**

1. Menjalankan Apache dan Nginx sebagai web server container dengan konfigurasi custom.

2. Membuat virtual host berbasis nama untuk beberapa site lab.

3. Menggunakan Nginx sebagai reverse proxy ke backend service.

4. Menerapkan sertifikat TLS self-signed untuk simulasi HTTPS.

5. Membaca access log dan error log web service dari bind mount.

## Konsep Inti dan Landasan Teori

Web server dalam container membuat konfigurasi aplikasi lebih portabel dan repeatable. Image resmi httpd dan nginx dapat digunakan sebagai baseline, lalu dikustomisasi melalui file konfigurasi, bind mount, atau image turunan.

Reverse proxy memisahkan interface publik dari backend internal. Pola ini umum dipakai untuk TLS termination, routing /api ke service backend, dan pembatasan port yang terbuka ke host.

TLS self-signed hanya cocok untuk lab. Di produksi, gunakan certificate authority terpercaya dan otomatisasi renewal, misalnya melalui ACME/Let’s Encrypt di reverse proxy.

| Aspek | Apache httpd | Nginx |
| --- | --- | --- |
| Arsitektur | Process/thread per request | Event-driven asynchronous |
| Direktori config | /usr/local/apache2/conf | /etc/nginx |
| Document root | /usr/local/apache2/htdocs | /usr/share/nginx/html |
| Reverse proxy | mod_proxy | proxy_pass built-in |
| Use case lab | Virtual host dan kompatibilitas .htaccess | Reverse proxy, static file, TLS termination |

### Web Service sebagai Boundary Aplikasi

Web service merupakan titik pertemuan antara client, jaringan, dan logika aplikasi. Pada arsitektur container, Apache HTTP Server atau Nginx dapat berperan sebagai origin server, penyaji berkas statis, maupun reverse proxy. Reverse proxy menerima koneksi dari client, menerapkan kebijakan pada lapisan depan, kemudian meneruskan permintaan ke service internal. Pola ini menyederhanakan terminasi TLS, routing berbasis host atau path, pembatasan ukuran request, pencatatan akses, dan penerapan header keamanan. Meskipun demikian, reverse proxy bukan pengganti kontrol pada aplikasi. Validasi input, autentikasi, otorisasi, dan pengelolaan sesi tetap harus diterapkan pada service yang memiliki konteks bisnis.

Perbedaan antara port container dan published port perlu dipahami sebagai keputusan boundary. Service dapat mendengarkan port 80 di dalam network Compose tanpa dipublikasikan ke host. Hanya reverse proxy yang seharusnya memiliki published port apabila seluruh trafik eksternal wajib melewati proxy. Dokumentasi Docker menjelaskan bahwa publikasi port membuat pemetaan dari host ke container dan, tanpa pembatasan alamat bind, dapat membuka layanan pada interface host yang lebih luas daripada yang dimaksud [43,49]. Oleh sebab itu, database dan backend internal sebaiknya menggunakan network privat, sedangkan exposure ke host dinyatakan secara eksplisit.

### HTTP, Routing, dan Header Terpercaya

HTTP membentuk pasangan request dan response. Reverse proxy harus mempertahankan informasi yang diperlukan backend, seperti host tujuan, skema koneksi, serta alamat client, melalui header yang disepakati. Header `X-Forwarded-For`, `X-Forwarded-Proto`, dan `Host` berguna, tetapi hanya dapat dipercaya jika aplikasi memastikan bahwa request datang dari proxy yang sah. Apabila backend dapat diakses langsung dari luar, penyerang dapat mengirim header palsu dan memengaruhi audit log, pembuatan URL, atau keputusan keamanan. Karena itu, trust terhadap forwarded header harus dibatasi oleh topologi network dan konfigurasi daftar proxy terpercaya.

Routing juga mempunyai implikasi keamanan. Aturan path yang terlalu longgar dapat membuka endpoint administratif, file internal, atau route debugging. Normalisasi URI antara proxy dan aplikasi harus konsisten agar karakter terenkode, garis miring ganda, dan traversal tidak ditafsirkan berbeda. Konfigurasi harus diuji dengan request positif dan negatif, bukan hanya halaman utama. Respons untuk route tidak dikenal, method yang tidak didukung, body terlalu besar, dan upstream yang gagal perlu ditetapkan agar sistem gagal secara terkendali.

### TLS dan Siklus Hidup Sertifikat

Transport Layer Security melindungi kerahasiaan dan integritas data selama transit serta membantu client memverifikasi identitas server. Terminasi TLS pada reverse proxy memusatkan pengelolaan sertifikat, tetapi menciptakan keputusan baru: apakah koneksi proxy-ke-backend tetap terenkripsi, bagaimana private key disimpan, dan siapa yang berhak memperbarui sertifikat. Pada laboratorium, sertifikat self-signed dapat digunakan untuk mempelajari mekanisme, tetapi browser tidak akan mempercayainya secara otomatis. Pada produksi, sertifikat harus berasal dari certificate authority yang dipercaya, memiliki nama host yang tepat, dan diperbarui sebelum kedaluwarsa.

Private key tidak boleh dimasukkan ke image atau repository. Key sebaiknya dipasang saat runtime sebagai secret atau mount read-only dengan permission minimum. Rotasi sertifikat perlu diuji tanpa membangun ulang seluruh aplikasi. Kualitas konfigurasi TLS tidak hanya ditentukan oleh keberhasilan koneksi HTTPS; protokol dan cipher usang, redirect yang salah, atau mixed content masih dapat menurunkan perlindungan. Evidence yang memadai mencakup identitas sertifikat, masa berlaku, hasil handshake, redirect HTTP-ke-HTTPS, dan kegagalan koneksi ketika hostname tidak cocok.

### Availability, Health Check, dan Failure Mode

Reverse proxy harus membedakan kegagalan koneksi, timeout, dan respons aplikasi yang sah. Timeout yang terlalu panjang menahan worker serta memperbesar risiko resource exhaustion, sedangkan timeout terlalu pendek menolak request yang sebenarnya valid. Health check juga perlu dibedakan menjadi liveness dan readiness. Liveness menunjukkan proses masih hidup; readiness menunjukkan service siap menerima trafik, termasuk dependency kritis. Urutan startup pada Compose membantu orkestrasi laboratorium, tetapi status container `running` tidak membuktikan aplikasi siap. Dokumentasi Compose menekankan penggunaan health condition untuk dependency yang memang harus sehat sebelum service lain dimulai [55,57].

Logging akses dan error harus mendukung investigasi tanpa merekam secret, token, atau data pribadi secara berlebihan. Field minimum umumnya meliputi waktu, method, path yang telah disanitasi, status, durasi, ukuran respons, identitas request, dan upstream. Log bukan sekadar keluaran diagnostik; ia merupakan evidence yang harus memiliki timestamp konsisten dan retensi yang sesuai. Dengan demikian, rancangan web service yang baik menggabungkan exposure minimum, routing eksplisit, TLS yang dapat dipelihara, failure mode terukur, dan auditability.

### Implikasi Rekayasa dan Tata Kelola Perubahan

Konfigurasi reverse proxy sebaiknya diperlakukan sebagai kode. Setiap perubahan pada virtual host, route, header, sertifikat, dan timeout perlu melalui review, uji sintaks, test integrasi, serta pencatatan versi. Pemisahan konfigurasi dasar dan konfigurasi environment mengurangi duplikasi, tetapi nilai efektif tetap harus dapat direkonstruksi. Rollback bukan hanya mengembalikan berkas lama; kompatibilitas aplikasi, cache, sertifikat, dan koneksi aktif perlu diperhitungkan.

Dari perspektif DevSecOps, perubahan web layer menghubungkan tim aplikasi, platform, dan keamanan. Developer menjelaskan kebutuhan route dan header; platform menjaga deployment serta availability; keamanan menetapkan baseline TLS, exposure, dan logging. Matriks tanggung jawab mencegah asumsi bahwa pihak lain telah menerapkan kontrol. Praktikum harus mencatat risiko sebelum perubahan, evidence setelah perubahan, dan residual risk. Pola tersebut membiasakan mahasiswa melihat Nginx atau Apache bukan sebagai file konfigurasi terpisah, melainkan bagian dari rantai layanan yang memiliki lifecycle, owner, dan tujuan keamanan terukur.

Uji penerimaan juga perlu dijalankan ulang setelah perubahan dependency atau base image. Perilaku HTTP dapat berubah akibat versi modul, default cipher, atau konfigurasi bawaan, sehingga hasil lama tidak selalu mewakili artefak baru. Dokumentasi versi dan test regresi menjaga keputusan tetap dapat dipertanggungjawabkan.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan. Jalankan perintah cleanup setelah praktikum selesai, terutama jika port yang sama digunakan pada bab berikutnya.

```bash
# Struktur direktori umum Bab 4
mkdir -p ~/docker-lab/bab-4
cd ~/docker-lab/bab-4
# Simpan file compose, Dockerfile, konfigurasi, dan log di direktori ini.
```

## Langkah Praktikum Eksploratif

### Struktur project web service

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
mkdir -p ~/docker-lab/web-stack/{apache/sites,nginx/conf,certs,logs,app}
cd ~/docker-lab/web-stack
openssl req -x509 -nodes -days 365 -newkey rsa:2048   -keyout certs/lab.key -out certs/lab.crt   -subj "/CN=localhost/O=PENS Docker Lab"
```

### Nginx reverse proxy minimal

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
server {
  listen 80;
  server_name localhost;
  location / {
    proxy_pass http://apache-web:80;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
  }
  location /api/ {
    proxy_pass http://flask-app:5000/;
  }
}
```

### Compose web stack

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```yaml
services:
  proxy:
    image: nginx:alpine
    ports:
      - "8080:80"
      - "8443:443"
    volumes:
      - ./nginx/conf:/etc/nginx/conf.d:ro
      - ./certs:/etc/nginx/certs:ro
      - ./logs/nginx:/var/log/nginx
    networks: [web-net]
    depends_on: [apache-web, flask-app]
  apache-web:
    image: httpd:2.4-alpine
    volumes:
      - ./apache/sites:/usr/local/apache2/htdocs:ro
    networks: [web-net]
  flask-app:
    build: ./app
    networks: [web-net]
networks:
  web-net:
```

## Verifikasi dan Skenario Pengujian

[ ] Nginx reverse proxy dapat mengakses Apache melalui nama service.

[ ] Endpoint /api diarahkan ke backend Flask.

[ ] HTTPS self-signed berjalan di port 8443.

[ ] Access log tersimpan di direktori host.

[ ] Mahasiswa mampu menjelaskan alur request client-proxy-backend.

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

1. Mengapa reverse proxy tidak seharusnya menjalankan semua logic aplikasi?

2. Apa perbedaan TLS termination dan end-to-end TLS?

3. Bagaimana cara mengisolasi backend agar tidak langsung diakses dari host?

4. Apa konsekuensi menyimpan private key TLS di bind mount?

5. Bandingkan log Nginx dan log Apache dari sisi format dan kegunaan debugging.

## Format Laporan Praktikum

Laporan Bab 4 dikumpulkan dalam PDF maksimum 5 halaman. Isi laporan harus menunjukkan bukti eksekusi, analisis, dan refleksi keamanan/operasional.

Bukti minimum:

- Screenshot docker compose ps atau docker ps yang menunjukkan service berjalan.

- Screenshot hasil curl/browser/API/dashboard sesuai target bab.

- Cuplikan log atau query yang membuktikan sistem bekerja.

Analisis wajib:

- Jelaskan satu masalah yang muncul dan cara Anda mendiagnosisnya.

- Jelaskan risiko keamanan atau operasional yang relevan pada bab ini.

- Berikan rekomendasi perbaikan bila lab ini akan dibawa ke production-like environment.
