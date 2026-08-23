# T213 — API+Web: Import CSV/Excel Jadwal (Menggantikan T160, Format Baru untuk JadwalSlot)

## Depends on
**WAJIB SETELAH T204, T206, T207, T208** (butuh `JadwalSlot`, `MapelGuru`, validasi bentrok, dan Date Generator sudah ada). Bagian dari rangkaian T203-T215.

## Objective
Import CSV/Excel Jadwal Mengajar (SAAT INI `ImportService.importSchedules`, T160, target `Schedule` LAMA) — DIBUAT ULANG untuk target `JadwalSlot` BARU. Template diperbaiki sesuai user ("perbaiki juga template import excelnya").

## Konteks — Masalah di Format LAMA yang WAJIB Diperbaiki

Riset menemukan `ImportService.importSchedules()` LAMA (T160) punya **side-effect tersembunyi berbahaya**: kolom `minggu` terisi → DIAM-DIAM mengubah `Kelas.modeJadwal` jadi `blok` tanpa konfirmasi eksplisit admin (`import.service.ts` baris ~838-857). **INI TIDAK RELEVAN LAGI** di arsitektur baru (mode sekarang atribut Opsi Jadwal, BUKAN Kelas) — TAPI JADIKAN PELAJARAN: import BARU ini **JANGAN PERNAH mengubah data lain secara diam-diam** di luar apa yang eksplisit ada di kolom CSV — SEMUA efek samping harus AGAR JELAS/eksplisit dikonfirmasi, bukan implisit.

## Spec Detail

### 1. Format kolom CSV/Excel BARU

- **1 file = 1 Opsi Jadwal** (WAJIB pilih Opsi Jadwal target SEBELUM upload, endpoint terima `opsiJadwalId` sebagai parameter terpisah dari file — BUKAN kolom di CSV, karena admin sudah masuk konteks 1 Opsi Jadwal spesifik saat mengimport dari Workspace T212, KONSISTEN alur "template import utama TIDAK berdasarkan guru/kelas" yang disebut user — import ini mengisi BANYAK kelas+guru SEKALIGUS dalam 1 file, bukan dipecah per entitas).
- Kolom: `kelas` (nama kelas, wajib), `hari` (nama hari, wajib), `jam_ke` (angka, wajib), `minggu` (A/B, WAJIB kalau Opsi Jadwal ini mode Blok, HARUS KOSONG kalau mode Normal — validasi ketat sesuai mode Opsi target), `mapel` (nama mapel, wajib), `guru` (SATU ATAU LEBIH nama guru dipisah koma/titik-koma untuk team-teaching, wajib minimal 1), `catatan` (opsional).
- **VALIDASI PER BARIS** (REUSE method validasi yang SAMA PERSIS dengan `JadwalSlotService.create()` T206, JANGAN duplikasi logic terpisah — best-effort per baris, KONSISTEN pola import lain di proyek):
  1. Kelas ditemukan (lookup by nama).
  2. `jamKe` valid untuk `hari` itu di Alokasi Waktu Opsi Jadwal target.
  3. `minggu` sesuai mode Opsi Jadwal target (kosong utk normal, terisi utk blok).
  4. Mapel ditemukan.
  5. SETIAP nama guru di kolom `guru` (split koma) ditemukan DAN terdaftar `MapelGuru` untuk Mapel baris itu — TOLAK baris kalau ADA SATU guru saja yang tidak terdaftar (sebutkan nama guru mana).
  6. Bentrok — SAMA seperti create() manual, REPLIKASI logic tapi track kandidat IN-MEMORY untuk baris-baris DALAM FILE yang sama (KONSISTEN pola T160 lama `bentrokCandidates`).
  7. Duplikat kelas-hari-jamKe-minggu DALAM FILE yang sama — TOLAK baris kedua sebagai duplikat.

### 2. Backend — endpoint import + template

- `POST /jadwal-slot/import?opsiJadwalId=` — best-effort per baris (KONSISTEN pola import lain, BUKAN all-or-nothing seperti T194 Jam Pelajaran — karena di sini SETIAP BARIS independen, 1 baris gagal TIDAK merusak baris lain, BEDA dari T194 yang 1 file = 1 struktur utuh yang saling bergantung).
- `GET /jadwal-slot/import/template?opsiJadwalId=` — template Excel dengan CONTOH DATA sesuai mode Opsi Jadwal target (kalau target mode Blok, contoh WAJIB isi kolom minggu; kalau Normal, contoh kosongkan kolom minggu — template ADAPTIF sesuai konteks, BUKAN template statis generik).

### 3. Frontend — tombol import di Workspace (T212)

- Di dalam Workspace (`T212`), tambah tombol "Import CSV/Excel" (REUSE `ImportDialog` dengan `templateEndpoint`, T187) — di level Opsi Jadwal (BUKAN di dalam tab hari tertentu, karena 1 file bisa isi BANYAK hari sekaligus) — hasil import langsung REFRESH tampilan Section Kelas dan Guru (data baru langsung kelihatan tanpa reload manual).

## Edge Cases
- File CSV/Excel dengan campuran hari DAN kelas yang BERBEDA-BEDA dalam 1 file — WAJAR dan DIDUKUNG (ini keunggulan utama import dibanding input manual satu-satu).
- Import ke Opsi Jadwal yang MODE-nya salah (misal target Normal tapi banyak baris isi kolom minggu) — SEMUA baris yang isi minggu DITOLAK dengan pesan jelas ("Opsi Jadwal ini mode Normal, kolom minggu harus dikosongkan"), baris LAIN yang benar tetap diproses (best-effort).

