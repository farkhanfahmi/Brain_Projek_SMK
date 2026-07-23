# T045 — API: Jurnal Mengajar & Presensi Kelas (Koreksi Manual Siswa)

## Depends on
T043 (sesi harus bisa di-start dulu sebelum jurnal/presensi diisi)

## Objective
Buat endpoint untuk guru mengisi jurnal (materi, tujuan pembelajaran, tugas/penilaian, catatan) dan mengoreksi daftar hadir siswa per sesi (default semua hadir dari data tap gerbang, guru tandai manual yang tidak ada di kelas).

## Context
- **App:** `apps/api`
- **Tables:** `journal_entries`, `class_attendance_marks`, `teaching_sessions`, `students`, `attendance_records` (untuk cek siapa yang tap gerbang)
- **Role:** `guru`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Jurnal Mengajar — Field" & poin 6-7 "Konsep Inti"

## Spec Detail

### API: `GET /teaching-sessions/:sessionId/detail`
- Auth: `guru`, verifikasi kepemilikan sesi (`teaching_sessions.teacher_id === req.user.teacher_id`) → 403 kalau bukan miliknya
- Response:
```json
{
  "session_id": 501,
  "kelas": "XI-RPL-1",
  "mapel": "Pemrograman Web",
  "status": "sedang_berlangsung",
  "started_at": "2026-07-21T07:08:00Z",
  "closed_at": null,
  "jurnal": {
    "materi": null,
    "tujuan_pembelajaran": null,
    "tugas_penilaian": null,
    "catatan": null
  },
  "siswa": [
    { "student_id": 1001, "nama": "Budi Santoso", "status": "hadir", "sudah_tap_gerbang": true },
    { "student_id": 1002, "nama": "Ani Wijaya", "status": "hadir", "sudah_tap_gerbang": false }
  ]
}
```
- **Daftar siswa**: ambil semua siswa di `kelas_id` sesi ini. Default `status: hadir` untuk semua. Kalau ada baris `class_attendance_marks` untuk kombinasi `(session_id, student_id)` itu → pakai status dari situ (`tidak_ada_di_kelas`)
- **`sudah_tap_gerbang`**: informatif saja (cek `attendance_records` siswa itu untuk tanggal ini) — **tidak** menentukan status default (default tetap selalu `hadir` sesuai spec, meski `sudah_tap_gerbang: false` — siswa yang lupa tap gerbang tapi fisik hadir di kelas tidak boleh otomatis ke-mark salah)

### API: `PATCH /teaching-sessions/:sessionId/journal`
- Body (semua field opsional, partial update):
```json
{
  "materi": "Pengenalan React Hooks",
  "tujuan_pembelajaran": "Siswa memahami useState dan useEffect",
  "tugas_penilaian": "Latihan membuat counter component",
  "catatan": "Proyektor bermasalah, pakai laptop masing-masing"
}
```
- Upsert ke `journal_entries` (buat kalau belum ada baris untuk `session_id` ini, update kalau sudah ada)
- Guru hanya bisa edit jurnal miliknya sendiri (cek kepemilikan sesi sama seperti endpoint detail)

### API: `PATCH /teaching-sessions/:sessionId/attendance`
- Body:
```json
{
  "marks": [
    { "student_id": 1002, "status": "tidak_ada_di_kelas" }
  ]
}
```
- Untuk tiap item: upsert `class_attendance_marks` dengan `session_id`, `student_id`, `status`, `marked_by = req.user.id`, `marked_at = now()`
- **Kalau status yang dikirim adalah `hadir`** dan sebelumnya ada baris `tidak_ada_di_kelas` → hapus baris itu (kembali ke default implicit "hadir", jangan simpan baris eksplisit untuk status default — sesuai catatan desain T038)
- Validasi: `student_id` harus benar-benar siswa di `kelas_id` sesi ini — reject kalau bukan (403 atau 400, bukan diam-diam diabaikan)

## JANGAN
- ❌ JANGAN buat endpoint ini bisa diakses untuk sesi yang belum di-`start` (`started_at IS NULL`) — kalau guru belum klik "Mulai Mengajar", tidak masuk akal ada jurnal/presensi. Validasi ini di `PATCH /journal` dan `PATCH /attendance`: kalau `started_at IS NULL` → 409 `"Sesi belum dimulai"`
- ❌ JANGAN insert baris `class_attendance_marks` untuk status `hadir` — hanya untuk `tidak_ada_di_kelas` (lihat T038, hindari bloat data)
- ❌ JANGAN gunakan `sudah_tap_gerbang` sebagai default status kehadiran kelas — default HARUS selalu `hadir` terlepas dari status tap gerbang, sesuai keputusan eksplisit di interview (siswa tap gerbang ATAU tidak, keduanya dianggap hadir di kelas sampai guru koreksi manual)
- ❌ JANGAN buat batasan waktu edit jurnal di task ini — itu masih open question terpisah di spec, jangan asumsikan sendiri (misal auto-lock setelah closed) tanpa instruksi eksplisit

## Files
- **Buat:** `apps/api/src/journal/journal.module.ts`
- **Buat:** `apps/api/src/journal/journal.service.ts`
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` — tambah 3 endpoint di atas (atau buat controller terpisah `journal.controller.ts` kalau lebih rapi, keduanya bisa diterima — putuskan saat implementasi berdasarkan konsistensi struktur folder existing)

## Acceptance Criteria
- [ ] `GET detail` untuk sesi yang belum pernah ada jurnal → `jurnal` semua field `null`, bukan error
- [ ] `GET detail` daftar siswa semua default `hadir` meski belum ada tap gerbang
- [ ] `PATCH journal` dengan hanya 1 field terisi → field lain di DB tidak berubah dari yang sudah ada (partial update, bukan overwrite total)
- [ ] `PATCH attendance` tandai 1 siswa `tidak_ada_di_kelas` → `GET detail` berikutnya tampilkan status itu untuk siswa tsb
- [ ] `PATCH attendance` balikkan siswa yang tadinya `tidak_ada_di_kelas` jadi `hadir` → baris `class_attendance_marks` untuk siswa itu terhapus
- [ ] `PATCH journal`/`PATCH attendance` untuk sesi yang `started_at IS NULL` → 409
- [ ] Guru A akses sesi milik guru B → 403 di ketiga endpoint
- [ ] `PATCH attendance` dengan `student_id` yang bukan dari kelas sesi ini → ditolak, tidak masuk DB

## Handoff ke T046
T046 (UI halaman sesi guru) akan konsumsi ketiga endpoint ini sebagai satu halaman terpadu (detail sesi + form jurnal + tabel presensi).
