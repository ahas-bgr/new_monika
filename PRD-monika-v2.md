# PRD: MONIKA v2 — Aplikasi Monitoring Proyek (Pipeline SPH → Pembayaran)

> Prompt ini siap ditempel ke Claude Code / AI coding assistant. Kerjakan secara bertahap sesuai fase di bagian akhir dokumen. Setelah tiap fase, jalankan aplikasi dan pastikan alur transisi status berfungsi sebelum lanjut.

## 1. Ringkasan Produk

Aplikasi web internal untuk tim kecil yang memantau siklus hidup proyek dari penawaran hingga pembayaran. Setiap proyek berjalan melalui pipeline berurutan sebagai **jalur normal**, namun sistem mengakomodasi dinamika lapangan (PO menyusul, invoice DP sebelum pengerjaan selesai, BAST tertahan di klien) melalui **mekanisme override yang tercatat dan tertagih** — bukan dengan melonggarkan aturan.

**Prinsip desain:** disiplin proses sebagai default, fleksibilitas sebagai pengecualian yang terdokumentasi. Setiap penyimpangan meninggalkan jejak audit dan "hutang dokumen" yang terlihat di dashboard sampai dilunasi.

**Pipeline utama:**

```
1. Buat & Kirim SPH → 2. Terima PO → 3. Pengerjaan Proyek → 4. BAST → 5. Invoice → 6. Pembayaran Masuk → 7. Selesai
```

## 2. Tech Stack

- **Backend:** Laravel 11 (PHP 8.3)
- **Database:** MySQL 8
- **Frontend:** Blade + Livewire 3 + Tailwind CSS (SPA-like tanpa framework JS terpisah)
- **Auth:** Laravel Breeze (session-based)
- **PDF:** barryvdh/laravel-dompdf untuk cetak SPH, BAST, dan Invoice
- **Ekspor:** maatwebsite/excel
- **Deployment target:** VPS (Nginx + PHP-FPM)

## 3. Pengguna & Peran

| Peran | Hak Akses |
|---|---|
| Admin | Semua fitur + kelola user + hapus/arsip data + pengaturan perusahaan + **override transisi status** |
| Staff | Buat & kelola proyek, SPH, update progres, invoice; tidak bisa kelola user, hapus permanen, atau override |

Multi-user dalam satu perusahaan (bukan SaaS multi-tenant).

## 4. Konsep Inti: State Machine Proyek dengan Override Terkendali

### 4.1 Status & Transisi Normal

Setiap proyek punya satu status aktif. Transisi normal hanya maju satu langkah, dengan syarat tahap sebelumnya selesai.

| # | Status | Syarat Masuk Normal | Syarat Selesai Normal |
|---|---|---|---|
| 1 | `sph_draft` | Proyek dibuat | SPH revisi aktif ditandai "Terkirim" + tanggal kirim |
| 2 | `menunggu_po` | SPH terkirim | Nomor PO + tanggal PO + file PO diinput |
| 3 | `pengerjaan` | PO diterima | Semua task selesai (progress 100%) |
| 4 | `bast` | Pengerjaan selesai | BAST dibuat & ditandai ditandatangani + scan diupload |
| 5 | `invoicing` | BAST selesai | Minimal 1 invoice diterbitkan & dikirim |
| 6 | `menunggu_pembayaran` | Invoice terkirim | Total pembayaran ≥ total nilai seluruh invoice proyek |
| 7 | `selesai` | Pembayaran lunas | — (masuk riwayat/arsip) |

### 4.2 Override Transisi (khusus Admin) — INTI FLEKSIBILITAS

Realita lapangan: pekerjaan sering harus jalan sebelum PO resmi terbit, invoice DP diminta sebelum BAST, dsb. Aturannya:

