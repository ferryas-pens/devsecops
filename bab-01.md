<a id="bab-01"></a>

# Bab 1 — Fondasi Teoretis dan Kerangka Kerja DevSecOps

Topik utama: evolusi DevOps menuju DevSecOps, tanggung jawab bersama, shift-left dan shift-right, manajemen risiko, secure software development framework, software supply chain, security gate, bukti jaminan, serta pengukuran kematangan.

## Tujuan Pembelajaran dan Kompetensi Utama

- Menjelaskan DevSecOps sebagai sistem sosio-teknis, bukan sekadar penambahan alat pemindai ke pipeline.

- Memetakan praktik DevSecOps ke kelompok praktik NIST SSDF, panduan OWASP, dan jaminan rantai pasok SLSA.

- Membedakan shift-left, shift-right, security gate, waiver berbasis risiko, serta evidence yang dapat diaudit.

- Menyusun baseline laboratorium dan kriteria keberhasilan eksperimen yang aman, legal, dan dapat direplikasi.

## Konsep Inti dan Landasan Teori

### Transformasi Digital sebagai Konteks DevOps

Transformasi digital tidak tepat dipahami hanya sebagai kegiatan memindahkan proses manual ke aplikasi. Transformasi tersebut merupakan perubahan cara organisasi menciptakan nilai dengan memanfaatkan data, aplikasi, dan infrastruktur sebagai satu sistem. Materi DevOps Concept-Day1 menempatkan business architecture, information architecture, application architecture, dan technology architecture dalam satu kesinambungan [35]. Logika ini penting karena keputusan teknologi tidak berdiri sendiri: rancangan aplikasi harus mendukung proses bisnis, informasi harus tersedia dalam kualitas yang memadai, dan platform harus mampu menjalankan perubahan secara konsisten. Dengan demikian, DevOps lahir bukan karena organisasi ingin menggunakan alat baru, melainkan karena jarak antara kebutuhan bisnis dan kemampuan TI harus diperkecil.

Pada lingkungan digital, kebutuhan pelanggan, pola kompetisi, regulasi, dan teknologi berubah dengan kecepatan yang sulit diprediksi. Business agility kemudian menjadi kemampuan organisasi untuk mendeteksi perubahan, menentukan prioritas, menguji respons, dan mengalirkan hasilnya kepada pengguna dalam waktu yang relevan. Agility tidak identik dengan bekerja tergesa-gesa. Agility mensyaratkan siklus belajar yang pendek, data umpan balik yang dapat dipercaya, serta kemampuan membatalkan atau memperbaiki keputusan tanpa biaya yang tidak proporsional. Apabila proses pengembangan tetap bergantung pada serah-terima manual, lingkungan yang tidak konsisten, dan persetujuan yang tidak memiliki kriteria, perubahan bisnis yang cepat justru meningkatkan antrean dan risiko operasional.

Konsekuensinya, organisasi modern semakin menyerupai perusahaan teknologi, sekalipun produk utamanya berada pada sektor keuangan, kesehatan, pendidikan, manufaktur, atau layanan publik [35]. Perangkat lunak menjadi medium utama interaksi dengan pengguna dan menjadi bagian dari proses inti organisasi. Gangguan layanan, kebocoran data, atau kegagalan pembaruan tidak lagi hanya merupakan masalah teknis; kejadian tersebut dapat memengaruhi pendapatan, kepatuhan, kepercayaan, dan keselamatan. Dasar teori DevSecOps karena itu harus dimulai dari keterkaitan antara nilai bisnis, aliran perubahan perangkat lunak, dan risiko yang menyertai perubahan tersebut.

### Evolusi SDLC: dari Waterfall menuju Agile dan DevOps

