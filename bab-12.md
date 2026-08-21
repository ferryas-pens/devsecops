<a id="bab-12"></a>

# Bab 12 — Secure Coding, SAST, dan Secret Scanning

Bab ini membahas secure coding sebagai disiplin untuk mencegah kelemahan sejak penulisan kode, kemudian menggunakan Static Application Security Testing (SAST), secret scanning, pengujian perilaku, dan review manusia sebagai lapisan verifikasi. Praktikum memakai aplikasi Flask mini dan data sintetis. Kredensial nyata tidak boleh dimasukkan ke source code, terminal bersama, tangkapan layar, atau laporan.

## Tujuan Pembelajaran dan Kompetensi Utama

- Membedakan weakness, vulnerability, finding, false positive, false negative, severity, dan confidence.

- Menjelaskan prinsip input validation, output encoding, query terparameterisasi, authorization, serta pengelolaan error dan log.

- Menjelaskan cara kerja SAST berbasis pola, struktur sintaksis, dan aliran data beserta keterbatasannya.

- Menyusun, menguji, dan menjalankan rule Semgrep sederhana pada aplikasi Python.

- Mendeteksi secret sintetis pada working tree dan riwayat Git serta menjelaskan respons yang benar terhadap secret terpapar.

- Melakukan triage, perbaikan, regression test, dan penyusunan evidence untuk gate CI yang dapat diaudit.

## Peta Konsep Bab

![Peta konsep yang menghubungkan secure coding, SAST, secret scanning, verifikasi, triage, perbaikan, dan evidence.](assets/gambar-09.png)

*Gambar 10. Peta konsep secure coding, SAST, dan secret scanning.*

> Sumber: sintesis penulis berdasarkan NIST SSDF [80], OWASP [81-84,90], Semgrep [87-88], dan Gitleaks [89].

## Konsep Inti dan Landasan Teori

### Secure Coding sebagai Praktik Rekayasa

Secure coding adalah penerapan prinsip keamanan ketika perangkat lunak dirancang dan diimplementasikan. Tujuannya bukan membuat kode kebal terhadap semua serangan, melainkan mengurangi kelemahan yang dapat diprediksi, membatasi dampak ketika kegagalan terjadi, dan menyediakan bukti bahwa kontrol bekerja. NIST SP 800-218 versi 1.1 menempatkan praktik keamanan sebagai bagian yang dapat diintegrasikan ke berbagai model SDLC [80]. Artinya, keamanan tidak ditambahkan sesudah aplikasi selesai, tetapi hadir pada requirement, desain, implementasi, review, pengujian, rilis, dan respons terhadap kelemahan.

SAST dan secret scanning merupakan kontrol detektif. Keduanya penting, tetapi tidak menggantikan desain aman, threat modeling, unit test, integration test, dynamic test, dependency analysis, atau review manusia. Alat dapat menemukan pola yang dicurigai, sedangkan keputusan apakah temuan benar, dapat dieksploitasi, dan perlu memblokir rilis membutuhkan konteks. Model yang lebih tepat adalah defense in depth: developer mencegah kelemahan, alat memberikan umpan balik, reviewer memeriksa konteks, test membuktikan perilaku, dan gate memastikan kebijakan dijalankan secara konsisten.

> **Prinsip utama: **Temuan nol tidak sama dengan aplikasi aman. Hasil tersebut hanya menunjukkan bahwa alat, konfigurasi, rule, scope, dan versi tertentu tidak menemukan pola yang cocok pada saat pemindaian.

### Weakness, Vulnerability, dan Finding

Istilah yang tepat membantu mahasiswa menghindari kesimpulan berlebihan. Weakness adalah kelemahan pada desain atau implementasi yang dapat berkontribusi pada kerentanan. Common Weakness Enumeration (CWE) menyediakan bahasa bersama untuk mengelompokkan kelemahan, misalnya CWE-89 untuk SQL injection dan CWE-79 untuk cross-site scripting [85-86]. Vulnerability adalah kondisi pada produk atau sistem tertentu yang dapat dieksploitasi dan menimbulkan dampak. Finding adalah keluaran alat atau reviewer yang masih harus divalidasi. Satu finding dapat terbukti sebagai vulnerability, menjadi perbaikan kualitas tanpa risiko langsung, atau ditutup sebagai false positive dengan alasan dan bukti.

