<a id="bab-08"></a>

# Bab 8 — Docker Security: Hardening, Secrets, Trivy, dan Private Registry

Topik utama: container security, non-root user, secrets, Trivy, private registry, resource limits.

## Tujuan Pembelajaran dan Kompetensi Utama

**Setelah menyelesaikan bab ini, pembaca diharapkan mampu:**

1. Mengidentifikasi attack surface umum Docker: root container, exposed socket, hardcoded secret, image rentan, dan resource abuse.

2. Menerapkan non-root user, read-only filesystem, drop capabilities, dan resource limits.

3. Memindai image dengan Trivy untuk menemukan CVE dan misconfiguration.

4. Membangun private registry untuk push/pull image internal.

5. Memahami posisi Docker Bench for Security sebagai audit baseline host.

## Konsep Inti dan Landasan Teori

Keamanan container harus dilihat sebagai defense in depth. Hardening Dockerfile tidak cukup bila runtime memberi privileged access, Docker socket dipasang ke container, atau registry menerima image tanpa scanning.

Menjalankan container sebagai non-root mengurangi dampak exploit aplikasi. Namun, non-root di dalam container bukan pengganti konfigurasi kernel security, capability drop, seccomp, AppArmor/SELinux, dan pembatasan filesystem.

Secret tidak boleh disimpan di Dockerfile, image layer, atau commit repository. Dalam lab Compose, file secret dapat digunakan sebagai pendekatan sederhana, tetapi untuk produksi gunakan Docker Swarm/Kubernetes secrets atau secret manager khusus.

| Risiko | Mitigasi utama |
| --- | --- |
| Container berjalan sebagai root | USER non-root, least privilege, user namespace |
| Secret di image atau env | Secrets file, vault, .gitignore, rotasi kredensial |
| Image rentan CVE | Trivy scan, base image minimal, patch berkala |
| Docker socket exposed | Hindari mount /var/run/docker.sock kecuali benar-benar perlu |
| Resource exhaustion | CPU/memory limits, monitoring, alerting |
| Registry tidak aman | TLS, authentication, authorization, scanning gate |

### Model Ancaman Container

Keamanan container mencakup image, registry, orchestrator atau engine, konfigurasi runtime, kernel host, network, storage, dan pipeline pembentuk artefak. NIST SP 800-190 mengelompokkan risiko pada image, registry, orchestrator, container, dan host; pendekatan ini menegaskan bahwa pemindaian image hanya menangani satu bagian attack surface [143]. Dokumentasi Docker juga menempatkan isolasi kernel, attack surface daemon, konfigurasi container, dan fitur hardening sebagai wilayah yang harus dianalisis [44].

Container berbagi kernel host. Namespace memisahkan pandangan resource dan cgroup membatasi konsumsi, tetapi konfigurasi seperti `--privileged`, host PID, host network, bind mount luas, atau pemasangan Docker socket mengurangi boundary secara sengaja. Docker socket memberi kemampuan mengendalikan engine dan pada praktiknya dapat berujung pada kendali host. Karena itu, akses tersebut harus dianggap privilege tinggi, bukan sekadar kebutuhan integrasi.

### Least Privilege dan Hardening Runtime

Menjalankan proses sebagai non-root mengurangi dampak eksploitasi, khususnya jika digabungkan dengan user namespace atau rootless mode [31,46]. Namun, UID non-zero saja tidak cukup. Linux capabilities perlu dikurangi, `no-new-privileges` mencegah proses memperoleh privilege baru, seccomp membatasi system call, dan AppArmor atau SELinux menerapkan policy akses. Root filesystem read-only mengurangi persistence, sementara tmpfs atau volume terbatas menyediakan lokasi tulis yang memang diperlukan.

Hardening harus berbasis kebutuhan aplikasi. Strategi praktis dimulai dengan menolak kemampuan berlebih, menjalankan test, lalu menambahkan hanya capability yang terbukti diperlukan. `CAP_SYS_ADMIN` sangat luas dan umumnya tidak layak diberikan. Limit CPU, memori, PID, dan file descriptor membatasi risiko denial-of-service dan fork bomb. Health check serta restart policy membantu availability, tetapi restart tanpa diagnosis dapat menyembunyikan crash loop.

