# T006 — Import CSV: Siswa & Guru

## Depends on
T005 (tabel students & teachers harus ada, API kelas/jurusan sudah jalan)

## Objective
Buat fitur upload CSV untuk import massal data siswa dan guru dengan validasi per-baris, partial commit, dan laporan hasil yang bisa di-download.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `students`, `teachers`
- **Role akses:** `super_admin` saja
- **Ref:** `Projek/AbsenSI/06-Features/import-data-master.md`

## Spec Detail

### Install:
```
pnpm add multer csv-parse --filter api
pnpm add -D @types/multer --filter api
```

### API Endpoints:

**POST `/import/students`** (multipart/form-data, field: `file`)
- Parse CSV baris per baris
- **Kolom wajib CSV:** `nisn`, `nama`, `kelas`, `jurusan`
- **Kolom opsional:** `tanggal_lahir` (format: YYYY-MM-DD)
- Validasi per baris:
  - `nisn` tidak boleh kosong, harus unik (cek di DB)
  - `nama` tidak boleh kosong
  - `kelas` harus match nama kelas yang sudah ada di DB
  - `jurusan` harus match nama jurusan yang sudah ada di DB
  - Kombinasi `kelas` + `jurusan` harus valid (kelas tersebut memang jurusan itu)
- **Partial commit:** baris valid masuk ke DB meskipun ada baris lain yang gagal
- Response:
```json
{
  "total": 100,
  "berhasil": 95,
  "gagal": 5,
  "gagal_detail": [
    { "baris": 3, "nisn": "123", "alasan": "NISN sudah terdaftar" },
    { "baris": 7, "nisn": "456", "alasan": "Kelas 'X-RPL-99' tidak ditemukan" }
  ]
}
```

**POST `/import/teachers`** (sama strukturnya)
- **Kolom wajib CSV:** `nip`, `nama`
- Validasi: `nip` unik

**GET `/import/template/students`** — download file CSV template kosong dengan header yang benar
**GET `/import/template/teachers`** — sama untuk guru

### Admin UI (`apps/web/app/(admin)/import/page.tsx`):

Satu halaman dengan 2 tabs: **Siswa** | **Guru**

Per tab:
1. Tombol "Download Template CSV" (panggil endpoint template)
2. Area upload file (drag & drop atau klik pilih file) — hanya `.csv`
3. Tombol "Mulai Import" — disable sebelum file dipilih
4. Setelah upload: tampilkan hasil
   - Kalau semua berhasil: banner hijau "95 siswa berhasil diimport"
   - Kalau ada yang gagal: banner kuning + tabel error (no. baris, NISN/NIP, alasan) + tombol "Download Laporan Gagal (CSV)"
5. Loading state selama proses (bisa lama kalau file besar)

### Format file laporan gagal (downloadable):
CSV dengan header: `baris,nisn_atau_nip,nama,alasan`

## JANGAN
- ❌ JANGAN auto-create kelas/jurusan yang belum ada — kalau tidak ditemukan, baris tersebut GAGAL dengan pesan jelas. User harus input kelas/jurusan manual dulu via T004
- ❌ JANGAN buat fitur "undo" import — sudah diputuskan tidak ada undo
- ❌ JANGAN proses seluruh CSV di memory kalau file besar — gunakan streaming/chunking via `csv-parse` dengan mode stream
- ❌ JANGAN izinkan upload selain `.csv` — validasi ekstensi dan MIME type
- ❌ JANGAN simpan file CSV yang diupload ke disk server — proses langsung dari buffer, tidak perlu storage permanen
- ❌ JANGAN buat endpoint import kartu di task ini — itu T008 (import CSV kartu) dan T009 (tap-to-assign)

## Files
- **Buat:** `apps/api/src/import/import.module.ts`
- **Buat:** `apps/api/src/import/import.service.ts` (logic parse + validasi + commit)
- **Buat:** `apps/api/src/import/import.controller.ts`
- **Buat:** `apps/web/app/(admin)/import/page.tsx`

## Acceptance Criteria
- [ ] Upload CSV 100 siswa valid → semua masuk DB, response `berhasil: 100, gagal: 0`
- [ ] Upload CSV dengan 3 baris NISN duplikat → 97 masuk, 3 gagal dengan alasan "NISN sudah terdaftar"
- [ ] Upload CSV dengan kelas yang tidak ada → baris tersebut gagal, siswa lain tetap masuk
- [ ] Tombol "Download Template" menghasilkan file CSV dengan header yang benar
- [ ] Tombol "Download Laporan Gagal" muncul dan bisa didownload kalau ada baris gagal
- [ ] Upload file bukan CSV → error "Hanya file .csv yang diterima"

## Handoff ke T007
T007 (Card CRUD) tidak bergantung langsung pada T006, tapi secara praktis card baru bisa di-assign ke siswa/guru yang sudah ada. Pastikan setelah T006 ada data siswa yang cukup untuk testing T007–T009.