| Istilah | Makna operasional | Pertanyaan saat triage |
| --- | --- | --- |
| Weakness | Pola desain atau implementasi yang lemah. | Kelas kesalahan apa yang terjadi? |
| Vulnerability | Kondisi yang dapat dieksploitasi pada konteks tertentu. | Adakah jalur serangan, aset, dan dampak? |
| Finding | Indikasi dari alat atau reviewer. | Apakah benar, relevan, baru, dan berada dalam scope? |
| False positive | Finding dilaporkan, tetapi kondisi berbahaya tidak ada. | Bukti apa yang menunjukkan hasil ini tidak berlaku? |
| False negative | Kelemahan ada, tetapi tidak dilaporkan. | Apakah rule, bahasa, data flow, atau scope tidak tercakup? |
| Confidence | Tingkat keyakinan bahwa pola yang dilaporkan benar. | Seberapa kuat bukti sintaksis dan konteksnya? |

*Tabel 12.1. Istilah dasar untuk membaca hasil analisis keamanan.*

### Boundary Kepercayaan dan Aliran Data

Secure coding dimulai dengan memahami asal dan tujuan data. Input dari query string, body HTTP, header, cookie, file, message queue, environment variable, atau database eksternal harus diperlakukan sesuai tingkat kepercayaannya. Dalam analisis aliran data, source adalah asal data yang berpotensi tidak tepercaya, propagator adalah operasi yang meneruskan atau mengubah data, sanitizer adalah kontrol yang menurunkan risiko untuk konteks tertentu, dan sink adalah operasi sensitif seperti eksekusi perintah, query database, pembuatan HTML, akses file, atau deserialisasi. Konsep ini menjelaskan mengapa sekadar mencari nama fungsi tidak selalu cukup.

Semgrep mendukung rule pola dan taint analysis. Taint analysis menelusuri aliran data tidak tepercaya dari source menuju sink dalam batas analisis yang didukung [87-88]. Walaupun lebih kontekstual daripada pencocokan teks, hasilnya tetap dipengaruhi kelengkapan model source, sanitizer, sink, bahasa, framework, serta cakupan interprocedural. Reviewer harus memeriksa apakah data benar-benar dapat dikendalikan penyerang, apakah sanitizer sesuai konteks, dan apakah sink dapat dijangkau pada konfigurasi aplikasi yang digunakan.

### Input Validation, Output Encoding, dan Injection

Input validation memastikan data memiliki bentuk, tipe, panjang, rentang, format, dan makna yang dapat diterima sebelum diproses. OWASP menyarankan validasi sedini mungkin setelah data diterima dari pihak eksternal [83]. Allowlist lebih mudah dipertanggungjawabkan daripada denylist karena sistem mendefinisikan nilai yang diperbolehkan. Namun, validasi input bukan pengganti output encoding. Data yang sah sebagai nama, misalnya karakter kurung atau tanda kutip, tetap harus di-encode ketika ditempatkan pada HTML, atribut, JavaScript, URL, atau konteks lain. Encoding harus sesuai konteks keluaran.

SQL injection terjadi ketika data yang dapat dipengaruhi pengguna mengubah struktur perintah SQL. Perbaikan utama bukan menghapus karakter tertentu, melainkan memisahkan data dari perintah melalui prepared statement atau query terparameterisasi [81,85]. Prinsip serupa berlaku pada command injection, LDAP injection, dan template injection: hindari membangun instruksi dengan konkatenasi input. Jika aplikasi harus memilih operasi, gunakan pemetaan eksplisit dari identifier yang diizinkan menuju fungsi yang telah ditentukan. Least privilege pada akun database membatasi dampak jika kontrol aplikasi gagal.

Cross-site scripting terjadi ketika data tidak tepercaya ditempatkan pada keluaran web tanpa netralisasi yang sesuai [86]. Framework modern sering melakukan escaping otomatis pada template, tetapi perlindungan dapat hilang ketika developer menandai string sebagai aman, membangun HTML manual, atau menulis ke DOM melalui API berbahaya. Review perlu memeriksa konteks output, bukan hanya keberadaan fungsi sanitize. Content Security Policy dapat menjadi lapisan tambahan, tetapi bukan alasan untuk mengabaikan encoding yang benar.

### Authentication, Authorization, Error, dan Logging

