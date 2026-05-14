# AI-Based Audit Checker - High-Level Planning

## 1. Ringkasan Proyek

Aplikasi ini merupakan sistem otomasi pengecekan audit (Audit Checker) berbasis AI (Vision & Text Analysis). Sistem bertujuan untuk menggantikan proses pengecekan manual dengan membandingkan data input (sistem perusahaan) terhadap file pendukung (gambar nota, dokumen, PDF) untuk menemukan indikasi anomali atau pelanggaran. Aplikasi dirancang secara dinamis agar mampu menangani berbagai jenis "Tema Audit" secara modular (scalable).

## 2. High-Level Architecture

Sistem menggunakan pendekatan modular agar penambahan "Tema Audit" baru tidak mengganggu (atau memodifikasi) _core logic_ yang sudah berjalan (Open-Closed Principle).

- **Design Pattern (Code Level):**
  - **Strategy Pattern & Factory Pattern:**
    - `AuditStrategyInterface`: Kontrak baku (misal: method `analyze(Payload $data, File $file)`).
    - `Concrete Strategies`: Implementasi per tema (contoh: `EntertainmentAuditStrategy`, `BbmAuditStrategy`).
    - `AuditFactory`: Mengambil/menghasilkan instance Strategy yang tepat berdasarkan parameter `tema_audit` dari request.
- **Database Architecture:**
  - `audits` (Tabel Master): Menyimpan ID transaksi, `tema_audit`, status (pending/success/failed), dan waktu dibuat.
  - `audit_payloads`: Menyimpan data dinamis (input data sistem) menggunakan tipe kolom `JSON` agar skema tidak perlu di-_alter_ setiap kali ada tema audit baru dengan parameter yang berbeda.
  - `audit_findings`: Menyimpan hasil temuan anomali dari AI beserta deskripsinya (satu audit bisa memiliki banyak temuan).

## 3. Alur Kerja Sistem (System Workflow)

1. **Data Ingestion:** Data (payload text/angka + file media) masuk melalui API atau Internal Web Form.
2. **Dispatching to Queue:** Job audit dimasukkan ke dalam antrean (Queue) di _background_ agar _request_ tidak mengalami _timeout_, mengingat proses AI vision bisa memakan waktu.
3. **AI Processing (Worker Level):**
   - **OpenAI API Integration:** Sistem secara spesifik menggunakan OpenAI API (GPT-4o Vision & Text). Implementasi dirancang untuk mendukung 2 mode eksekusi:
     - **Realtime Mode:** Digunakan jika hasil audit dibutuhkan secepatnya (misal: di-trigger manual oleh auditor). Request dikirim secara sinkron atau via standard queue ke OpenAI.
     - **Batch Mode (Default/Utama):** Karena proses audit massal memakan waktu dan biaya, data akan diakumulasi (pooled) ke dalam format JSONL, lalu dikirim via **OpenAI Batch API** untuk mendapatkan diskon cost 50% dan efisiensi _rate limit_.
   - _Worker_ dan `AuditFactory` akan menentukan apakah _job_ dieksekusi secara _Realtime_ atau dimasukkan ke _Batch processing_.
   - Setelah AI selesai memproses (baik dari respons _Realtime_ maupun hasil _Batch download_), data di-parsing untuk menentukan "Lolos" atau "Anomali".
4. **Recording:** Jika ada anomali, sistem menyimpannya ke dalam tabel `audit_findings` dan mengubah status di tabel `audits` menjadi _Completed with Findings_.
5. **Reporting (Email):** Scheduler berjalan secara berkala untuk merekap temuan yang belum terkirim, lalu menyusunnya menjadi _body email_ dan mengirimkannya ke pihak Auditor/Manajemen.

## 4. Deployment Strategy (Native Windows 7 & 10)

Karena _production environment_ dijalankan secara _bare-metal_ tanpa Docker pada OS lawas, berikut strategi konfigurasinya:

- **Stack Versioning (Aman untuk Windows 7):**
  - **PHP Version:** PHP 8.1. Versi ini sangat stabil untuk Laravel 10 dan masih bisa berjalan di Windows 7 SP1 (memerlukan instalasi _Visual C++ Redistributable for Visual Studio 2019/2022_). (Jangan gunakan PHP 8.2+ karena support Windows 7 sudah banyak dihilangkan).
  - **MySQL Version:** MySQL 5.7 atau MariaDB 10.4. MySQL 8.x sering menimbulkan isu dependensi memori/CPU di OS Windows lawas. MySQL 5.7 sangat aman dan ringan.
- **Rekomendasi Web Server Bundle:** Gunakan **Laragon** daripada XAMPP. Laragon lebih terisolasi, mudah di-upgrade/downgrade versi PHP-nya secara _drop-in_, dan otomatis menangani _virtual host_.
- **Background Processes (Queue & Scheduler):**
  - Karena Windows tidak memiliki `cron`, Anda WAJIB menggunakan **NSSM (Non-Sucking Service Manager)** untuk mendaftarkan command `php artisan queue:work` dan `php artisan schedule:run` sebagai **Windows Service** agar berjalan otomatis di latar belakang saat Windows _booting_, tanpa terminal yang terbuka.

## 5. Daftar Task Utama (Epic / Stories)

- **Epic 1: Foundation & Architecture Setup**
  - Inisialisasi proyek Laravel dan konfigurasi koneksi DB.
  - Pembuatan struktur Migration (`audits`, `audit_payloads`, `audit_findings`).
  - Implementasi interface `AuditStrategy` dan `AuditFactory`.
- **Epic 2: OpenAI API Integration (Realtime & Batch)**
  - Membuat _Service Class_ untuk interaksi dengan OpenAI API (Vision & Text).
  - Implementasi **OpenAI Batch API Handler**: Pembuatan file JSONL, mekanisme _upload batch_, sinkronisasi (_polling_) status _batch_, dan _download/parsing_ hasil _batch_.
  - Implementasi **Realtime API Handler** sebagai mode opsional untuk pengecekan audit instan.
  - Membuat sistem manajemen _Prompt Template_ dinamis untuk AI vision.
- **Epic 3: Theme Implementations**
  - **Story:** Implementasi _Entertainment Audit_ (Validasi alkohol, karaoke, nominal tips).
  - **Story:** Implementasi _BBM Audit_ (Kalkulasi rasio KM, kelengkapan plat nomor, argometer, konversi tarif).
- **Epic 4: Queue & Background Worker**
  - Setup Laravel Queue (menggunakan database driver untuk kemudahan instalasi di Windows).
  - Implementasi _Job_ untuk memproses _AuditStrategy_ di _background_.
- **Epic 5: Reporting Module**
  - Pembuatan template email HTML dinamis untuk rekapitulasi temuan.
  - Membuat _Command_ & _Scheduler_ untuk mengirim laporan _batch_ ke auditor secara berkala.
