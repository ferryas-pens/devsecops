<a id="bab-09"></a>

# Bab 9 — Identity and Access Management dengan Keycloak di Docker

Topik utama: Keycloak, IAM, OIDC, OAuth2, JWT, SSO, RBAC.

## Tujuan Pembelajaran dan Kompetensi Utama

**Setelah menyelesaikan bab ini, pembaca diharapkan mampu:**

1. Membedakan autentikasi dan otorisasi dalam konteks aplikasi web modern.

2. Menjalankan Keycloak dengan PostgreSQL backend di Docker.

3. Membuat realm, client, user, role, dan group pada Keycloak.

4. Menjelaskan Authorization Code Flow dan struktur JWT.

5. Mengintegrasikan aplikasi Flask dengan login OIDC dan RBAC sederhana.

## Konsep Inti dan Landasan Teori

Identity and Access Management memusatkan proses login, session, token, dan kebijakan akses. Tanpa IAM, setiap aplikasi cenderung membuat autentikasi sendiri-sendiri, sehingga audit dan kontrol akses menjadi sulit.

OpenID Connect adalah lapisan identity di atas OAuth 2.0. Dalam flow authorization code, aplikasi tidak menerima password user; aplikasi menerima authorization code yang kemudian ditukar dengan token dari identity provider.

JWT membawa claims seperti subject, issuer, audience, expiration, dan role. Token harus divalidasi secara ketat: signature, issuer, audience, expiration, dan algoritma tidak boleh diabaikan.

| Konsep | Deskripsi |
| --- | --- |
| Realm | Tenant terisolasi yang berisi konfigurasi user, role, client, dan policy |
| Client | Aplikasi yang mendelegasikan login ke Keycloak |
| User | Identitas yang melakukan autentikasi |
| Role | Hak akses yang diberikan ke user atau group |
| Access token | Token untuk mengakses resource/API |
| Refresh token | Token untuk memperoleh access token baru sesuai kebijakan session |

### Identitas, Autentikasi, dan Otorisasi

Identity and Access Management mengatur bagaimana identitas dibentuk, dibuktikan, diberi hak, dan dihentikan. Autentikasi menjawab siapa principal, sedangkan otorisasi menentukan tindakan apa yang diperbolehkan pada resource tertentu. Keycloak menyediakan identity provider yang mendukung realm, client, user, group, role, session, dan federasi [10]. Menjalankan Keycloak dalam container mempermudah laboratorium, tetapi model keamanannya tetap harus dirancang; credential admin default dan mode development tidak sesuai untuk produksi.

Realm merupakan boundary administratif logis. Client merepresentasikan aplikasi atau service yang mempercayai identity provider. User dan service account memiliki atribut serta role. Group memudahkan pengelolaan role berbasis keanggotaan. Desain sebaiknya memisahkan role bisnis dari detail endpoint. Role `dosen`, `mahasiswa`, atau `auditor` lebih bermakna daripada role yang menyalin nama route, selama policy pada aplikasi tetap eksplisit dan least privilege.

### OAuth 2.0 dan OpenID Connect

OAuth 2.0 adalah kerangka delegasi otorisasi, sedangkan OpenID Connect menambahkan lapisan identitas di atas OAuth. Access token digunakan untuk mengakses resource server; ID token menyampaikan informasi autentikasi kepada client dan tidak seharusnya digunakan sebagai pengganti access token pada API. Refresh token memperoleh access token baru dan memiliki sensitivitas tinggi. Pemilihan flow bergantung pada jenis client. Authorization Code dengan PKCE lazim untuk aplikasi interaktif, sedangkan Client Credentials digunakan untuk komunikasi machine-to-machine tanpa pengguna.

Redirect URI harus dibatasi secara tepat karena URI longgar dapat mengalihkan authorization code kepada pihak yang tidak sah. Public client tidak dapat menjaga client secret seperti server backend. Confidential client memerlukan penyimpanan secret yang benar. Scope menyatakan izin yang diminta, tetapi keputusan akhir tetap berada pada authorization server dan resource server. Consent, session lifetime, refresh rotation, dan logout memengaruhi pengalaman sekaligus risiko.

### Token dan Validasi Kriptografis