Authentication menjawab siapa pengguna, sedangkan authorization menentukan tindakan yang boleh dilakukan terhadap objek tertentu. Pemeriksaan authorization harus berlangsung pada server dan diulang untuk setiap operasi sensitif; menyembunyikan tombol pada antarmuka bukan kontrol akses. Kode juga harus mencegah insecure direct object reference dengan memeriksa hubungan antara identitas, peran, dan objek. ASVS menyediakan requirement yang dapat digunakan untuk menyusun acceptance criteria pengembangan dan verifikasi aplikasi web [84].

Pesan error perlu membantu pengguna tanpa mengungkap stack trace, query, path internal, token, atau konfigurasi. Detail diagnosis disimpan pada log yang aksesnya dibatasi. Logging harus mencatat event yang relevan, misalnya kegagalan autentikasi, penolakan authorization, perubahan hak akses, atau aksi administratif, tetapi tidak boleh menyimpan password, access token, private key, atau data pribadi berlebihan. Nilai dari pengguna juga perlu dinormalisasi agar tidak memungkinkan log injection. Dengan demikian, keamanan error handling dan logging mencakup kerahasiaan, integritas bukti, serta kegunaan operasional.

| Area kontrol | Praktik yang dianjurkan | Contoh bukti |
| --- | --- | --- |
| Input | Tipe, panjang, rentang, format, dan allowlist divalidasi pada server. | Negative test untuk nilai batas dan format salah. |
| Database | Gunakan query terparameterisasi dan akun dengan privilege minimum. | Code review, unit test, serta hasil SAST. |
| Output | Lakukan encoding sesuai konteks dan hindari API keluaran berbahaya. | Test respons dan review template/DOM. |
| Authorization | Periksa identitas, aksi, dan objek pada setiap request sensitif. | Test pengguna sah, pengguna lain, dan tanpa autentikasi. |
| Error dan log | Tampilkan pesan umum; simpan detail terkontrol tanpa secret. | Review log dan test error path. |
| Secret | Ambil dari secret manager atau sumber terproteksi; rotasi dan audit. | Secret scan, konfigurasi deployment, dan catatan rotasi. |

*Tabel 12.2. Pemetaan praktik secure coding dan bukti verifikasi.*

### Cara Kerja dan Batas SAST

SAST menganalisis source code atau representasi program tanpa menjalankan aplikasi. Mesin dapat menggunakan token, abstract syntax tree, semantic information, control flow, serta data flow. Rule Semgrep mengenkapsulasi pola atau logika aliran yang hendak dideteksi [87]. Kelebihan SAST adalah umpan balik relatif cepat, lokasi temuan dekat dengan baris kode, dan kemampuan dijalankan sebelum aplikasi dapat dideploy. Hal ini membuat SAST sesuai untuk editor, pre-commit terpilih, pull request, dan pipeline CI.

Keterbatasan SAST berasal dari model yang tidak lengkap. Dynamic dispatch, reflection, generated code, framework convention, konfigurasi runtime, dependency behavior, dan aliran lintas layanan dapat sulit dianalisis. Rule terlalu umum menghasilkan noise, sedangkan rule terlalu sempit melewatkan variasi kelemahan. Karena itu, kualitas program SAST tidak dinilai hanya dari jumlah rule. Ukuran yang lebih berguna mencakup coverage bahasa dan framework, waktu umpan balik, proporsi temuan valid, usia temuan, waktu remediasi, cakupan test rule, serta jumlah kelemahan berulang yang berhasil dicegah.

Custom rule berguna untuk pola yang khas organisasi, misalnya fungsi wrapper internal, larangan penggunaan API tertentu, atau kewajiban authorization helper. Rule harus memiliki identifier stabil, pesan yang menjelaskan risiko dan perbaikan, severity yang konsisten, metadata CWE atau requirement, contoh positif, dan contoh negatif. Semgrep menyediakan mekanisme pengujian rule sehingga penulis dapat menandai baris yang harus dan tidak boleh menghasilkan finding [88]. Rule yang belum diuji dapat menjadi sumber false positive maupun false negative dan seharusnya tidak langsung dijadikan gate keras.

### Secret Scanning dan Respons Kebocoran

Secret scanning mencari pola kredensial seperti API key, token, password, atau private key pada file dan riwayat version control. Detektor dapat menggunakan prefix, regular expression, checksum, entropi, konteks kata kunci, atau validasi terhadap penyedia layanan. Gitleaks dapat memindai repository Git, direktori, file, dan input standar [89]. Secret scanning berbeda dari SAST umum karena objek utamanya adalah kredensial dan material autentikasi, bukan semata perilaku kode.

