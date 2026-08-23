# T194 — API+Web: Import Excel Jam Pelajaran + Download Template

## Depends on
**REKOMENDASI setelah T187** (infrastruktur `ImportDialog`+`templateEndpoint` reusable). **BERGANTUNG pada T158** (SUDAH SELESAI — model `JamPelajaranOption`/`Slot`/dst sudah ada).

## Objective
Halaman Jam Pelajaran (T158) — tambah **Import Excel** dengan **template downloadable** untuk mengisi SEMUA slot 1 Opsi Jam Pelajaran sekaligus — SAAT INI hanya bisa input manual baris-per-baris di form (yang untuk 13+ jam pelajaran × beberapa hari SANGAT LAMBAT diisi manual).

## Context — Temuan Riset (2026-08-15) & Perubahan Keputusan Sebelumnya

**PENTING — task T174 LAMA (Import CSV modul lain) SECARA EKSPLISIT MENGECUALIKAN Jam Pelajaran** dengan alasan "strukturnya kompleks (banyak baris per hari dalam 1 opsi), bukan record flat sederhana". **User SEKARANG MEMINTA fitur ini SECARA EKSPLISIT** — jadi keputusan lama itu DIBATALKAN/DIREVISI oleh permintaan baru ini, BUKAN diabaikan. Task ini WAJIB menyelesaikan tantangan struktur kompleks yang sebelumnya jadi alasan pengecualian.

`(admin)/jam-pelajaran/` — 4 model: `JamPelajaranOption` (1 opsi), `JamPelajaranOptionTingkat` (cakupan tingkat), `JamPelajaranAktivasi` (status aktif per tingkat), `JamPelajaranSlot` (banyak baris per hari: jamKe, jamMulai, jamSelesai, keterangan, urutan — TERMASUK baris istirahat `jamKe: null`).

## Spec Detail

### 1. Format Excel — 1 file = 1 Opsi Jam Pelajaran lengkap (BUKAN flat sederhana biasa)

**KEPUTUSAN STRUKTUR FORMAT** — BEDA dari import lain (yang 1 baris = 1 record independen), format ini 1 FILE = 1 OPSI dengan BANYAK baris slot:
- Kolom: `hari` (nama hari atau angka DAYOFWEEK, KONSISTEN `Schedule.hari`), `jam_ke` (angka, KOSONGKAN untuk baris istirahat), `jam_mulai` (HH:mm), `jam_selesai` (HH:mm), `keterangan` (opsional, misal "Istirahat Ke-1"), `urutan` (angka posisi kronologis dalam hari itu).
- **Metadata opsi** (nama opsi, cakupan tingkat) — TIDAK BISA masuk kolom flat yang SAMA dengan slot (beda level data) — PUTUSKAN: SHEET TERPISAH dalam file Excel yang SAMA (sheet 1 = "Info Opsi" dengan nama+cakupan tingkat, sheet 2 = "Slot" dengan semua baris jam) — REKOMENDASI KUAT pendekatan ini, KONSISTEN dengan sifat Excel yang mendukung multi-sheet secara native, LEBIH JELAS daripada memaksa metadata jadi kolom berulang di setiap baris slot.

### 2. Backend — parsing multi-sheet + validasi struktural

- `ImportService`/modul `JamPelajaranService` — method baru `importOpsiJamPelajaran(buffer)` — PARSE KEDUA sheet via `exceljs` (REUSE library yang sudah ada, BUKAN `csv-parse` yang cuma 1-dimensi):
  1. Sheet "Info Opsi" — baca nama opsi + cakupan tingkat (semua/spesifik).
  2. Sheet "Slot" — baca SEMUA baris, GROUP BY `hari`, urutkan by `urutan`.
- Validasi PER BARIS slot: `jam_mulai`/`jam_selesai` format HH:mm valid, `jam_ke` boleh kosong (istirahat) TAPI kalau kosong WAJIB ada `keterangan` (supaya jelas baris itu memang istirahat, bukan salah isi) — KONSISTEN validasi yang SUDAH ADA di form manual `jam-pelajaran-view.tsx` (REPLIKASI, jangan buat validasi baru berbeda).
- Create OPSI BARU (bukan update opsi existing — import ini untuk MEMBUAT opsi baru dari nol, KONSISTEN pola import lain yang selalu CREATE, TOLAK duplikat nama opsi kalau relevan) via `JamPelajaranService.create()` YANG SUDAH ADA (REUSE method existing, JANGAN duplikasi logic transaksi create+slot+tingkatScopes).
- Endpoint download template: `GET /import/jam-pelajaran/template` — file Excel 2-SHEET dengan CONTOH DATA LENGKAP (REKOMENDASI KUAT: isi contoh dari referensi Excel sekolah asli yang PERNAH diberikan user untuk T158 — "Sistem Jurnal Baru 2026-2027.xlsx" sheet Alokasi Waktu, Senin-Kamis 13 jam + istirahat, Jumat 7 jam + istirahat — supaya admin punya CONTOH NYATA bukan dummy generik).

