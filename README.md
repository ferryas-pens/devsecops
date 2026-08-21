# Materi Ajar — DevOps (DevSecOps Terintegrasi)

Program Studi D4 Teknik Informatika — Politeknik Elektronika Negeri Surabaya (PENS).
Materi praktikum bersumber dari buku ajar **"Buku Tutorial DevSecOps Praktis Berbasis Docker" (Bab 1–17)**, dipecah menjadi satu berkas Markdown per bab. Isi tiap bab identik dengan naskah asli; berkas ini hanya memisahkan per bab dan menautkannya.

## Identitas Mata Kuliah

| Butir | Nilai |
| --- | --- |
| Nama mata kuliah | DevOps |
| Bobot | 3 SKS (T = 1, P = 2) — *asumsi, sesuaikan dengan kurikulum prodi* |
| Semester | 6 — *asumsi, sesuaikan* |
| Rumpun | Keilmuan Inti (Rekayasa Perangkat Lunak) — *asumsi, sesuaikan* |
| Struktur | 16 minggu; UTS pada Minggu 8; UAS pada Minggu 16 |

> Catatan lingkup: mata kuliah diminta bernama "DevOps", tetapi materi sumber adalah DevSecOps. Materi ini disusun sebagai **DevOps dengan keamanan terintegrasi (shift-left)**, sehingga secara praktik setara dengan DevSecOps. Jika dikehendaki DevOps murni, bab keamanan mendalam (Bab 11–16) dapat dipangkas.

## Deskripsi Singkat

Mata kuliah DevOps membekali mahasiswa dengan kemampuan merancang, membangun, dan mengoperasikan alur pengiriman perangkat lunak (software delivery) yang otomatis, andal, teramati, dan aman menggunakan pendekatan kontainerisasi berbasis Docker. Perkuliahan diawali dengan fondasi budaya dan proses DevOps/DevSecOps sebagai sistem sosio-teknis serta evolusi SDLC; dilanjutkan dengan kontainerisasi aplikasi dan layanan (web, basis data), jaringan dan penyimpanan container, orkestrasi dengan Compose, observability (centralized logging dan monitoring metrics), keamanan container (hardening, secret, image scanning) dan Identity & Access Management; kemudian pembangunan pipeline CI/CD sebagai code beserta secure build, signing, SBOM, dan provenance; serta praktik keamanan shift-left (threat modeling, SAST, SCA/SBOM) dan shift-right (DAST, policy-as-code, runtime security, respons insiden). Pembelajaran berbasis praktikum dan proyek (project-based), berpuncak pada penyusunan platform web terkontainerisasi yang aman dan teramati yang dievaluasi pada UAS.

## Mata Kuliah Prasyarat

Disarankan: Sistem Operasi / Praktikum Sistem Operasi; Konsep Jaringan; Metodologi Agile / Pengembangan Perangkat Lunak. *(sesuaikan dengan struktur kurikulum prodi)*

## Daftar Bab

Kolom "Minggu" menunjukkan pemetaan ke RPS. Minggu 8 (UTS) dan Minggu 16 (UAS) tidak memuat bab baru. Urutan pengajaran Bab 11–14 sengaja ditata ulang dari urutan buku (Bab 14 diajarkan lebih dulu pada Minggu 12, lalu Bab 11–13 pada Minggu 13); lihat catatan di bawah tabel.

| Bab | Judul | Minggu |
| --- | --- | --- |
| 1 | [Fondasi Teoretis dan Kerangka Kerja DevSecOps](bab-01.md) | 1 |
| 2 | [Konsep Container dan Instalasi Docker](bab-02.md) | 2 |
| 3 | [Docker Network, Volume, Bind Mount, tmpfs, dan Compose](bab-03.md) | 3 |
| 4 | [Web Service Container: Apache, Nginx, Reverse Proxy, dan TLS](bab-04.md) | 4 |
| 5 | [Database Service di Docker: PostgreSQL](bab-05.md) | 5 |
| 6 | [Centralized Logging dengan Fluent Bit dan PostgreSQL](bab-06.md) | 6 |
| 7 | [Monitoring Resource dengan Prometheus, cAdvisor, Node Exporter, dan Grafana](bab-07.md) | 7 |
| 8 | [Docker Security: Hardening, Secrets, Trivy, dan Private Registry](bab-08.md) | 9 |
| 9 | [Identity and Access Management dengan Keycloak di Docker](bab-09.md) | 10 |
| 10 | [CI/CD Pipeline dengan Docker: Gitea, Drone CI, Registry, dan Deploy](bab-10.md) | 11 |
| 11 | [Security Requirements dan Threat Modeling](bab-11.md) | 13 |
| 12 | [Secure Coding, SAST, dan Secret Scanning](bab-12.md) | 13 |
| 13 | [SCA, SBOM, dan Tata Kelola Dependensi](bab-13.md) | 13 |
| 14 | [Secure Build, Image Scanning, Signing, dan Provenance](bab-14.md) | 12 |
| 15 | [DAST, API Security Testing, dan Security Regression](bab-15.md) | 14 |
| 16 | [Policy-as-Code, Runtime Security, dan Respons Insiden](bab-16.md) | 14 |
| 17 | [Studi Kasus Terpadu: Platform Web Terkontainerisasi yang Aman dan Teramati](bab-17.md) | 15 |

Catatan beban: Minggu 13 memadatkan tiga bab (11–13) dan Minggu 14 memadatkan dua bab (15–16). Pertimbangkan pembagian tugas/asesmen agar beban minggu tersebut proporsional.

## Struktur 16 Minggu (ringkas)

- Minggu 1–7 — fondasi, kontainerisasi, layanan, observability (Bab 1–7).
- Minggu 8 — UTS: produk stack multi-container yang teramati + demo (sintesis Minggu 1–7).
- Minggu 9–11 — keamanan container, IAM, CI/CD (Bab 8–10).
- Minggu 12–14 — secure build/signing, keamanan shift-left, DAST/policy/runtime (Bab 14, 11–13, 15–16).
- Minggu 15 — proyek terpadu (Bab 17).
- Minggu 16 — UAS: proyek platform terpadu (dokumen + demo + presentasi).

## Cara Pakai dan Catatan Aset

- Berkas bab memakai referensi gambar relatif `assets/gambar-XX.png`. **Pertahankan seluruh berkas dalam satu folder bersama folder `assets/`.** Jika berkas `.md` dipindah keluar dari folder ini, tautan gambar akan putus.
- Total 19 gambar berada di `assets/`.
- Untuk pemakaian di LMS/GitHub, unggah folder ini secara utuh (termasuk `assets/`).

## Sumber dan Verifikasi

- Sumber materi: buku ajar "Buku Tutorial DevSecOps Praktis Berbasis Docker" (Bab 1–17). *Lengkapi identitas penulis, tahun, penerbit/repositori resmi.*
- Identitas mata kuliah yang ditandai *asumsi* (SKS, semester, rumpun) perlu disesuaikan dengan Kurikulum prodi yang ditetapkan.
- Dokumen RPS terpisah (`RPS_DevOps.docx`) memuat CPL/CPMK/Sub-CPMK, matriks OBE, rencana mingguan, sistem penilaian, dan rubrik.