Menghapus secret dari commit terbaru tidak membatalkan paparan. Secret dapat tetap berada pada commit lama, cache CI, artefak, fork, log, atau salinan lokal. Respons pertama adalah menganggap secret telah diketahui pihak lain, kemudian mencabut atau merotasi kredensial, memeriksa log penggunaan, menilai dampak, dan baru setelah itu mempertimbangkan pembersihan riwayat sesuai prosedur organisasi [90]. Rewriting history dapat mengganggu kolaborator dan tidak menjamin seluruh salinan hilang. Oleh sebab itu, pencegahan melalui secret manager, pre-commit scan, least privilege, masa berlaku singkat, serta monitoring penggunaan lebih kuat daripada hanya mengandalkan pembersihan.

Allowlist harus sempit, memiliki owner, alasan, scope, dan masa tinjau. Menambahkan pola luas seperti seluruh folder test dapat menyembunyikan secret nyata yang tidak sengaja ditempatkan di sana. Untuk buku dan laboratorium ini, hanya digunakan token sintetis yang telah dipublikasikan sebagai contoh dokumentasi. Mahasiswa tetap harus menandainya sebagai data uji, tidak menggunakannya untuk layanan nyata, dan menghapusnya setelah eksperimen.

### Triage, Prioritas, dan Security Gate

Triage mengubah finding menjadi keputusan. Reviewer memeriksa kebenaran pola, sumber input, sink, jalur eksekusi, authorization, kontrol kompensasi, aset, dampak, likelihood, confidence, serta apakah finding diperkenalkan oleh perubahan saat ini. Severity dari alat adalah titik awal, bukan kebenaran final. Finding dengan label tinggi dapat tidak terjangkau pada konfigurasi tertentu, sedangkan pola sedang pada endpoint administratif yang terekspos dapat memiliki risiko besar. Keputusan harus mencatat owner, status, alasan, target perbaikan, dan evidence.

| Keputusan | Kriteria minimum | Tindakan |
| --- | --- | --- |
| Perbaiki sekarang | Valid, dapat dijangkau, dampak tinggi, atau melanggar kebijakan wajib. | Blok merge; tambah test; perbaiki dan scan ulang. |
| Perbaiki terjadwal | Valid, risiko terbatas, ada kontrol sementara, dan target jelas. | Buat tiket, owner, due date, serta residual risk. |
| False positive | Kondisi berbahaya tidak ada dan bukti teknis dapat direproduksi. | Dokumentasikan; gunakan pengecualian paling sempit. |
| Accepted risk | Risiko dipahami dan disetujui pemilik berwenang untuk waktu terbatas. | Catat alasan, expiry, kontrol kompensasi, dan review. |
| Needs investigation | Bukti belum cukup atau lingkungan tidak lengkap. | Jangan menyatakan aman; kumpulkan data tambahan. |

*Tabel 12.3. Keputusan triage dan bukti yang diperlukan.*

Gate yang sehat bersifat risk-based dan change-aware. Pada tahap awal, organisasi dapat memblokir secret yang terverifikasi, error SAST baru dengan confidence tinggi, serta kegagalan security unit test. Temuan lama dimasukkan baseline dan memiliki rencana pengurangan agar tim tidak lumpuh oleh utang keamanan. Policy-as-code menjaga konsistensi, tetapi tetap memerlukan jalur pengecualian yang terdokumentasi. Gate tanpa mekanisme triage mendorong developer mengabaikan alat; gate tanpa konsekuensi berubah menjadi laporan pasif.

Evidence minimal meliputi versi atau digest alat, konfigurasi dan rule, commit yang dipindai, scope file, waktu pemindaian, exit code, laporan teredaksi, hasil test, keputusan triage, dan referensi perbaikan. Evidence tidak boleh menyimpan secret mentah. Redaction melindungi tampilan laporan, tetapi apabila secret nyata ditemukan, nilai tersebut tetap harus dicabut atau dirotasi. Bukti yang dapat direproduksi memungkinkan reviewer membedakan perbaikan nyata dari sekadar perubahan konfigurasi pemindai.

## Siklus Praktikum

![Diagram alir perubahan kode menuju unit test, SAST dan secret scan, triage, perbaikan, review, gate, evidence, lalu kembali sebagai umpan balik.](assets/gambar-10.png)

*Gambar 11. Secure feedback loop pada perubahan kode.*