## Files
- **Buat:** `apps/api/src/jadwal-slot/` tambah method `importJadwalSlot()`+`generateImportTemplate()` (REUSE validasi T206, JANGAN duplikasi), endpoint controller baru.
- **Modifikasi:** komponen Workspace T212 (tombol import).
- **Jangan sentuh:** `ImportService.importSchedules()` LAMA (T160, TIDAK dihapus sampai T215).

## Acceptance Criteria
- [x] Import berhasil untuk baris valid, tolak baris invalid dengan pesan jelas (best-effort, bukan all-or-nothing).
- [x] Template ADAPTIF sesuai mode Opsi Jadwal target (kolom minggu ada/tidak contoh sesuai konteks).
- [x] SEMUA validasi (T206) di-reuse persis, tidak ada validasi baru yang berbeda dari create() manual.
- [x] ~~Team-teaching (banyak guru per baris, dipisah koma) berfungsi~~ — **REVISI, dikonfirmasi user 2026-08-17**: kolom `guru` HANYA 1 nama, KONSISTEN keputusan T209 (1 JadwalSlot = TEPAT 1 guru, `MapelGuru` cuma filter dropdown). Spec asli T213 ditulis SEBELUM T209 dan sudah usang di poin ini — baris dengan >1 nama guru (dipisah koma/titik-koma) DITOLAK eksplisit dengan pesan jelas, bukan diam-diam ambil yang pertama.
- [x] Build + type-check hijau, jest untuk skenario best-effort campuran.

## Validasi Claudian
- [x] **WAJIB konfirmasi** import ini TIDAK punya efek samping tersembunyi apa pun di luar yang eksplisit ada di kolom CSV (pelajaran dari bug T160 lama) — dikonfirmasi: hanya `jadwalSlot.create`+`jadwalSlotGuru.create` per baris via `create()`, tidak ada update ke Kelas/Mapel/entity lain.
- [x] Konfirmasi REUSE validasi JadwalSlotService.create() (T206), bukan duplikasi logic terpisah yang berisiko divergen dari waktu ke waktu — `importJadwalSlot()` memanggil `this.create()` APA ADANYA per baris (try/catch untuk best-effort), TIDAK reimplementasi 1 pun dari 5 validasi.

## Implementasi (2026-08-17)

**Koreksi scope vs spec asli**: kolom `guru` di CSV HANYA terima 1 nama (bukan multi-guru "team-teaching" seperti tertulis di spec awal) — dikonfirmasi eksplisit via AskUserQuestion sebelum implementasi, karena T209 (task sebelumnya, sama hari) sudah mengoreksi arsitektur ke 1 guru per slot. Baris >1 nama guru (koma/titik-koma) ditolak dengan pesan jelas ("kolom guru hanya boleh 1 nama... pisahkan jadi baris terpisah"), bukan silent-take-first (pelajaran dari bug T160 yang sama persis mau dihindari task ini).

**Backend** (`apps/api/src/jadwal-slot/jadwal-slot.service.ts`):
- `importJadwalSlot(opsiJadwalId, buffer, filename, actorId, ipAddress)` — parse CSV/xlsx (helper `parseImportRows()`, KONSISTEN pola `ImportService.parseRows()` T187: xlsx by-extension pakai ExcelJS, else csv-parse), validasi format per kolom (kelas/hari/jam_ke/minggu/mapel/guru lookup by nama, ambigu >1 match ditolak), lalu panggil `this.create()` APA ADANYA per baris valid di try/catch (best-effort, 1 baris gagal tidak menghentikan baris lain) — SEMUA 5 validasi T206 (minggu-sesuai-mode, jamKe valid, duplikat, guru-terdaftar-mapel, bentrok) otomatis ikut tanpa duplikasi logic.
- `generateImportTemplate(opsiJadwalId)` — cek `OpsiJadwal.mode`, contoh baris isi kolom minggu HANYA kalau Blok.
- 1 activity log RINGKASAN per operasi import (`jadwal_slot.import`, manual `ActivityLogService.record()` bukan `@LogActivity` — KONSISTEN aturan CLAUDE.md utk operasi bulk).
- Endpoint baru di `jadwal-slot.controller.ts`: `POST /jadwal-slot/import?opsiJadwalId=`, `GET /jadwal-slot/import/template?opsiJadwalId=` (route `import/template` WAJIB didaftarkan SEBELUM `@Get(":id")` — kalau tidak, Nest match "import" sebagai :id duluan berdasar urutan registrasi).
- 6 test baru: baris valid sukses, best-effort campuran (1 gagal 1 sukses), guru multi-nama ditolak, mode-mismatch minggu ditolak (reuse validasi create()), guru tidak ditemukan, activity log ringkasan. Total 565/565 test lulus.

**Frontend**: tombol "Import CSV/Excel" ditambah di Workspace (T212) `workspace-view.tsx`, level Opsi Jadwal (bukan di dalam tab hari), reuse `ImportDialog` (T187) apa adanya dengan `templateEndpoint`+`onImported={refetchSlots}` — hasil import langsung refresh Section Kelas/Guru tanpa reload manual. Fix kecil di `proxy-upload/[...path]/route.ts`: forward query string (`request.nextUrl.search`) yang sebelumnya di-drop — dibutuhkan supaya `opsiJadwalId` sampai ke backend saat upload lewat proxy (tidak mengubah perilaku endpoint import lain yang belum pernah butuh query param).

**Verifikasi**: tsc api+web bersih, `nest build` bersih, `next build` bersih, jest 565/565 (naik dari 557, +6+2 test baru). Live browser verify TIDAK dilakukan (kredensial login gagal, dikonfirmasi user sebelumnya lanjut tanpa live-verify), dev server direstart bersih pasca `next build`.