Model waterfall mengorganisasikan pengembangan ke dalam tahap yang relatif berurutan, misalnya analisis, desain, implementasi, pengujian, dan rilis. Model ini memberi struktur dokumentasi dan titik persetujuan yang jelas, tetapi umpan balik dari tahap akhir dapat tiba ketika asumsi awal sudah mahal untuk diubah. Pada sistem dengan kebutuhan stabil dan tuntutan kepatuhan yang tinggi, unsur perencanaan berurutan tetap dapat berguna. Permasalahan muncul ketika organisasi memperlakukan seluruh perubahan seolah-olah dapat diprediksi sejak awal, padahal kebutuhan digital bersifat dinamis dan teknologi berubah selama proyek berlangsung.

Agile memperpendek horizon perencanaan melalui iterasi, backlog yang diprioritaskan, kolaborasi lintas fungsi, dan evaluasi inkremental. Scrum, misalnya, mengatur pekerjaan ke dalam sprint dengan perencanaan, daily scrum, review, dan retrospective. Nilai teoritis dari iterasi bukan sekadar membagi proyek besar menjadi bagian kecil, tetapi mempercepat pembelajaran. Setiap increment menyediakan kesempatan untuk memvalidasi kebutuhan, kualitas, dan arah produk. Namun, Agile pada sisi pengembangan belum otomatis menyelesaikan persoalan rilis apabila operasi masih menerima paket secara periodik, konfigurasi dilakukan manual, atau lingkungan uji berbeda dari produksi.

DevOps memperluas prinsip iteratif ke seluruh aliran delivery dengan mempertemukan development dan operations. Dalam materi sumber, perbedaan tersebut digambarkan sebagai perpindahan dari proses berbulan-bulan menuju beberapa siklus yang lebih pendek, didukung continuous integration dan continuous delivery [35]. Continuous integration mengintegrasikan perubahan kecil secara sering dan memverifikasinya melalui build serta pengujian otomatis. Continuous delivery menjaga perangkat lunak berada dalam kondisi siap dirilis, sedangkan continuous deployment menerapkan perubahan yang telah memenuhi kebijakan secara otomatis ke lingkungan sasaran. Ketiganya perlu dibedakan agar organisasi tidak mengklaim otomatisasi yang lebih luas daripada praktik sebenarnya.

Pengurangan ukuran batch merupakan mekanisme utama yang menjelaskan mengapa DevOps dapat meningkatkan kecepatan sekaligus stabilitas. Perubahan kecil lebih mudah ditinjau, diuji, dilacak, dan dikembalikan dibandingkan rilis besar yang menggabungkan banyak asumsi. Otomasi mengurangi variasi pelaksanaan, tetapi manfaatnya bergantung pada kualitas proses yang diotomasi. Proses manual yang tidak jelas, apabila langsung diubah menjadi skrip, hanya menghasilkan ketidakjelasan dengan kecepatan lebih tinggi. Karena itu, standardisasi, version control, kriteria penerimaan, dan observability harus tumbuh bersama otomasi.

![Diagram alur dari tekanan bisnis dan agility menuju DevOps, platform container-cloud, dan DevSecOps dengan umpan balik berkelanjutan.](assets/gambar-01.png)

*Gambar 2. Alur logis transformasi digital menuju DevSecOps.*

> Sumber: diadaptasi dan digambar ulang oleh penulis berdasarkan DevOps Concept-Day1 [35], khususnya hlm. 7-18, 31, dan 48-50.

### Perbandingan Evolusi Pendekatan Delivery

| Pendekatan | Unit Perubahan | Umpan Balik | Otomasi Dominan | Risiko Utama |
| --- | --- | --- | --- | --- |
| Waterfall | Tahap atau rilis besar | Cenderung terlambat | Terbatas per tahap | Asumsi awal mahal dikoreksi |
| Agile | Increment per sprint | Review setiap iterasi | Build/test dapat parsial | Rilis tetap menjadi antrean operasi |
| DevOps | Perubahan kecil dan sering | Pipeline dan telemetry | CI/CD, IaC, observability | Kecepatan menyebarkan salah konfigurasi |
| DevSecOps | Perubahan kecil dengan evidence | Risiko sejak desain hingga runtime | Kontrol keamanan sebagai kode | Gate buruk memicu bypass atau bottleneck |

