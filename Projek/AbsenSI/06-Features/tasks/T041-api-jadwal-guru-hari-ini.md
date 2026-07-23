# T041 — API: Jadwal Guru Hari Ini (untuk Dashboard Guru)

## Depends on
T040 (teaching_sessions harus sudah bisa digenerate)

## Objective
Buat endpoint yang mengembalikan daftar sesi mengajar guru hari ini, lengkap dengan status masing-masing slot (belum bisa mulai / bisa mulai / sedang berlangsung / sudah closed / izin) — jadi satu-satunya sumber data untuk halaman utama dashboard guru (T045).

## Context
- **App:** `apps/api`
- **Tables:** `teaching_sessions`, `schedules`, `teacher_permits`, `attendance_records` (cek tap gerbang)
- **Role:** `guru`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Konsep Inti" & "Gating & Validasi"

## Spec Detail

### API: `GET /teaching-sessions/my-today`
- Auth: JwtAuthGuard, role `guru`
- `teacher_id` dari JWT payload (`req.user.teacher_id`) — **JANGAN** dari query param

**Response:**
```json
{
  "sudah_tap_gerbang": true,
  "sesi": [
    {
      "session_id": 501,
      "schedule_id": 88,
      "kelas": "XI-RPL-1",
      "mapel": "Pemrograman Web",
      "jam_mulai": "07:00",
      "jam_selesai": "08:30",
      "status": "belum_mulai",
      "bisa_mulai": true,
      "sudah_diizinkan": false,
      "started_at": null,
      "closed_at": null
    },
    {
      "session_id": 502,
      "schedule_id": 91,
      "kelas": "XI-RPL-2",
      "mapel": "Matematika",
      "jam_mulai": "08:30",
      "jam_selesai": "10:00",
      "status": "sudah_diizinkan",
      "bisa_mulai": false,
      "sudah_diizinkan": true,
      "permit_id": 12,
      "tugas_sudah_diisi": false,
      "started_at": null,
      "closed_at": null
    }
  ]
}
```

### Logic tiap field `status` (computed, bukan kolom DB):
- `sudah_diizinkan` — kalau ada `teacher_permits` untuk `teacher_id` + tanggal ini yang cocok (`session_id` sama ATAU `session_id IS NULL` yang berarti izin seharian)
- `belum_mulai` — `teaching_sessions.started_at IS NULL` dan tidak diizinkan dan jam sekarang belum masuk jendela mulai
- `bisa_dimulai` — `started_at IS NULL`, tidak diizinkan, DAN jam sekarang sudah masuk jendela (>= `jam_mulai`, belum lewat `jam_selesai`)
- `sedang_berlangsung` — `started_at IS NOT NULL` dan `closed_at IS NULL`
- `selesai` — `closed_at IS NOT NULL`

### Logic `bisa_mulai` (boolean terpisah, dipakai FE untuk enable/disable tombol):
`bisa_mulai = true` HANYA kalau SEMUA syarat berikut terpenuhi:
1. `sudah_tap_gerbang = true` (lihat di bawah)
2. Jam sekarang antara `jam_mulai` s/d `jam_selesai` slot itu
3. `teaching_sessions.started_at IS NULL` (belum pernah mulai)
4. Tidak ada `teacher_permits` aktif untuk sesi/tanggal ini

> **Catatan:** endpoint ini TIDAK mengecek geofence (lokasi baru dikirim saat POST start-session, lihat T043). `bisa_mulai` di sini murni syarat #1-4, validasi lokasi terjadi di request terpisah saat guru benar-benar klik.

### `sudah_tap_gerbang`
- Cek `attendance_records` where `teacher_id = req.user.teacher_id AND tanggal = hari ini` — ada record berarti sudah tap
- Field level-atas (bukan per-sesi) karena syaratnya sama untuk semua sesi guru itu hari ini

## JANGAN
- ❌ JANGAN hitung ulang jadwal dari `schedules` langsung di endpoint ini — HARUS baca dari `teaching_sessions` yang sudah digenerate job T040 (kalau job belum jalan/gagal, endpoint ini return array kosong, itu bukan bug yang harus di-workaround di sini — perbaikannya di admin trigger manual T040)
- ❌ JANGAN expose endpoint ini tanpa filter `teacher_id` dari JWT — celah kalau guru A bisa lihat jadwal guru B
- ❌ JANGAN sertakan data siswa/jurnal lengkap di endpoint ini — itu scope T042 (detail 1 sesi), endpoint ini hanya ringkasan daftar untuk halaman utama dashboard

## Files
- **Buat:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` — tambah method `getMyToday(teacherId)`
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` — tambah `GET /teaching-sessions/my-today`

## Acceptance Criteria
- [ ] Guru belum tap gerbang → semua sesi `bisa_mulai: false`, `sudah_tap_gerbang: false` di response level-atas
- [ ] Guru sudah tap gerbang, sesi jam 07:00-08:30, waktu sekarang 07:10 → `bisa_mulai: true` untuk sesi itu
- [ ] Waktu sekarang 06:50 (belum masuk jendela) → `bisa_mulai: false`, `status: belum_mulai`
- [ ] Sesi yang sudah ada `teacher_permits` → `sudah_diizinkan: true`, `bisa_mulai: false` meski syarat lain terpenuhi
- [ ] Login guru A, akses endpoint → hanya sesi guru A yang muncul, bukan guru B
