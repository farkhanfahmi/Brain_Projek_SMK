# T072 — UI+API: Penempatan Siswa Baru ke Kelas (Paste-NISN)

## Depends on
T071 (kolom `jumlahSiswa` di `GET /kelas`)

## Objective
Tambah kolom "Jumlah Siswa" ke tabel Kelas + tombol aksi "Plot Siswa" per baris yang membuka halaman paste-NISN dengan validasi visual real-time (hijau=valid, merah=tidak ditemukan/sudah ada kelas lain).

## Context
- **App:** `apps/api` + `apps/web`
- **Ref:** `Projek/AbsenSI/06-Features/plotting-siswa-kelas.md` — bagian "1️⃣ Penempatan Siswa Baru", baca detail alur lengkap
- **⚠️ Ref WAJIB dibaca sebelum menulis UI:** `Projek/AbsenSI/06-Features/design-system/*.md` — token warna validasi (hijau=`success`, merah=`danger`), radius, dll

## Spec Detail

### API: `POST /students/validate-nisn-batch`
```json
{ "nisns": ["0044360916", "0088825021", "9999999999"] }
```
Response:
```json
{
  "hasil": [
    { "nisn": "0044360916", "status": "valid", "nama": "Putri Rahayu", "studentId": 123 },
    { "nisn": "0088825021", "status": "sudah_ada_kelas_lain", "nama": "Marcello S.", "kelasLain": "XI-RPL-2" },
    { "nisn": "9999999999", "status": "tidak_ditemukan" }
  ]
}
```
- `valid` = NISN ditemukan, `kelasId` masih `null`
- `sudah_ada_kelas_lain` = NISN ditemukan, `kelasId` sudah terisi (kelas APAPUN termasuk kelas tujuan yang sedang diproses — tetap ditandai ini, bukan "valid", karena keputusan final adalah TOLAK bukan auto-pindah)
- `tidak_ditemukan` = NISN tidak match siswa manapun

### API: `POST /kelas/:id/plot-siswa`
```json
{ "studentIds": [123, 456] }
```
- HANYA proses `studentIds` yang dikirim (FE yang filter mana yang statusnya `valid` sebelum kirim — backend TETAP validasi ulang server-side, jangan percaya FE, tolak kalau ada `studentId` yang ternyata `kelasId`-nya sudah terisi saat request ini diproses, untuk cegah race condition)
- Update `kelasId` untuk semua `studentIds` yang lolos validasi ulang
- Response: `{ "berhasil": [...nama+nisn], "gagal": [...] }`
- **WAJIB `@LogActivity`** (sesuai aturan T067) — `action: student.plot_kelas`, catat berapa siswa dipindah

### UI: Tabel Kelas (extend existing)
- Tambah kolom **"Jumlah Siswa"** (dari `jumlahSiswa`, T071)
- Tambah kolom Aksi: tombol **"Plot Siswa"** per baris → navigasi ke `/kelas/[id]/plot-siswa`

### UI: Halaman `/kelas/[id]/plot-siswa` (2 kolom)
**Kolom kiri:**
- Textarea besar, placeholder "Paste NISN, 1 per baris"
- Debounce ~500ms setelah user berhenti mengetik/paste → parse baris (split newline/koma, trim whitespace, buang baris kosong) → panggil `POST /students/validate-nisn-batch`
- Preview list di bawah textarea, tiap baris NISN dengan badge status:
  - `valid` → `StatusBadge` variant `success` (hijau), tampilkan nama siswa
  - `tidak_ditemukan` → `StatusBadge` variant `danger` (merah), teks "NISN tidak ditemukan"
  - `sudah_ada_kelas_lain` → `StatusBadge` variant `danger` (merah), teks "Sudah ada di kelas {nama kelas lain}"
- Tombol **"Input"** — kirim `studentIds` dari baris yang `valid` SAJA ke `POST /kelas/:id/plot-siswa`, disabled kalau tidak ada baris valid
- Setelah submit sukses, textarea+preview **direset kosong** (siap untuk paste batch berikutnya), TIDAK reload halaman

**Kolom kanan:**
- List akumulatif siswa yang berhasil diinput (state React di halaman, bertambah tiap submit sukses dalam sesi ini) — Nama, NISN, waktu (client-side timestamp cukup)

## JANGAN
- ❌ JANGAN proses baris `tidak_ditemukan`/`sudah_ada_kelas_lain` saat klik "Input" — hanya baris `valid`, sesuai keputusan "input yang valid saja, laporkan yang gagal"
- ❌ JANGAN percaya validasi dari FE saat `POST /kelas/:id/plot-siswa` — backend validasi ulang tiap `studentId`, tolak kalau ternyata `kelasId` sudah terisi (race condition antara validasi & submit)
- ❌ JANGAN reload seluruh halaman setelah submit — reset state form saja, list kanan tetap akumulatif
- ❌ JANGAN lupa `@LogActivity` di endpoint plot-siswa

## Files
- **Modifikasi:** `apps/api/src/core/students/students.controller.ts` — tambah `POST /students/validate-nisn-batch`
- **Modifikasi:** `apps/api/src/core/kelas/kelas.controller.ts` — tambah `POST /kelas/:id/plot-siswa`
- **Modifikasi:** halaman tabel Kelas existing — tambah kolom Jumlah Siswa + tombol aksi
- **Buat:** `apps/web/app/(admin)/kelas/[id]/plot-siswa/page.tsx` + komponen terkait

## Acceptance Criteria
- [ ] Paste 3 NISN campuran (1 valid, 1 tidak ditemukan, 1 sudah ada kelas lain) → preview tampil 3 badge warna benar sesuai statusnya
- [ ] Klik Input → hanya siswa valid yang ter-plot, verifikasi via MySQL MCP `kelasId` ter-update
- [ ] Kolom kanan menampilkan hasil akumulatif setelah 2x submit berturut-turut dalam 1 sesi
- [ ] Tabel Kelas menampilkan Jumlah Siswa yang bertambah setelah plot berhasil (refresh halaman kelas)