> Sumber: sintesis penulis berdasarkan alur SDLC, Agile, DevOps, dan DevSecOps pada DevOps Concept-Day1 [35].

### DevOps sebagai Sistem Sosio-Teknis

DevOps sering dipersempit menjadi pipeline CI/CD, padahal materi sumber menekankan perubahan budaya TI, praktik agile dan lean, orientasi sistem, serta kolaborasi antara development dan operations [35]. Secara teoretis, DevOps merupakan sistem sosio-teknis: hasil delivery dibentuk oleh interaksi manusia, proses, arsitektur, alat, dan kebijakan. Tool yang baik tidak dapat mengompensasi tujuan yang saling bertentangan. Apabila pengembang dinilai hanya berdasarkan jumlah fitur, sementara operator dinilai hanya berdasarkan ketiadaan perubahan, organisasi akan menciptakan konflik struktural. Tujuan bersama perlu dirumuskan melalui indikator yang menggabungkan kecepatan, reliabilitas, keamanan, dan nilai pengguna.

Siklus DevOps lazim direpresentasikan sebagai plan, code, build, test, release, deploy, operate, dan monitor. Bentuk tak berujung menegaskan bahwa produksi bukan terminal akhir, melainkan sumber pengetahuan untuk perencanaan berikutnya. Data build, hasil pengujian, catatan deployment, metrics, logs, traces, dan laporan insiden membentuk rantai bukti tentang keadaan sistem. Umpan balik harus cukup cepat untuk memengaruhi keputusan yang sedang dibuat dan cukup bermutu untuk membedakan gejala dari penyebab. Oleh sebab itu, observability bukan aksesori operasional, tetapi mekanisme pembelajaran organisasi.

Continuous testing juga tidak berarti menjalankan semua pengujian pada setiap commit. Strategi yang efisien menyusun lapisan pemeriksaan berdasarkan biaya dan informasi yang dihasilkan. Linting, unit test, secret scanning, dan pemeriksaan kebijakan yang cepat dijalankan lebih awal; integration test dan analisis dependensi dijalankan setelah build; pengujian yang memerlukan lingkungan, seperti DAST, ditempatkan pada staging yang terisolasi. Hasil setiap lapisan menjadi evidence untuk keputusan berikutnya. Pendekatan ini mempertahankan kecepatan umpan balik tanpa mengorbankan pemeriksaan mendalam.

### Arsitektur dan Platform sebagai Enabler

Perubahan proses delivery berkaitan erat dengan evolusi arsitektur aplikasi dan platform. Aplikasi monolitik pada infrastruktur statis dapat memiliki siklus rilis yang panjang karena setiap perubahan membawa cakupan pengujian dan koordinasi yang besar. Microservices memungkinkan komponen dikembangkan dan dirilis lebih independen, tetapi menambah kompleksitas komunikasi, data terdistribusi, observability, identitas layanan, dan pengelolaan kegagalan. Dengan demikian, microservices bukan prasyarat DevOps dan bukan solusi universal. Arsitektur harus dipilih berdasarkan kebutuhan domain, skala tim, dan kemampuan operasi.

Container memberikan unit packaging yang memuat aplikasi dan dependensinya sehingga artefak dapat dipindahkan secara lebih konsisten antarlingkungan. Sifatnya yang lebih ringan daripada mesin virtual membantu pembuatan lingkungan sementara, pengujian paralel, dan deployment yang cepat. Container juga cocok dengan CI/CD karena image dapat dibangun sekali, diberi identitas digest, disimpan dalam registry, dan dipromosikan melalui lingkungan. Namun, portabilitas tidak menghapus kebutuhan konfigurasi eksternal, secret management, storage persisten, jaringan, dan kontrol runtime. Konsistensi paket harus dibedakan dari kesetaraan penuh lingkungan.

