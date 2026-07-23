# T049 — UI: Halaman Sesi — Jurnal, Presensi Kelas & Form Izin

## Depends on
T045 (API jurnal & presensi), T046 (API izin guru), T048 (halaman utama guru, sumber navigasi ke sini)

## Objective
Buat halaman detail 1 sesi mengajar tempat guru mengisi jurnal, mengoreksi presensi siswa, dan (kalau berlaku) mengisi form tugas titipan untuk sesi yang diizinkan.

## Context
- **App:** `apps/web`
- **Route:** `/guru/sesi/:sessionId` (halaman jurnal) dan `/guru/izin/:permitId` (halaman/modal form tugas titipan — bisa 1 halaman terpisah atau modal, keduanya diterima, pilih yang lebih konsisten dengan pola modal existing seperti `IzinModal` di dashboard piket T023)
- **Role:** `guru`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Jurnal Mengajar — Field" dan "Izin Guru Tidak Mengajar" langkah 4
- **⚠️ Ref WAJIB dibaca sebelum menulis UI:** `Projek/AbsenSI/06-Features/design-system/MASTER.md` + companion files (`01-colors.md`, `03-components.md`) — token warna badge di task ini WAJIB ikut palet situ (hanya ada `success`/`danger`/`primary`, TIDAK ada "hijau"/"merah" generik di luar token itu)

## Spec Detail

### Halaman `/guru/sesi/:sessionId`
Fetch `GET /teaching-sessions/:sessionId/detail` (T045) saat load.

**Bagian atas — info sesi:**
- Kelas, mapel, jam, status (badge sama seperti T048), waktu mulai aktual jika sudah dimulai

**Bagian tengah — form jurnal:**
- 4 textarea: Materi/Topik, Tujuan Pembelajaran, Tugas/Penilaian, Catatan/Kendala
- Autosave dengan debounce 2 detik setelah berhenti mengetik (`PATCH /teaching-sessions/:id/journal`), tampilkan indikator kecil "Tersimpan" setelah berhasil — **jangan** butuh tombol submit eksplisit untuk textarea ini, pengalaman seperti Google Docs (autosave)
- Kalau sesi `status: selesai` (closed) — form tetap bisa diedit untuk saat ini (batas waktu edit setelah closed masih open question di spec, JANGAN buat read-only sendiri tanpa instruksi eksplisit)

**Bagian bawah — tabel presensi siswa:**
| Nama | Status | Aksi |
|---|---|---|
| Budi Santoso | Hadir (Badge Pill: bg `--color-success-bg`, text `--color-success-text`, `03-components.md`) | [Tandai Tidak Ada] |
| Ani Wijaya | Tidak Ada di Kelas (Badge Pill: bg `--color-danger-bg`, text `--color-danger-text`) | [Batalkan / Tandai Hadir] |

Tombol aksi pakai Secondary Button spec (`03-components.md`) — bg putih, border `1px solid --color-border-subtle`, `radius-full`. Tabel row pakai `text-body` (14px), header kolom `text-label` (13px, `--color-text-secondary`).

- Klik "Tandai Tidak Ada" → langsung `PATCH /teaching-sessions/:id/attendance` dengan `{ marks: [{ student_id, status: "tidak_ada_di_kelas" }] }`, update baris tanpa reload
- Klik "Batalkan" (untuk siswa yang sudah ditandai) → kirim `{ status: "hadir" }`, baris kembali ke default

**Catatan `sudah_tap_gerbang`:** tampilkan indikator kecil (ikon) di baris siswa yang `sudah_tap_gerbang: false` — informatif saja, **bukan** filter atau alasan otomatis menandai `tidak_ada_di_kelas` (sesuai T045: default selalu hadir)

### Halaman/Modal Form Tugas Titipan (untuk sesi berstatus "Diizinkan")
Diakses dari tombol "Izin" di T048 (halaman utama).

**Form:**
- Info: nama guru, tanggal, kelas+mapel yang diizinkan (atau "Semua sesi hari ini" kalau izin seharian)
- Upload file (drag-drop atau file picker) — tampilkan tipe yang diterima: PDF, JPG, PNG, DOCX, maks 10MB
- Textarea "Keterangan Tugas Titipan" (wajib diisi, tidak boleh kosong kalau ada file — atau bisa kosong kalau guru hanya mau isi teks tanpa file, keduanya opsional independent tapi minimal salah satu harus diisi)
- Tombol "Simpan" → `POST /teacher-permits/:id/submit-tugas` (multipart)
- Setelah submit sukses → tampilkan konfirmasi, kembali ke halaman utama guru

## JANGAN
- ❌ JANGAN buat tombol submit terpisah untuk tiap field jurnal — pakai autosave debounce untuk textarea, sesuai UX yang diminta (mengurangi friksi, guru sering terburu-buru saat mengajar)
- ❌ JANGAN buat form tugas titipan bisa diakses untuk sesi yang belum `sudah_diizinkan` — validasi ganda di FE (disable akses) meski backend juga sudah validasi
- ❌ JANGAN reload seluruh halaman saat update 1 baris presensi siswa — update state lokal + panggil API, konsisten dengan pola UI lain di codebase (lihat T023 dashboard piket sebagai referensi update parsial)
- ❌ JANGAN buat validasi "wajib isi semua field jurnal sebelum bisa keluar halaman" — semua field jurnal sifatnya opsional/best-effort, tidak ada requirement wajib lengkap sebelum sesi bisa closed

## Files
- **Buat:** `apps/web/app/(guru)/guru/sesi/[sessionId]/page.tsx`
- **Buat:** `apps/web/app/(guru)/guru/sesi/[sessionId]/components/journal-form.tsx`
- **Buat:** `apps/web/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx`
- **Buat:** `apps/web/app/(guru)/guru/izin/[permitId]/page.tsx` (atau komponen modal, sesuai keputusan implementasi)
- **Buat:** `apps/web/lib/use-session-detail.ts` — hook fetch + mutate detail sesi

## Acceptance Criteria
- [ ] Buka halaman sesi yang baru di-start → form jurnal kosong, semua siswa default "Hadir"
- [ ] Ketik di textarea materi, tunggu 2 detik → indikator "Tersimpan" muncul, refresh halaman → data tetap ada
- [ ] Klik "Tandai Tidak Ada" pada 1 siswa → badge berubah tanpa reload halaman
- [ ] Klik "Batalkan" pada siswa yang sudah ditandai → kembali ke "Hadir"
- [ ] Buka form tugas titipan dari sesi yang diizinkan → upload PDF + isi keterangan → submit sukses → kembali ke `/guru`
- [ ] Upload file dengan tipe tidak didukung → pesan error jelas sebelum/saat submit, tidak lolos ke backend dengan silent fail