> Sumber: sintesis penulis berdasarkan NIST SSDF [80], OWASP Secure Code Review [82], Semgrep [87-88], dan Gitleaks [89].

## Arsitektur Laboratorium dan Prasyarat Lingkungan

Studi kasus berupa endpoint kalkulator pada Flask. Versi awal menggunakan eval terhadap input pengguna sehingga operasi yang diterima lebih luas daripada requirement. Praktikum membuktikan kelemahan melalui negative test, mendeteksinya dengan custom rule Semgrep, memperbaikinya menggunakan allowlist operasi, lalu menjalankan secret scanning pada nilai sintetis. Lingkungan harus disposable dan tidak memuat credential nyata.

- Docker Engine dan Git tersedia; semua perintah dijalankan pada direktori laboratorium milik mahasiswa.

- Gunakan image atau paket yang disetujui dosen; catat versi dan digest pada laporan.

- Buat direktori reports dan jangan memublikasikan laporan yang mungkin berisi nilai sensitif.

- Eksperimen secret hanya menggunakan nilai sintetis pada repository lokal yang dapat dibuang.

## Langkah Praktikum Eksploratif

### Langkah 1 - Membuat Aplikasi yang Sengaja Lemah

```python
mkdir -p ~/devsecops-lab/ch12/{app,policy,tests,reports}
cd ~/devsecops-lab/ch12
cat > app/app.py <<'PY'
from flask import Flask, jsonify, request

app = Flask(__name__)

@app.get('/calculate')
def calculate():
    expression = request.args.get('q', '0')
    # SENGAJA LEMAH UNTUK LAB: jangan gunakan eval pada input pengguna.
    return jsonify(result=eval(expression))
PY
```

> **Batas etis laboratorium: **Jangan mengekspos aplikasi lemah ke jaringan publik. Contoh dibuat untuk pembelajaran lokal dan harus diperbaiki sebelum eksperimen dinyatakan selesai.

### Langkah 2 - Menulis Negative Test

```python
cat > tests/test_security.py <<'PY'
import sys
from pathlib import Path

sys.path.insert(0, str(Path(__file__).parents[1] / 'app'))
from app import app

def test_only_documented_addition_is_allowed():
    client = app.test_client()
    response = client.get('/calculate?q=2**8')
    assert response.status_code == 400

def test_addition_works():
    client = app.test_client()
    response = client.get('/calculate?q=2+3')
    assert response.status_code == 200
    assert response.get_json()['result'] == 5
PY

python -m venv .venv
. .venv/bin/activate
python -m pip install --upgrade pip
python -m pip install flask pytest
pytest -q
```

Pada versi lemah, test pertama diharapkan gagal karena ekspresi 2**8 diterima dan dievaluasi, walaupun requirement hanya mengizinkan penjumlahan dua bilangan bulat. Kegagalan ini merupakan evidence bahwa perilaku aplikasi lebih luas daripada spesifikasi. Simpan output dan exit code tanpa menyembunyikan hasil gagal yang memang diharapkan pada baseline.

### Langkah 3 - Membuat dan Menguji Rule Semgrep

```python
cat > policy/flask-security.yml <<'YAML'
rules:
  - id: flask-dangerous-eval
    languages: [python]
    message: Hindari eval; petakan operasi yang diizinkan secara eksplisit.
    severity: ERROR
    metadata:
      cwe: CWE-95
      requirement: SR-CODE-12-01
    pattern: eval(...)
YAML

cat > tests/rule_test.py <<'PY'
def weak(user_input):
    # ruleid: flask-dangerous-eval
    return eval(user_input)

def expected(user_input):
    # ok: flask-dangerous-eval
    return int(user_input)
PY

docker run --rm -v "$PWD:/src" -w /src semgrep/semgrep   semgrep --test --config policy/flask-security.yml tests/rule_test.py

docker run --rm -v "$PWD:/src" -w /src semgrep/semgrep   semgrep scan --config policy/flask-security.yml app tests   --error --json --output reports/semgrep-before.json
```

Perintah --test memeriksa apakah rule menghasilkan finding pada contoh ruleid dan tidak menghasilkan finding pada contoh ok. Pemindaian aplikasi sebelum perbaikan harus menemukan penggunaan eval. Catat versi image atau digest, identifier rule, file dan baris, severity, exit code, serta hash commit. Jangan hanya menyalin jumlah finding.

### Langkah 4 - Memperbaiki Kode dengan Allowlist Operasi