### 3. Frontend — `ImportDialog` khusus (kemungkinan BUTUH VARIAN, bukan reuse polos)

- `jam-pelajaran-view.tsx` — pasang import Excel — TAPI VERIFIKASI apakah `ImportDialog` GENERIK (dari T187) CUKUP untuk kasus ini (report hasil per-BARIS SLOT mungkin kurang bermakna dibanding report per-OPSI keseluruhan — 1 file = 1 opsi, jadi hasil sukses/gagal lebih masuk akal di level OPSI bukan per baris) — PERTIMBANGKAN varian pesan hasil KHUSUS untuk import ini (1 opsi berhasil dibuat dengan N slot, ATAU gagal dengan alasan spesifik) DAAMPAK dari `ImportDialog` generic yang mengasumsikan report per-baris seperti import lain.

## Edge Cases
- File dengan sheet HANYA 1 (lupa sheet Info Opsi ATAU Slot) — TOLAK dengan pesan jelas menyebut sheet mana yang hilang.
- Baris slot dengan `hari` yang tidak valid (bukan Senin-Jumat, atau Sabtu — KONSISTEN aturan proyek Sabtu bukan hari wajib berjadwal formal, TAPI VERIFIKASI apakah Jam Pelajaran BOLEH ada Sabtu untuk kasus ekstra/khusus — PUTUSKAN saat implementasi, klarifikasi user kalau ragu) — TOLAK atau IZINKAN sesuai keputusan.
- Import GAGAL DI TENGAH (misal baris ke-30 dari 50 error) — SELURUH opsi TIDAK DIBUAT SAMA SEKALI (all-or-nothing, BEDA dari import lain yang best-effort per-baris independen — KARENA di sini SEMUA baris adalah BAGIAN dari 1 opsi yang sama, opsi setengah-jadi TIDAK BERGUNA/BERBAHAYA) — WAJIB transaksi penuh, rollback total kalau ADA error di baris manapun.

## Files
- **Modifikasi:** `apps/api/src/jam-pelajaran/jam-pelajaran.service.ts` (method import baru, REUSE `create()` existing), `apps/api/src/import/import.controller.ts` ATAU controller jam-pelajaran sendiri (endpoint import+template — PUTUSKAN lokasi paling konsisten), `apps/web/src/app/(admin)/jam-pelajaran/jam-pelajaran-view.tsx` (tombol import+download template).
- **Jangan sentuh:** `JamPelajaranService.create()`/`activate()`/`update()`/`delete()` (REUSE `create()` apa adanya, TIDAK diubah).

## Acceptance Criteria
- [x] Download template Excel 2-sheet dengan contoh data NYATA (referensi Excel sekolah asli).
- [x] Upload Excel terisi lengkap — 1 Opsi Jam Pelajaran baru berhasil dibuat dengan SEMUA slot (termasuk baris istirahat) dalam 1 transaksi.
- [x] Import GAGAL sebagian (error di 1 baris) → SELURUH opsi TIDAK dibuat (all-or-nothing) — verified via unit test (create() tidak pernah dipanggil kalau parsing gagal di baris manapun).
- [x] Sheet hilang/salah nama — ditolak dengan pesan jelas.
- [x] Build + type-check hijau, jest baru untuk parsing multi-sheet + all-or-nothing.

## Validasi Claudian
- [x] **Konfirmasi** import ini all-or-nothing (BEDA dari pola best-effort import lain) — dijelaskan di komentar kode + laporan di bawah.
- [x] Konfirmasi REUSE `JamPelajaranService.create()` existing untuk proses create akhir (bukan duplikasi logic transaksi) — `importOpsi()` HANYA parsing+validasi, lalu satu panggilan `this.create(dto, ...)`.
- [x] Konfirmasi template Excel berisi CONTOH DATA NYATA dari referensi sekolah — diekstrak langsung dari `dariDev/Sistem Jurnal Baru 2026-2027.xlsx` sheet "Alokasi Waktu" (bukan dummy).