Platform DevOps mengintegrasikan source repository, CI/CD engine, artifact repository, runtime, keamanan, dan observability. Integrasi tersebut menciptakan rantai kendali: commit memicu build; build menghasilkan artefak; artefak diuji dan dipindai; registry menyimpan versi; orchestrator melakukan deployment; telemetry memberi umpan balik. Keuntungan terbesar platform bukan jumlah komponennya, melainkan tersedianya golden path yang dapat diulang oleh banyak tim. Pada saat yang sama, platform menjadi aset bernilai tinggi. Kredensial runner, hak menulis ke registry, plugin pipeline, dan konfigurasi deployment harus dilindungi karena kompromi pada jalur ini dapat memengaruhi seluruh produk.

### Dari DevOps menuju DevSecOps

Kecepatan DevOps memperbesar kebutuhan integrasi keamanan. Jika pipeline dapat merilis puluhan perubahan dalam sehari, audit manual pada akhir proyek tidak mampu mengikuti laju tersebut. DevSecOps membawa kontrol ke dalam lifecycle yang sama: threat modeling pada perencanaan, secure coding dan pemeriksaan secret pada tahap code, SAST dan SCA pada build, pengujian keamanan pada test, verifikasi SBOM serta signature pada release, policy check pada deploy, dan deteksi runtime pada operate-monitor. Keamanan tidak ditempelkan sebagai tahap kesembilan, tetapi hadir sebagai kriteria pada setiap tahap.

Integrasi tersebut tidak menghilangkan independensi fungsi keamanan. Pemisahan tugas, review berbasis risiko, dan otorisasi tetap diperlukan untuk perubahan sensitif. Yang berubah adalah cara kontrol dioperasionalkan: aturan dijelaskan secara eksplisit, diterapkan melalui automation yang dapat ditinjau, dan menghasilkan evidence yang dapat diaudit. Tim keamanan beralih dari pemeriksa manual untuk setiap perubahan menjadi perancang guardrail, penyedia intelijen, fasilitator threat modeling, dan penanggung jawab assurance. Pengembang dan operator memperoleh umpan balik lebih awal, sedangkan organisasi tetap memiliki mekanisme eskalasi untuk risiko yang melampaui toleransi.

### Adopsi dan Kematangan DevSecOps

Adopsi DevSecOps adalah perubahan organisasi bertahap. Materi sumber mengusulkan urutan penyelarasan strategi, penilaian kematangan saat ini, perencanaan kondisi sasaran, penyusunan roadmap, serta implementasi dan pengukuran [35]. Urutan tersebut mencegah organisasi membeli tool sebelum memahami bottleneck. Penilaian awal perlu mencakup teknologi, proses, manusia, dan budaya. Sebuah tim mungkin telah memiliki scanner modern tetapi tetap berada pada tingkat kematangan rendah apabila hasilnya tidak memiliki owner, pipeline sering dilewati, atau insiden tidak menghasilkan perbaikan sistemik.

Quick win bermanfaat untuk membangun kepercayaan dan menguji asumsi pada lingkup terbatas. Kandidat yang baik memiliki pain point nyata, pemilik yang bersedia, arsitektur yang dapat dipahami, dan dampak yang dapat diukur dalam satu sampai tiga bulan. Contohnya adalah mengurangi waktu build, menghapus secret dari repositori, menghasilkan SBOM untuk satu layanan, atau menambahkan scan image dengan proses waiver yang jelas. Quick win tidak boleh berhenti sebagai demonstrasi alat. Hasilnya harus diringkas menjadi template, standar, pelajaran, dan keputusan investasi yang dapat digunakan tim lain.

Kematangan kemudian berkembang dari implementasi awal menuju standardisasi lintas tim dan optimasi berbasis continuous feedback. Pada tahap awal, keberhasilan ditentukan oleh kemampuan menjalankan alur dasar secara konsisten. Tahap standardisasi menambahkan governance, mentoring, pelaporan, dan platform bersama. Tahap optimasi menggunakan data untuk memperbaiki rule, mengurangi waktu remediasi, meningkatkan reliabilitas, dan menyesuaikan kontrol terhadap profil risiko. Maturity model berfungsi sebagai alat diagnosis dan perencanaan, bukan kompetisi memperoleh skor tertinggi. Sasaran yang tepat adalah kapabilitas yang proporsional terhadap ancaman, regulasi, dan nilai aset organisasi.