```python
cat > app/app.py <<'PY'
import operator
from flask import Flask, jsonify, request

app = Flask(__name__)
ALLOWED = {'add': operator.add, 'sub': operator.sub}

@app.get('/calculate')
def calculate():
    expression = request.args.get('q', '')
    operation = request.args.get('op', 'add')
    parts = expression.split('+') if operation == 'add' else expression.split('-')
    if operation not in ALLOWED or len(parts) != 2:
        return jsonify(error='operasi tidak diizinkan'), 400
    try:
        left, right = (int(value.strip()) for value in parts)
    except ValueError:
        return jsonify(error='operand harus bilangan bulat'), 400
    return jsonify(result=ALLOWED[operation](left, right))
PY

pytest -q
docker run --rm -v "$PWD:/src" -w /src semgrep/semgrep   semgrep scan --config policy/flask-security.yml app tests   --error --json --output reports/semgrep-after.json
```

Perbaikan menghapus evaluasi kode dinamis dan memetakan identifier operasi ke fungsi yang telah ditentukan. Test 2**8 harus menghasilkan HTTP 400, sedangkan 2+3 harus tetap menghasilkan 5. Semgrep setelah perbaikan tidak boleh lagi melaporkan flask-dangerous-eval pada aplikasi. Reviewer tetap perlu menilai nilai batas bilangan, format operator, error handling, dan requirement tambahan; hilangnya satu finding tidak membuktikan seluruh endpoint aman.

### Langkah 5 - Eksperimen Secret Scanning yang Aman

```bash
git init
git config user.name 'DevSecOps Lab'
git config user.email 'lab@example.invalid'
printf 'AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
' > app/.env.lab
git add app/.env.lab
git commit -m 'lab: add synthetic documentation token'
git rm app/.env.lab
git commit -m 'lab: remove synthetic documentation token'

docker run --rm -v "$PWD:/repo" -w /repo zricethezav/gitleaks:latest   git --redact --report-format json   --report-path reports/gitleaks-history.json

docker run --rm -v "$PWD:/repo" -w /repo zricethezav/gitleaks:latest   dir --redact --report-format json   --report-path reports/gitleaks-working-tree.json .
```

Nilai AKIAIOSFODNN7EXAMPLE adalah contoh sintetis yang telah lama digunakan pada dokumentasi publik dan tidak boleh diganti dengan key nyata. Pemindaian working tree sesudah file dihapus dapat lulus, sedangkan pemindaian riwayat Git seharusnya masih menemukan pola pada commit lama. Perbedaan ini menunjukkan bahwa penghapusan file tidak menghapus sejarah. Jika pola tidak terdeteksi, periksa versi alat, konfigurasi rule, scope, exit code, dan isi repository; jangan langsung menyimpulkan aman.

> **Respons secret nyata: **Cabut atau rotasi terlebih dahulu, periksa penggunaan, dokumentasikan insiden, lalu lakukan pembersihan riwayat melalui prosedur yang disetujui. Redaction laporan tidak membuat secret kembali aman.

### Langkah 6 - Menyusun Gate dan Evidence

```python
python - <<'PY'
import json
from pathlib import Path

required = [
    Path('reports/semgrep-before.json'),
    Path('reports/semgrep-after.json'),
    Path('reports/gitleaks-history.json'),
    Path('reports/gitleaks-working-tree.json'),
]
missing = [str(path) for path in required if not path.exists()]
assert not missing, f'evidence belum lengkap: {missing}'
json.loads(Path('reports/semgrep-after.json').read_text())
print('PASS: evidence tersedia dan JSON Semgrep dapat dibaca')
PY
```

Buat matriks traceability yang menghubungkan requirement, weakness atau rule, test, hasil, perbaikan, dan evidence. Gate akhir praktikum dinyatakan lulus apabila unit test dan negative test lulus, custom rule telah diuji, finding eval tidak ada pada kode perbaikan, laporan teredaksi tersedia, serta mahasiswa dapat menjelaskan mengapa pemindaian riwayat secret berbeda dari pemindaian working tree.

## Verifikasi dan Skenario Pengujian