### Keamanan Image dan Build

Image membawa dependency, konfigurasi, dan metadata. Base image kecil mengurangi jumlah paket, tetapi provenance dan pemeliharaan lebih penting daripada ukuran semata. Tag dapat berubah; digest mengidentifikasi content spesifik. Multi-stage build memisahkan compiler dan tool build dari runtime. `.dockerignore` mencegah source sensitif dan artefak lokal masuk ke build context. Secret build harus menggunakan secret mount sehingga tidak menjadi layer atau build argument yang terekspos [30,45].

Image scanning mencocokkan komponen dengan advisori vulnerability. Trivy dapat memindai paket OS, dependency bahasa, secret, dan misconfiguration bergantung pada mode [9,23]. Hasil HIGH atau CRITICAL tidak otomatis berarti risiko tertinggi pada konteks aplikasi. Triage perlu mempertimbangkan apakah komponen digunakan, apakah jalur rentan dapat dicapai, apakah exploit tersedia, dan apakah ada compensating control. Sebaliknya, hasil kosong bukan bukti tidak ada kelemahan karena database scanner, coverage, dan zero-day memiliki keterbatasan.

### Secret dan Credential Lifecycle

Secret mencakup password, token, private key, dan material autentikasi lain. Secret tidak boleh disimpan di repository, Dockerfile, image layer, atau log. Compose secret membantu memasang nilai sebagai file runtime [59], tetapi platform produksi biasanya memerlukan secret manager dengan kontrol akses, audit, rotasi, dan masa berlaku. Environment variable mudah digunakan namun dapat terekspos dalam inspect, dump, atau diagnostic output. Pilihan mekanisme harus mengikuti threat model.

Rotasi bukan sekadar mengganti string. Sistem harus menerbitkan credential baru, memperbarui consumer secara terkoordinasi, memverifikasi akses, mencabut nilai lama, dan memastikan cache atau koneksi lama tidak mempertahankan hak terlalu lama. Break-glass credential memerlukan penyimpanan, persetujuan, dan audit khusus. Praktikum harus menguji negative case: aplikasi tanpa secret gagal secara aman dan secret tidak muncul pada image history maupun log.

### Private Registry dan Kepercayaan Artefak

Registry menyimpan serta mendistribusikan image. Registry privat tidak otomatis terpercaya; akses perlu autentikasi, otorisasi repository, TLS, retensi, backup, dan audit. Push dan pull harus mengikuti identitas workload atau pengguna dengan scope minimum. Image yang dipromosikan antarenvironment sebaiknya tetap menggunakan digest yang sama agar artefak yang diuji identik dengan artefak yang dideploy.

Scanning di registry berguna untuk mendeteksi advisori baru setelah build. Namun, organisasi juga memerlukan provenance, SBOM, signature, dan policy penerimaan. Signature menunjukkan hubungan artefak dengan identitas penanda tangan; provenance menjelaskan bagaimana artefak dibangun. Kontrol ini melengkapi, bukan menggantikan, review source dan test runtime.

### Defense-in-Depth dan Evidence

OWASP Docker Security Cheat Sheet merekomendasikan pengurangan privilege, patching, resource limit, perlindungan daemon socket, dan konfigurasi aman [8]. Defense-in-depth berarti kontrol pada build, registry, deployment, dan runtime saling menutup celah. Misalnya image non-root masih memerlukan policy network; image bersih dari CVE tetap dapat memiliki autentikasi lemah; runtime read-only tetap dapat mengekstrak data melalui koneksi yang diizinkan.

Kriteria PASS harus mengukur konfigurasi efektif: UID proses, capabilities, seccomp, mount, published port, resource limit, digest, hasil scan, dan negative test. Evidence diperoleh dari Dockerfile, Compose hasil render, `docker inspect`, scan report, serta log eksekusi. Analisis akhir menyatakan residual risk dan exception yang memiliki owner serta batas waktu. Hardening yang profesional bukan kumpulan flag, melainkan keputusan risiko yang dapat diuji dan dipelihara.

### Baseline, Drift, dan Pemeliharaan Kontrol