### Kerangka DevSecOps, Risiko, dan Evidence

DevSecOps merupakan pendekatan pengembangan dan pengoperasian perangkat lunak yang mengintegrasikan keamanan ke dalam keputusan, alur kerja, otomasi, dan tanggung jawab sepanjang siklus hidup sistem. Istilah ini bukan berarti membentuk tim baru di antara development dan operations. Maknanya lebih mendasar: keputusan keamanan didistribusikan kepada pihak yang memiliki konteks teknis paling kuat, sementara organisasi menyediakan kebijakan, platform, kompetensi, dan mekanisme verifikasi yang memadai. Dengan demikian, pengembang tidak menunggu audit akhir untuk mengetahui bahwa dependensi berisiko; operator tidak menunggu insiden untuk memahami perilaku normal workload; dan tim keamanan tidak berfungsi sebagai gerbang manual yang selalu berada di ujung proses. Ketiganya bekerja dengan tujuan bersama, bahasa risiko yang konsisten, dan bukti yang dihasilkan secara berulang.

DevOps memperpendek jarak antara perubahan kode dan umpan balik produksi melalui integrasi berkelanjutan, delivery berkelanjutan, infrastructure as code, observability, dan budaya kolaboratif. Kecepatan tersebut sekaligus memperbesar konsekuensi kesalahan: pipeline yang terkompromi dapat menyebarkan artefak berbahaya dalam skala besar, kredensial yang tertanam dalam repositori dapat tereplikasi ke banyak lingkungan, dan konfigurasi yang salah dapat diterapkan secara otomatis. DevSecOps merespons karakteristik ini dengan menempatkan kontrol keamanan pada jalur yang sama dengan delivery. Kontrol tidak boleh dipahami sebagai aktivitas paralel yang terpisah, melainkan sebagai guardrail yang ikut terversi, diuji, dipantau, dan diperbaiki sebagaimana kode aplikasi.

Prinsip tanggung jawab bersama memiliki implikasi tata kelola. Tanggung jawab bersama tidak berarti semua orang melakukan semua pekerjaan keamanan. Pembagian peran tetap diperlukan: product owner menetapkan toleransi risiko dan kebutuhan bisnis; developer menerapkan secure coding dan memperbaiki temuan; platform engineer menyediakan jalur build yang aman; security engineer mengembangkan threat model, rule, dan kebijakan; operator mengelola hardening serta deteksi; auditor menilai kecukupan evidence. Yang dibagi adalah akuntabilitas terhadap hasil, bukan duplikasi tugas. Model RACI, security champion, review dua orang, dan escalation path membantu mencegah area abu-abu ketika sebuah temuan menghambat rilis.

Shift-left berarti memberikan umpan balik keamanan sedini mungkin ketika biaya perubahan masih rendah. Contohnya adalah linting Dockerfile, secret scanning pada pre-commit, static application security testing pada pull request, software composition analysis setelah resolusi dependensi, serta threat modeling pada tahap desain. Namun, shift-left tidak dapat menggantikan pengujian runtime. Analisis statis memiliki keterbatasan konteks, basis data kerentanan berubah setelah rilis, dan serangan terjadi pada lingkungan operasional yang memiliki identitas, jaringan, serta data nyata. Karena itu, shift-right melengkapi shift-left melalui DAST, monitoring, runtime detection, verification saat deploy, chaos/security testing terkontrol, dan pembelajaran pascainsiden.