| ID | Pengujian | Kondisi PASS | Evidence |
| --- | --- | --- | --- |
| TEST-12-01 | Baseline negative test | Versi lemah gagal pada input 2**8. | Output pytest sebelum perbaikan. |
| TEST-12-02 | Functional regression | 2+3 menghasilkan 5 setelah perbaikan. | Output pytest sesudah perbaikan. |
| TEST-12-03 | Input restriction | 2**8 dan format tidak sah menghasilkan HTTP 400. | Test report dan respons. |
| TEST-12-04 | Semgrep rule test | Contoh ruleid dan ok sesuai ekspektasi. | Output semgrep --test. |
| TEST-12-05 | SAST after fix | Tidak ada finding flask-dangerous-eval pada app. | semgrep-after.json. |
| TEST-12-06 | Secret working tree | File sintetis yang dihapus tidak ada pada tree. | gitleaks-working-tree.json. |
| TEST-12-07 | Secret history | Pola sintetis pada commit lama terdeteksi. | gitleaks-history.json teredaksi. |
| TEST-12-08 | Evidence integrity | Semua laporan tersedia, terbaca, dan terkait commit. | Matriks traceability dan hash commit. |

*Tabel 12.4. Skenario pengujian dan evidence Bab 12.*

## Analisis Hasil

Analisis pertama membandingkan perilaku sebelum dan sesudah perbaikan. Pada baseline, input 2**8 diterima karena eval menafsirkan input sebagai ekspresi Python. Sesudah perbaikan, aplikasi tidak lagi mengevaluasi bahasa pemrograman; ia hanya memilih fungsi dari allowlist dan mengonversi operand menjadi integer. Perubahan ini mempersempit attack surface sekaligus membuat requirement dapat diuji. Mahasiswa harus menjelaskan kontrol yang berubah, bukan hanya menyatakan “test sudah pass”.

Analisis kedua membandingkan SAST dengan unit test. Rule sederhana pattern: eval(...) mendeteksi API yang dilarang secara langsung, tetapi tidak membuktikan bahwa argumen dapat dikendalikan pengguna. Sebaliknya, negative test membuktikan perilaku endpoint untuk contoh input, tetapi tidak mencakup seluruh jalur program. Keduanya saling melengkapi. Rule taint dapat meningkatkan konteks dengan memodelkan source dan sink, namun tetap harus diuji dan ditinjau terhadap framework serta wrapper yang digunakan.

Analisis ketiga membedakan working tree dari riwayat Git. Secret yang tidak ada pada file saat ini masih dapat ditemukan pada commit lama. Oleh sebab itu, kebijakan yang hanya memindai direktori build berpotensi melewatkan paparan historis, sedangkan pemindaian seluruh riwayat pada setiap commit dapat menambah waktu. Strategi praktis adalah pre-commit atau pull-request scan untuk perubahan baru, pemindaian riwayat terjadwal, dan incident response yang memprioritaskan rotasi. Pilihan harus didokumentasikan berdasarkan risiko dan biaya operasional.

Analisis keempat menilai kualitas gate. Gate yang memblokir semua temuan tanpa triage berisiko mendorong pengecualian luas. Gate yang tidak pernah memblokir tidak memberikan kontrol. Kebijakan yang lebih dapat dipertanggungjawabkan memblokir secret terverifikasi, security test gagal, dan finding baru dengan severity serta confidence tinggi; temuan lain memperoleh owner dan tenggat. Setiap pengecualian memiliki alasan, scope sempit, dan expiry.

| Dimensi review | Indikator kuat | Indikator lemah |
| --- | --- | --- |
| Coverage | Source, sink, test negatif, secret history, dan scope. | Hanya satu scan. |
| Reproducibility | Versi/digest, commit, rule, command, dan exit code. | Screenshot tanpa konteks. |
| Triage | Validasi, keputusan, owner, alasan, dan evidence. | Severity alat dianggap risiko final. |
| Remediation | Root cause diperbaiki; regression test ditambahkan. | Finding disembunyikan dengan allowlist luas. |
| Secret response | Rotasi/revocation didahulukan; history dipahami. | Hanya menghapus file pada commit terbaru. |
| Safety | Data sintetis, laporan teredaksi, lab tidak dipublikasikan. | Credential nyata masuk repo atau laporan. |

*Tabel 12.5. Rubrik analisis hasil secure coding dan scanning.*

## Troubleshooting dan Analisis Hasil