Konfigurasi aman dapat mengalami drift ketika operator menambahkan port, mount, atau privilege untuk menyelesaikan masalah sementara. Baseline perlu dinyatakan sebagai policy-as-code dan dievaluasi pada konfigurasi efektif sebelum deployment. Runtime inventory kemudian dibandingkan dengan deklarasi untuk mendeteksi perbedaan. Exception yang sah memiliki owner, alasan, scope, serta tanggal kedaluwarsa.

Keamanan image juga berubah sepanjang waktu karena advisori baru diterbitkan setelah release. Re-scan berkala, rebuild dengan base image yang diperbarui, dan pengujian regresi diperlukan meskipun source aplikasi tidak berubah. Metric program seperti umur base image, waktu remediasi, jumlah exception kedaluwarsa, dan persentase workload non-root lebih informatif daripada jumlah temuan mentah. Dengan cara ini, hardening menjadi proses pemeliharaan yang terukur, bukan checklist satu kali sebelum rilis.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan. Jalankan perintah cleanup setelah praktikum selesai, terutama jika port yang sama digunakan pada bab berikutnya.

```bash
# Struktur direktori umum Bab 8
mkdir -p ~/docker-lab/bab-8
cd ~/docker-lab/bab-8
# Simpan file compose, Dockerfile, konfigurasi, dan log di direktori ini.
```

## Langkah Praktikum Eksploratif

### Dockerfile non-root

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```dockerfile
FROM python:3.11-slim
WORKDIR /app
RUN addgroup --system app && adduser --system --ingroup app app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY app.py .
USER app
EXPOSE 5000
CMD ["python", "app.py"]
```

### Compose hardening runtime

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```yaml
services:
  app:
    build: ./app
    read_only: true
    tmpfs:
      - /tmp:size=64m
    cap_drop:
      - ALL
    security_opt:
      - no-new-privileges:true
    deploy:
      resources:
        limits:
          cpus: "0.50"
          memory: 256M
```

### Trivy image scan

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
# Scan image lokal
trivy image pens-web:1.0
# Gagal-kan pipeline bila ada vulnerability HIGH/CRITICAL
trivy image --severity HIGH,CRITICAL --exit-code 1 pens-web:1.0
```

### Private registry lab

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
docker run -d --name registry -p 5000:5000 registry:2
docker tag pens-web:1.0 localhost:5000/pens-web:1.0
docker push localhost:5000/pens-web:1.0
docker pull localhost:5000/pens-web:1.0
```

## Verifikasi dan Skenario Pengujian

[ ] Image aplikasi berjalan sebagai user non-root.

[ ] Trivy scan dijalankan dan hasilnya dianalisis, bukan hanya dilampirkan.

[ ] Container memiliki batas CPU/memory dan filesystem read-only bila memungkinkan.

[ ] Private registry dapat menerima push dan melayani pull image.

[ ] Mahasiswa dapat menjelaskan bahaya mount Docker socket.

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

1. Mengapa root di container tetap berbahaya meskipun bukan selalu root host?

2. Apa kelemahan menyimpan secret pada environment variable?

3. Bagaimana menentukan apakah CVE harus segera diperbaiki atau dapat ditoleransi sementara?

4. Apa perbedaan registry private biasa dan registry dengan policy scanning gate?

5. Mengapa Docker socket setara dengan akses administratif ke Docker host?

## Format Laporan Praktikum

Laporan Bab 8 dikumpulkan dalam PDF maksimum 5 halaman. Isi laporan harus menunjukkan bukti eksekusi, analisis, dan refleksi keamanan/operasional.

Bukti minimum:

- Screenshot docker compose ps atau docker ps yang menunjukkan service berjalan.

- Screenshot hasil curl/browser/API/dashboard sesuai target bab.

- Cuplikan log atau query yang membuktikan sistem bekerja.

Analisis wajib:

- Jelaskan satu masalah yang muncul dan cara Anda mendiagnosisnya.

- Jelaskan risiko keamanan atau operasional yang relevan pada bab ini.

- Berikan rekomendasi perbaikan bila lab ini akan dibawa ke production-like environment.