## Status Eksekusi (2026-08-16)

**Selesai.**

### Keputusan desain

- **All-or-nothing dijelaskan eksplisit**: `importOpsi()` melakukan SELURUH parsing+validasi (2 sheet, semua baris slot) SEBELUM memanggil `create()` sama sekali — kalau ada error di baris manapun, exception dilempar dari parsing dan `create()` TIDAK PERNAH terpanggil. `create()` sendiri juga sudah transactional secara internal (lapis kedua). Beda dari import lain (best-effort per-baris) karena semua baris slot di sini BAGIAN dari 1 opsi yang sama — opsi setengah-jadi tidak berguna/berbahaya dipakai jadwal.
- **Constraint Sabtu/Minggu**: diverifikasi form manual (`jam-pelajaran-form-sheet.tsx`, `HARI_LIST = [2,3,4,5,6]`) HANYA mendukung Senin-Jumat — import mengikuti batasan yang SAMA (Sabtu/Minggu ditolak), bukan aturan baru.
- **Constraint "jam_ke kosong wajib keterangan"** — spec mengasumsikan ini SUDAH ADA di form manual, tapi riset kode (`jam-pelajaran-form-sheet.tsx`) TIDAK menemukan validasi ini (cuma placeholder hint "Istirahat Ke-1", bukan enforcement). Import MENGIKUTI perilaku manual yang sebenarnya (keterangan tetap opsional walau jam_ke kosong), bukan menambah aturan baru yang tidak ada di form aslinya.
- Lokasi endpoint: `JamPelajaranController` (bukan `ImportController`) — sesuai opsi yang ditawarkan spec, supaya `create()` bisa direuse langsung tanpa cross-module call.

### Backend

- `JamPelajaranService.importOpsi(buffer, actorId, ipAddress)` — return `JamPelajaranImportResult` (`success`, `optionId`, `optionNama`, `slotCount`, `reason`) BUKAN `ImportReport` per-baris (tipe baru di `@absensi/types`, alasan didokumentasikan di komentar tipe).
- `parseImportWorkbook()` — baca sheet "Info Opsi" (key-value: nama, semuaTingkat, tingkat dipisah koma) + sheet "Slot" (kolom `hari/jam_ke/jam_mulai/jam_selesai/keterangan/urutan`), validasi HH:mm, hari Senin-Jumat saja, urutan/jam_ke integer positif.
- `generateImportTemplate()` — 2 sheet, data Senin-Kamis (13 jam+2 istirahat) dan Jumat (7 jam+1 istirahat) diekstrak PERSIS dari referensi Excel sekolah asli (jam mulai/selesai per baris dibaca langsung dari file, bukan diketik ulang manual).
- `POST /jam-pelajaran/import` + `GET /jam-pelajaran/import/template` — guard `super_admin` (konsisten `create()` yang juga super_admin only).
- 9 unit test baru (file valid, cakupan tingkat parsial, sheet hilang ×2, hari Sabtu ditolak, format jam salah, kolom hilang, nama kosong, template generation) — 463/463 pass di seluruh suite (1 suite lain, `schedule-config.service.spec.ts`, gagal compile karena perubahan EKSTERNAL tidak terkait yang sedang berjalan paralel di sesi lain, bukan bagian T194).

### Frontend

- `JamPelajaranImportDialog` — komponen BARU terpisah (bukan reuse `ImportDialog` generik), sesuai analisis spec sendiri bahwa report per-opsi tidak cocok dengan bentuk `ImportReport` per-baris. UI: download template, upload, hasil sukses (nama opsi + jumlah slot) ATAU gagal (1 alasan), TIDAK ADA tabel error per-baris (tidak relevan untuk all-or-nothing).
- `jam-pelajaran-view.tsx` — tombol "Import Excel" dipasang di sebelah "Buat Opsi Baru".
- Tidak ada duplikat route admin-jurnal untuk halaman ini (hanya `(admin)/jam-pelajaran/`), jadi tidak perlu wiring kedua.

### Verifikasi

- `tsc --noEmit` bersih di `apps/api` dan `apps/web` (kecuali error pre-existing tidak terkait di `dispen-view.tsx`).
- `jest apps/api`: 463/463 pass (28/29 suite, 1 suite lain gagal compile karena perubahan eksternal tidak terkait — lihat di atas).
- Live-verify browser: belum dilakukan (konsisten pola T186-T193, verifikasi manual diserahkan ke user).