JSON Web Token lazim digunakan sebagai access token yang ditandatangani. Aplikasi harus memverifikasi signature menggunakan key yang diperoleh dari endpoint JWKS, kemudian memeriksa issuer (`iss`), audience (`aud`), masa berlaku (`exp`), waktu aktif, dan claim yang diperlukan. Hanya memecah payload base64 tanpa verifikasi signature adalah kesalahan kritis. Algoritme yang diterima harus dibatasi; aplikasi tidak boleh mempercayai algoritme dari token tanpa policy lokal.

Key rotation menuntut aplikasi menangani beberapa key ID selama masa transisi serta melakukan cache JWKS secara aman. Cache terlalu lama menunda penerimaan key baru, sedangkan fetch pada setiap request menciptakan dependency dan risiko availability. Clock skew kecil dapat ditoleransi, tetapi sinkronisasi waktu tetap diperlukan. Token jangan dicatat secara utuh pada log karena bearer token dapat digunakan oleh siapa pun yang memilikinya sampai kedaluwarsa atau dicabut.

### Otorisasi pada Resource Server

Identity provider menerbitkan claim, tetapi resource server bertanggung jawab menerapkan otorisasi pada setiap request. Pemeriksaan role di antarmuka pengguna tidak cukup karena client dapat dimodifikasi. Policy perlu mempertimbangkan subject, action, resource, tenant, dan context. Role-based access control mudah dipahami, sedangkan attribute-based control lebih fleksibel namun membutuhkan tata kelola atribut. Untuk object-level authorization, aplikasi harus memastikan pengguna memang berhak pada objek yang diminta, bukan hanya memiliki role umum.

Default deny mengharuskan akses ditolak ketika policy tidak cocok, token tidak valid, atau dependency identitas gagal. Respons 401 digunakan ketika autentikasi tidak tersedia atau tidak valid; 403 menunjukkan identitas dikenali tetapi tidak berwenang. Pesan error tidak boleh membocorkan detail berlebihan. Negative test wajib mencakup token kedaluwarsa, issuer salah, audience salah, signature rusak, role tidak cukup, dan akses lintas pengguna atau tenant.

### Lifecycle Identitas dan Administrasi

IAM tidak berhenti pada login. Proses joiner, mover, dan leaver memastikan akun dibuat berdasarkan otoritas, hak diubah ketika tanggung jawab berubah, dan akses dicabut segera ketika hubungan berakhir. Akun tidak aktif, service account tanpa owner, dan role yang terus bertambah menciptakan privilege creep. Review akses berkala, separation of duties, serta pencatatan tindakan admin diperlukan untuk menjaga akuntabilitas.

Multi-factor authentication mengurangi risiko password dicuri, terutama untuk administrator. Password policy harus mendukung panjang, proteksi brute force, dan pemulihan akun yang aman. Recovery sering menjadi jalur yang lebih lemah daripada login. Admin console sebaiknya tidak dipublikasikan tanpa pembatasan network, TLS, dan autentikasi kuat. Database Keycloak memerlukan backup serta restore test karena konfigurasi realm dan identitas merupakan data kritis.

### Deployment dan Evidence

Pada Compose, Keycloak dan PostgreSQL ditempatkan pada network internal; hanya endpoint yang diperlukan aplikasi dipublikasikan melalui TLS. Hostname, proxy mode, redirect URI, dan origin harus konsisten agar token tidak diterbitkan untuk alamat yang salah. Secret client dan password database dipasang saat runtime, bukan dimasukkan ke image. Health check perlu menilai readiness, sementara log audit harus dikirim ke sistem terpusat tanpa membocorkan token.

Evidence praktikum mencakup metadata realm dan client yang telah disanitasi, claim token tanpa data sensitif, hasil validasi signature dan claim, serta matriks role-endpoint dengan PASS/FAIL. OWASP ASVS menekankan kebutuhan verifikasi autentikasi, session, dan access control pada aplikasi [79]. Dengan demikian, keberhasilan IAM dibuktikan bukan oleh tampilnya halaman login, melainkan oleh konsistensi lifecycle identitas, validasi token, dan enforcement pada resource server.

### Assurance dan Batas Kepercayaan IAM

IAM merupakan komponen kritis, tetapi aplikasi tidak boleh menyerahkan seluruh keputusan keamanan kepadanya. Keycloak membuktikan identitas dan menerbitkan claim; resource server tetap memvalidasi token serta menerapkan policy terhadap objek bisnis. Kegagalan identity provider juga harus dirancang: endpoint terlindungi gagal secara tertutup, sementara health dan operasi pemulihan tertentu dapat mengikuti jalur yang terkontrol.