Otomasi hanya bernilai ketika keputusan yang diotomasi jelas. Sebuah pemindai dapat menghasilkan ratusan temuan, tetapi pipeline tetap tidak aman apabila organisasi tidak menentukan apa yang menyebabkan build gagal, siapa yang dapat memberikan waiver, berapa lama waiver berlaku, dan evidence apa yang wajib disimpan. Security gate adalah keputusan kebijakan berbasis hasil pemeriksaan. Gate yang baik memiliki input terdefinisi, threshold yang transparan, keluaran mesin yang dapat disimpan, mekanisme pengecualian dengan pemilik dan kedaluwarsa, serta jalur remediasi. Gate yang terlalu ketat tanpa triage mendorong tim menonaktifkan alat; gate yang terlalu longgar menciptakan kesan aman yang palsu.

Manajemen risiko menjadi dasar prioritas. Severity CVE tidak identik dengan risiko organisasi. Risiko dipengaruhi oleh keterpaparan, exploitability, keberadaan exploit, privilege proses, nilai aset, kontrol kompensasi, serta dampak bisnis. Temuan Critical pada paket yang tidak pernah dipanggil mungkin memiliki prioritas berbeda dari kelemahan otorisasi berseverity lebih rendah pada endpoint administratif yang terbuka. Oleh sebab itu, buku ini menggunakan threshold sebagai baseline laboratorium, lalu meminta pembaca menganalisis konteks. Hasil scan diperlakukan sebagai evidence untuk keputusan, bukan keputusan itu sendiri.

NIST Secure Software Development Framework mengelompokkan praktik ke dalam Prepare the Organization, Protect the Software, Produce Well-Secured Software, dan Respond to Vulnerabilities. Kerangka ini bersifat outcome-oriented sehingga dapat diterapkan pada model SDLC yang berbeda. OWASP DevSecOps Guideline memberi panduan teknis yang lebih langsung mengenai threat modeling, secrets management, linting, SAST, SCA, DAST, scanning container, dan praktik pipeline. OWASP SAMM serta DevSecOps Maturity Model membantu organisasi menilai kapabilitas secara bertahap. Kerangka tersebut sebaiknya dipakai bersama: NIST menyediakan kosakata tata kelola, OWASP memperkaya teknik implementasi, sedangkan metamodel kematangan membantu menyusun roadmap.

Keamanan rantai pasok perangkat lunak memperluas ruang lingkup dari kode yang ditulis organisasi ke semua input dan proses yang membentuk artefak. Base image, package manager, action atau plugin CI, compiler, build runner, registry, dan metadata rilis merupakan bagian dari rantai pasok. Software Bill of Materials menyediakan inventaris komponen; digest memberi identitas konten; signature mengikat identitas penerbit dengan artefak; provenance menjelaskan bagaimana artefak dibangun; dan verifikasi kebijakan menentukan apakah bukti tersebut memadai untuk deployment. SLSA menyusun tingkat jaminan berdasarkan keberadaan dan ketahanan provenance serta isolasi build.

Container tidak merupakan boundary keamanan yang setara dengan mesin virtual. Container berbagi kernel host dan bergantung pada namespace, cgroup, capability, seccomp, AppArmor atau SELinux, serta konfigurasi runtime. Oleh karena itu, image minimal, pengguna non-root, filesystem read-only, penghapusan Linux capabilities, pembatasan resource, rootless mode, network segmentation, dan penghindaran Docker socket merupakan kontrol penting. Keamanan build juga harus mencegah secret masuk ke layer image. Docker BuildKit menyediakan secret mount yang hanya tersedia selama langkah build dan tidak dimaksudkan menjadi bagian dari image akhir.

Evidence as product adalah cara pandang bahwa keluaran keamanan harus dapat digunakan oleh manusia dan mesin. Laporan SARIF, JUnit XML, SBOM CycloneDX atau SPDX, attestations, hasil policy evaluation, log deployment, serta event runtime perlu memiliki identitas artefak dan retensi yang memadai. Screenshot berguna untuk laporan praktikum, tetapi tidak cukup sebagai sumber kebenaran karena sulit diproses ulang dan dapat kehilangan konteks. Evidence yang baik dapat ditelusuri ke commit, pipeline run, image digest, waktu, tool version, konfigurasi rule, dan keputusan waiver.