- Di setiap tahap, admin punya tombol **"Lanjutkan Tanpa Syarat Lengkap (Override)"**.
- Override **wajib** mengisi alasan (textarea, min. 10 karakter).
- Override mencatat entri di `project_status_logs` dengan flag `is_override = true`, alasan, user, dan **daftar syarat yang belum terpenuhi** (JSON, mis. `["po_document"]`).
- Syarat yang dilewati menjadi **hutang dokumen** (lihat 4.3) — proyek maju, tapi kewajibannya tidak hilang.
- Contoh kanonis: proyek di `menunggu_po`, klien minta kerja dimulai. Admin override → status jadi `pengerjaan`, proyek dapat badge **"PO Menyusul"**, dan muncul di panel Dokumen Menyusul sampai PO diupload.
- Override juga mengizinkan **penerbitan invoice DP saat proyek masih `pengerjaan`** (lihat modul Invoice) tanpa mengubah status pipeline.

### 4.3 Hutang Dokumen (Pending Requirements)

- Tabel `pending_requirements` menyimpan syarat yang dilewati: jenis (`po_document`, `bast_signed`, `sph_sent`, dll.), proyek, tanggal override, status (`open`/`resolved`), tanggal resolved.
- Setiap proyek dengan pending requirement `open` menampilkan **badge peringatan kuning** di semua daftar dan detail proyek.
- Dashboard punya panel khusus **"Dokumen Menyusul"**: daftar semua hutang dokumen terbuka, diurutkan dari yang paling lama, dengan umur (hari sejak override) dan tombol langsung ke form pelengkapan.
- Saat dokumen dilengkapi (mis. PO diupload), requirement otomatis `resolved` dan badge hilang.
- Guard penting: proyek **tidak boleh** masuk status `selesai` jika masih ada pending requirement `open`. Ini jaring pengaman terakhir agar arsip selalu lengkap.

### 4.4 Aturan Umum

- Tombol aksi tahap berikutnya disabled untuk staff sampai tahap saat ini selesai; admin melihat opsi override.
- Setiap transisi (normal maupun override) dicatat di `project_status_logs` (siapa, kapan, dari→ke, catatan, flag override).
- Admin bisa membatalkan proyek di tahap manapun (status `dibatalkan`, wajib alasan).
- Admin bisa **mundur satu tahap** (rollback) dengan wajib alasan, untuk koreksi kesalahan input — juga tercatat di log.

## 5. Fitur per Modul

### Fase 1 — Login & Akun + Dashboard

**Login & Akun**
- Login email + password, lupa password via email.
- Admin menambah/menonaktifkan user staff.
- Pengaturan profil perusahaan: nama, logo, alamat, NPWP, info rekening (dipakai di header PDF SPH/Invoice/BAST), format penomoran SPH & invoice.

**Dashboard Proyek**
- Kartu ringkasan: jumlah proyek per status (7 status pipeline + dibatalkan).
- **Panel "Perlu Perhatian"** berisi tiga daftar:
  1. **Dokumen Menyusul** — hutang dokumen dari override (4.3), diurutkan dari terlama.
  2. **Proyek Macet** — tidak ada update > 7 hari di satu tahap (ambang bisa diatur per tahap di pengaturan; mis. `menunggu_po` wajar 14 hari, `bast` 7 hari).
  3. **Piutang Jatuh Tempo** — invoice melewati due date, dengan umur piutang.
- Daftar proyek aktif dengan stepper horizontal posisi tahap + badge override/menyusul bila ada.
- Pencarian & filter: nama proyek, klien, status, rentang tanggal, ada/tidaknya dokumen menyusul.

### Fase 2 — SPH (dengan Revisi) + Pantau Progres

**SPH & Revisi**
- Form SPH: klien (pilih master data atau buat baru), item pekerjaan (deskripsi, qty, satuan, harga satuan, subtotal), PPN opsional, catatan, masa berlaku.
- Nomor otomatis format terkonfigurasi, mis. `SPH/{urut}/{bulan-romawi}/{tahun}`.
- **Revisi SPH:** tombol "Buat Revisi" menduplikasi SPH menjadi revisi baru (`Rev.1`, `Rev.2`, …) via `parent_quotation_id`. Revisi lama otomatis berstatus `superseded` dan hanya bisa dibuka read-only. Proyek selalu menunjuk revisi aktif; nilai proyek mengikuti revisi aktif.
- **Kedaluwarsa:** SPH melewati masa berlaku tanpa PO otomatis berstatus `kedaluwarsa` (via scheduled job harian). Proyek dapat badge "SPH Kedaluwarsa"; opsi: buat revisi baru (perpanjang) atau batalkan proyek.
- Cetak PDF dengan kop perusahaan, mencantumkan nomor revisi.
- Status SPH: Draft, Terkirim, PO Diterima, Superseded, Kedaluwarsa.
- "Tandai Terkirim" → proyek ke `menunggu_po`. Input PO (nomor, tanggal, file) → proyek ke `pengerjaan`.