Perubahan mapper, role, key, atau redirect URI diperlakukan sebagai perubahan keamanan dan diuji sebelum promosi. Audit event login, kegagalan, perubahan hak, dan tindakan admin dikorelasikan dengan log aplikasi. Kriteria keberhasilan mencakup akses positif serta penolakan sistematis pada token dan role yang salah. Assurance semacam ini mengurangi kesenjangan antara konfigurasi identity provider dan perilaku aktual aplikasi, sekaligus menghasilkan evidence yang dapat direview oleh tim pengembang, platform, dan keamanan.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Gunakan satu direktori kerja per bab agar file konfigurasi, volume bind mount, dan laporan mudah dipisahkan. Jalankan perintah cleanup setelah praktikum selesai, terutama jika port yang sama digunakan pada bab berikutnya.

```bash
# Struktur direktori umum Bab 9
mkdir -p ~/docker-lab/bab-9
cd ~/docker-lab/bab-9
# Simpan file compose, Dockerfile, konfigurasi, dan log di direktori ini.
```

## Langkah Praktikum Eksploratif

### Compose Keycloak dan PostgreSQL

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```yaml
services:
  keycloak-db:
    image: postgres:16-alpine
    environment:
      POSTGRES_DB: keycloak
      POSTGRES_USER: keycloak
      POSTGRES_PASSWORD: keycloakpass
    volumes:
      - keycloak-db-data:/var/lib/postgresql/data
  keycloak:
    image: quay.io/keycloak/keycloak:latest
    command: start-dev
    environment:
      KC_DB: postgres
      KC_DB_URL_HOST: keycloak-db
      KC_DB_URL_DATABASE: keycloak
      KC_DB_USERNAME: keycloak
      KC_DB_PASSWORD: keycloakpass
      KEYCLOAK_ADMIN: admin
      KEYCLOAK_ADMIN_PASSWORD: admin123
    ports:
      - "8080:8080"
    depends_on: [keycloak-db]
volumes:
  keycloak-db-data:
```

### Konfigurasi minimum realm

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
1. Buka http://localhost:8080 dan login sebagai admin.
2. Buat realm: pens-lab.
3. Buat client OpenID Connect: flask-app.
4. Set redirect URI: http://localhost:5000/callback.
5. Buat user: mahasiswa1 dengan password sementara dimatikan.
6. Buat role: viewer dan admin.
7. Assign role viewer ke mahasiswa1.
```

### Validasi token di aplikasi

Jalankan langkah berikut secara berurutan, lalu verifikasi hasil sebelum melanjutkan ke sub-langkah berikutnya.

```bash
# Prinsip validasi token:
# - ambil public key dari JWKS endpoint Keycloak
# - verifikasi signature
# - cek iss, aud, exp
# - ambil role dari realm_access.roles atau resource_access
# - tolak request bila role tidak sesuai kebijakan endpoint
```

## Verifikasi dan Skenario Pengujian

[ ] Keycloak dan PostgreSQL berjalan dengan volume persisten.

[ ] Realm, client, user, dan role berhasil dibuat.

[ ] Aplikasi dapat redirect ke Keycloak dan menerima callback.

[ ] JWT berhasil divalidasi dan role dapat dibaca.

[ ] Endpoint admin ditolak untuk user tanpa role admin.

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

1. Apa perbedaan autentikasi dan otorisasi?

2. Mengapa Authorization Code Flow lebih aman daripada meminta password langsung ke aplikasi?

3. Apa yang harus dicek ketika memvalidasi JWT?

4. Bagaimana SSO mengurangi beban user tetapi juga menambah risiko konsentrasi identitas?

5. Apa strategi logout yang benar pada aplikasi berbasis token?

## Format Laporan Praktikum

Laporan Bab 9 dikumpulkan dalam PDF maksimum 5 halaman. Isi laporan harus menunjukkan bukti eksekusi, analisis, dan refleksi keamanan/operasional.

Bukti minimum:

- Screenshot docker compose ps atau docker ps yang menunjukkan service berjalan.

- Screenshot hasil curl/browser/API/dashboard sesuai target bab.

- Cuplikan log atau query yang membuktikan sistem bekerja.

Analisis wajib:

- Jelaskan satu masalah yang muncul dan cara Anda mendiagnosisnya.

- Jelaskan risiko keamanan atau operasional yang relevan pada bab ini.

- Berikan rekomendasi perbaikan bila lab ini akan dibawa ke production-like environment.