Pengukuran DevSecOps seharusnya mendorong perilaku sehat. Menghitung jumlah temuan saja dapat memberikan insentif yang salah, sebab tim mungkin mengurangi cakupan scan agar angka terlihat baik. Metrik yang lebih informatif mencakup mean time to remediate berdasarkan kelas risiko, persentase repositori dengan branch protection, persentase rilis yang memiliki SBOM dan signature terverifikasi, rasio false positive, usia waiver, cakupan threat model, serta keberhasilan recovery drill. Metrik delivery seperti lead time dan change failure rate tetap relevan untuk memastikan kontrol keamanan tidak menimbulkan bottleneck yang tidak proporsional.

Budaya belajar menentukan keberlanjutan program. Temuan sebaiknya disajikan dengan konteks dan saran perbaikan, bukan sebagai alat menyalahkan pengembang. Insiden harus menghasilkan tindakan perbaikan pada sistem: rule baru, unit test regresi, pembaruan threat model, rotasi secret, atau perubahan platform. Security champion membantu menerjemahkan kebutuhan keamanan ke praktik tim, tetapi tidak menggantikan dukungan manajemen dan layanan platform. Investasi pada golden path, template pipeline, dan dokumentasi membuat pilihan aman menjadi pilihan yang paling mudah.

Etika dan legalitas merupakan batas praktikum. DAST dan pengujian serangan hanya boleh dilakukan pada sistem milik sendiri atau sistem yang memiliki izin tertulis dan ruang lingkup jelas. Pemindai dapat menyebabkan beban, perubahan data, atau pemblokiran oleh sistem proteksi. Buku ini menggunakan target lokal dan pengujian pasif sebagai baseline. Data kredensial pada contoh bersifat sintetis; pembaca tidak boleh menempatkan secret nyata di repositori. Aktivitas yang berisiko tinggi ditempatkan sebagai demonstrasi opsional pada lingkungan disposable.

Keberhasilan DevSecOps pada akhirnya tidak diukur dari banyaknya alat, melainkan dari kemampuan organisasi menghasilkan perangkat lunak yang memenuhi kebutuhan, mengurangi risiko secara terukur, memulihkan layanan, dan menyediakan evidence yang dapat dipertanggungjawabkan. Karena itu, tutorial dalam buku ini selalu menghubungkan langkah teknis dengan kontrol, expected result, interpretasi, keterbatasan, dan troubleshooting. Pembaca diarahkan untuk memahami mengapa sebuah gate ada, bukan hanya menyalin perintah.

![Siklus DevSecOps delapan tahap dengan security as code, evidence as product, dan feedback berkelanjutan.](assets/gambar-02.png)

*Gambar 3. Siklus DevSecOps dan umpan balik keamanan.*

> Sumber: diolah penulis berdasarkan OWASP DevSecOps Guideline [16], NIST SSDF [14], dan visual CI/CD ByteByteGo [32].

### Perbandingan Orientasi DevOps dan DevSecOps

| Dimensi | DevOps | DevSecOps |
| --- | --- | --- |
| Tujuan | Kecepatan dan reliabilitas delivery | Kecepatan, reliabilitas, keamanan, dan bukti jaminan |
| Keamanan | Sering berupa review khusus | Terintegrasi pada desain, kode, build, release, deploy, dan operasi |
| Kepemilikan | Development dan operations | Product, development, security, platform, operations, dan governance |
| Artefak | Binary/image dan konfigurasi | Artefak + SBOM + signature + provenance + hasil verifikasi |
| Umpan balik | Test dan observability | Threat model, SAST/SCA/DAST, policy, runtime detection, incident learning |
| Pengecualian | Sering ad hoc | Waiver terdokumentasi, berbatas waktu, dengan risk owner |

> Sumber: diolah penulis berdasarkan NIST SSDF [14] dan OWASP [16].

### Pemetaan NIST SSDF ke Praktik Buku