**Pantau Progres**
- Task/milestone per proyek: judul, deskripsi, deadline, penanggung jawab, status (todo/in progress/done).
- Progress bar otomatis = persentase task selesai.
- Timeline proyek: log kronologis gabungan (transisi status termasuk override, update task, pelengkapan dokumen menyusul, catatan manual).
- Notifikasi in-app (database notification): proyek pindah tahap, task lewat deadline, task baru ditugaskan, **dokumen menyusul berumur > 14 hari** (eskalasi ke admin).
- "Selesaikan Pengerjaan" aktif jika semua task done → `bast`. Admin bisa override bila perlu.

### Fase 3 — BAST + Invoice Termin & Monitoring Pembayaran

**BAST**
- Generate BAST dari data proyek (pihak pertama = perusahaan, pihak kedua = klien, ringkasan pekerjaan dari item SPH revisi aktif).
- Cetak PDF dengan kolom tanda tangan kedua pihak.
- "Ditandatangani" + upload scan → `invoicing`.
- **Kasus BAST tertahan di klien:** admin override ke `invoicing` → requirement `bast_signed` jadi hutang dokumen; scan bisa diupload menyusul.

**Invoice Termin & Monitoring**
- **Satu proyek bisa punya banyak invoice.** Setiap invoice punya `invoice_type`: `dp` (uang muka), `progress` (termin), `pelunasan`, atau `full`.
- Buat invoice dari data SPH/proyek: pilih persentase dari nilai proyek (mis. DP 30%) atau nominal manual; item otomatis terisi dan bisa disesuaikan; nomor otomatis.
- **Invoice DP boleh diterbitkan saat status masih `pengerjaan` atau lebih awal** (fitur, bukan override) — pipeline status baru menuntut invoice di tahap `invoicing`, tapi tidak melarang invoice lebih awal.
- Validasi lunak: total seluruh invoice proyek dibandingkan nilai proyek; jika melebihi, tampilkan peringatan (bukan blokir — ada kasus adendum).
- Cetak PDF dengan info rekening dari pengaturan.
- Catat pembayaran per invoice: tanggal, jumlah, metode, bukti transfer (upload). Mendukung pembayaran bertahap per invoice.
- Status per invoice: Belum Bayar / Sebagian / Lunas. Status proyek → `selesai` saat **total pembayaran semua invoice ≥ total nilai semua invoice** DAN tidak ada pending requirement terbuka.
- Monitoring: daftar invoice outstanding, total piutang, aging (<30, 30–60, >60 hari), rekap piutang per klien.

### Fase 4 — Riwayat, Laporan Manajemen & Ekspor

- Arsip proyek selesai & dibatalkan, read-only lengkap dengan semua dokumen (semua revisi SPH, PO, BAST, semua invoice, bukti bayar) dan riwayat log termasuk override.
- Arsip invoice dengan filter tahun/bulan/klien.
- Ekspor: daftar proyek, rekap invoice/pembayaran, dan rekap piutang ke Excel & CSV.
- **Laporan manajemen:**
  - Nilai proyek per bulan (dibuat vs selesai vs dibayar).
  - Proyek selesai vs dibatalkan per periode.
  - **Durasi rata-rata per tahap pipeline** (dari `project_status_logs`) — mengungkap bottleneck: apakah proyek paling lama nyangkut di `menunggu_po` atau `bast`?
  - **Rekap override per periode** — berapa kali proses menyimpang, di tahap apa, oleh siapa. Sinyal untuk perbaikan proses bisnis.
  - Top klien berdasarkan nilai & kecepatan bayar rata-rata.

## 6. Skema Data (Ringkas)