| Gejala | Penyebab yang mungkin | Tindakan korektif |
| --- | --- | --- |
| pytest tidak menemukan modul app | Path proyek atau virtual environment salah. | Aktifkan .venv; jalankan dari root ch12; periksa sys.path test. |
| Negative test langsung lulus pada baseline | Kode lemah sudah berubah atau test tidak memanggil endpoint yang benar. | Periksa app/app.py, route, input 2**8, dan status response. |
| Semgrep rule tidak valid | Indentasi YAML atau field rule tidak lengkap. | Jalankan semgrep --validate/--test; periksa id, languages, message, severity, dan pattern. |
| Semgrep tidak menemukan eval | Scope salah, file dikecualikan, atau rule tidak dimuat. | Tampilkan konfigurasi; scan app/app.py secara eksplisit; periksa exit code. |
| Finding tetap ada setelah perbaikan | Fixture rule_test.py ikut dipindai atau artefak lama terbaca. | Pisahkan scope aplikasi dan test rule; baca lokasi finding sebelum mengubah allowlist. |
| Gitleaks tidak menemukan token sintetis | Subcommand, versi, rule, atau history Git tidak sesuai. | Catat gitleaks version; periksa git log; jalankan mode git dan dir secara terpisah. |
| Laporan JSON kosong atau tidak dibuat | Folder reports tidak ada, permission, atau proses berhenti lebih awal. | Buat folder; periksa mount, path, exit code, serta stdout/stderr. |
| Pipeline membocorkan nilai secret | Output tidak diredaksi atau debug logging aktif. | Hentikan publikasi log; rotasi secret nyata; gunakan --redact dan masking CI. |
| Terlalu banyak false positive | Rule terlalu umum atau scope terlalu luas. | Triage sampel; perbaiki rule; tambah negative fixture; gunakan pengecualian sempit. |

*Tabel 12.6. Troubleshooting praktikum Bab 12.*

## Kesimpulan

Secure coding mencegah kelas kelemahan melalui desain dan implementasi yang terkontrol. SAST mempercepat deteksi pola kode, sedangkan secret scanning membantu menemukan kredensial pada file dan sejarah version control. Ketiganya efektif apabila ditempatkan dalam feedback loop yang mencakup requirement, negative test, rule yang diuji, triage kontekstual, perbaikan root cause, review manusia, dan evidence. Tidak ada satu alat yang membuktikan keamanan secara mutlak.

Bagi mahasiswa, keterampilan inti bukan sekadar menjalankan Semgrep atau Gitleaks, melainkan menjelaskan hubungan antara data tidak tepercaya, operasi sensitif, finding, risiko, perbaikan, dan bukti. Praktikum dinyatakan berhasil ketika perilaku lemah dapat direproduksi secara aman, kontrol diperbaiki tanpa merusak fungsi yang sah, hasil scan dipahami keterbatasannya, dan keputusan gate dapat diaudit.

## Evaluasi dan Latihan Mandiri

- Mengapa finding dari SAST belum dapat langsung disebut vulnerability?

- Apa perbedaan input validation dan output encoding? Berikan konteks penggunaan masing-masing.

- Mengapa query terparameterisasi lebih kuat daripada menghapus tanda kutip dari input?

- Bagaimana source, sanitizer, dan sink digunakan dalam taint analysis?

- Mengapa rule Semgrep perlu memiliki contoh ruleid dan ok?

- Mengapa secret yang telah dihapus dari working tree masih harus dirotasi?

- Kapan finding layak memblokir merge dan kapan dapat diperbaiki terjadwal?

- Mengapa allowlist temuan harus sempit, memiliki owner, alasan, dan masa tinjau?

- Evidence minimum apa yang membuat hasil scan dapat direproduksi?

- Jelaskan satu false negative yang mungkin terjadi pada praktikum ini dan cara menguranginya.

## Format Laporan Praktikum

Laporan Bab 12 menggunakan bahasa formal akademis, maksimum 10 halaman di luar lampiran evidence. Data harus sintetis dan laporan pemindai wajib teredaksi. Laporan minimum memuat:

- Tujuan, scope, asumsi, arsitektur sederhana, versi/digest alat, dan hash commit.

- Kode sebelum dan sesudah perbaikan dengan penjelasan root cause.

- Hasil unit test dan negative test sebelum serta sesudah perbaikan.

- Custom rule Semgrep, rule test, hasil scan, dan satu pembahasan keterbatasan rule.

- Perbandingan Gitleaks mode working tree dan Git history menggunakan token sintetis.

- Matriks requirement–finding–test–perbaikan–evidence dan keputusan gate.

- Analisis false positive/false negative, residual risk, kesimpulan, dan rekomendasi.