| Kelompok SSDF | Makna Ringkas | Implementasi dalam Buku |
| --- | --- | --- |
| PO | Prepare the Organization | Peran, baseline, kebijakan gate, threat model, metrik, dan kompetensi. |
| PS | Protect the Software | Proteksi source, secret, build environment, registry, signature, dan provenance. |
| PW | Produce Well-Secured Software | Secure coding, review, SAST, SCA, SBOM, DAST, hardening, dan policy-as-code. |
| RV | Respond to Vulnerabilities | Triage, remediation, rotasi, runtime detection, incident response, dan pembelajaran. |

> Sumber: NIST SP 800-218 SSDF v1.1 [14]; ringkasan dan pemetaan oleh penulis.

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Laboratorium menggunakan satu host Linux atau mesin virtual yang menjalankan Docker Engine dan Docker Compose. Lingkungan diperlakukan sebagai baseline terisolasi: versi sistem, resource, network, izin pengguna, dan status daemon dicatat sebelum eksperimen agar hasil dapat direproduksi.

## Langkah Praktikum Eksploratif

### Praktikum 1 - Menetapkan Baseline Laboratorium

Praktikum ini membuat direktori kerja tunggal dan merekam versi komponen. Rekaman versi diperlukan karena hasil scan dan perilaku tool dapat berubah. Jalankan pada host Linux/VM yang memang disiapkan untuk laboratorium.

```bash
mkdir -p ~/devsecops-lab/{app,policy,reports,sbom,keys}
cd ~/devsecops-lab
docker version
docker compose version
git --version
openssl version
curl --version
docker info --format '{{json .SecurityOptions}}'
```

## Verifikasi dan Skenario Pengujian

- Docker daemon dapat diakses oleh akun praktikum dan docker compose menggunakan plugin v2.

- Direktori reports, sbom, dan keys tidak dipublikasikan sebagai web root.

- SecurityOptions menampilkan mekanisme host yang tersedia; hasil berbeda antardistribusi harus dicatat.

- Pembaca menulis satu paragraf threat statement: aset, aktor ancaman, jalur serangan, dan dampak.

## Analisis Hasil

Versi tool bukan sekadar informasi administratif. Sebuah scan hanya dapat direproduksi apabila versi engine, database vulnerability, rule set, dan target diketahui. Hasil SecurityOptions juga tidak boleh disimpulkan sebagai bukti bahwa seluruh container telah hardened; keluaran tersebut hanya menunjukkan mekanisme yang tersedia pada daemon/host.

## Troubleshooting dan Analisis Hasil

| Gejala | Penyebab Mungkin | Tindakan |
| --- | --- | --- |
| Permission denied pada Docker socket | Akun belum memiliki akses atau rootless context belum aktif | Gunakan mekanisme instalasi resmi; login ulang setelah perubahan grup. Jangan membuka permission socket ke semua pengguna. |
| Compose tidak ditemukan | Plugin compose v2 belum terpasang | Instal docker-compose-plugin dari repositori resmi Docker. |
| SecurityOptions kosong/berbeda | Perbedaan kernel, distro, atau mode daemon | Catat baseline; jangan memaksakan konfigurasi tanpa memahami dukungan host. |
| Disk cepat penuh | Image dan cache build menumpuk | Gunakan docker system df; lakukan cleanup selektif setelah memastikan volume data tidak dibutuhkan. |

## Evaluasi dan Latihan Mandiri

1. Mengapa DevSecOps tidak dapat direduksi menjadi penambahan scanner pada pipeline?
2. Evidence apa yang membedakan klaim kontrol dari kontrol yang benar-benar terverifikasi?
3. Bagaimana shared responsibility memengaruhi ownership risiko dan tindak lanjut temuan?

## Kesimpulan

DevSecOps adalah mekanisme pengelolaan risiko dan evidence pada sistem delivery. Otomasi, budaya, tata kelola, dan teknik keamanan harus berkembang bersama. Baseline yang dicatat pada bab ini menjadi acuan untuk seluruh eksperimen berikutnya.