```
users               (id, name, email, password, role[admin|staff], is_active)
company_settings    (id, name, logo_path, address, npwp, bank_info,
                     sph_format, invoice_format, stuck_thresholds_json)
clients             (id, name, company, address, email, phone)
projects            (id, client_id, name, status, value_total, created_by,
                     cancelled_reason, timestamps, soft_deletes)
project_status_logs (id, project_id, from_status, to_status, user_id, note,
                     is_override, skipped_requirements_json, created_at)
pending_requirements(id, project_id, type, status[open|resolved],
                     created_by, created_at, resolved_at, resolved_by)
quotations          (id, project_id, parent_quotation_id, revision_number,
                     number, date, valid_until, ppn_percent,
                     status[draft|terkirim|po_diterima|superseded|kedaluwarsa],
                     sent_at)
quotation_items     (id, quotation_id, description, qty, unit, unit_price)
purchase_orders     (id, project_id, number, date, file_path)
tasks               (id, project_id, title, description, assignee_id,
                     deadline, status)
bast_documents      (id, project_id, number, date, signed_at, signed_file_path)
invoices            (id, project_id, number, invoice_type[dp|progress|pelunasan|full],
                     date, due_date, total, status[belum|sebagian|lunas],
                     sent_at)
invoice_items       (id, invoice_id, description, qty, unit, unit_price)
payments            (id, invoice_id, date, amount, method, proof_file_path,
                     recorded_by)
```

Catatan relasi kunci:
- `projects` 1:N `quotations` (revisi), tapi hanya satu revisi berstatus aktif (bukan superseded/kedaluwarsa).
- `projects` 1:N `invoices`; kelunasan proyek dihitung agregat.
- `pending_requirements` adalah sumber kebenaran badge peringatan & panel Dokumen Menyusul.

## 7. Non-Fungsional

- Responsif — layout nyaman di HP untuk cek status, update task, dan melengkapi dokumen menyusul dari lapangan.
- Validasi server-side semua form; upload maks 5 MB, tipe PDF/JPG/PNG.
- Soft delete data utama; hanya admin yang mengarsipkan.
- Semua transisi status dijalankan melalui satu service class (`ProjectStateService`) — bukan tersebar di controller — agar aturan gating, override, dan pencatatan log konsisten di satu tempat dan mudah diuji.
- Locale Indonesia untuk tanggal & angka (Rp, dd MMMM yyyy).
- Scheduled job harian: tandai SPH kedaluwarsa, deteksi proyek macet, eskalasi dokumen menyusul > 14 hari.
- Seeder: 1 admin, 2 staff, 3 klien, 6 proyek contoh — termasuk 1 proyek dengan PO menyusul (override aktif), 1 proyek dengan invoice DP + pelunasan, dan 1 SPH dengan 2 revisi.

## 8. Urutan Pengerjaan untuk AI

1. **Fase 1:** Setup Laravel + Breeze + Livewire + Tailwind. Migrasi semua tabel, model + relasi, `ProjectStateService` (gating + override + logging + pending requirements), seeder. Login, manajemen user, pengaturan perusahaan. Dashboard: kartu status, panel Perlu Perhatian (3 daftar), daftar proyek + stepper + badge.
2. **Fase 2:** Modul klien. Modul SPH lengkap: CRUD, revisi (parent/child), penomoran otomatis, PDF, job kedaluwarsa. Transisi SPH→PO→pengerjaan (normal + override PO menyusul). Modul task & progres, timeline, notifikasi.
3. **Fase 3:** Modul BAST (generate, PDF, upload scan, override BAST menyusul). Modul invoice termin (multi-invoice per proyek, tipe DP/progress/pelunasan, PDF). Pencatatan pembayaran, logika kelunasan agregat + guard pending requirements. Monitoring piutang & aging.
4. **Fase 4:** Riwayat/arsip, ekspor Excel/CSV, laporan manajemen (durasi per tahap, rekap override, top klien), polish UI, pengujian alur end-to-end — termasuk skenario override dan pelengkapan dokumen menyusul.

**Pengujian wajib per fase:** alur normal penuh (SPH→selesai), alur PO menyusul, alur invoice DP saat pengerjaan, alur BAST tertahan, dan verifikasi proyek tidak bisa `selesai` selama ada hutang dokumen.
