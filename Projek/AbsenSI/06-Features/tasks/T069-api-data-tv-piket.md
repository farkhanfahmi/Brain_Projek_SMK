# T069 — API: Data Agregat TV Piket (4 Widget)

## Depends on
T068 (guard & token TV Piket), Rekap Admin Fase 1 (reuse logic hitung alfa/hadir/izin — JANGAN hitung ulang dari nol)

## Objective
Buat endpoint yang mengembalikan data 4 widget TV Piket (bar persentase, siswa tidak hadir, guru belum mulai mengajar, guru izin) — semuanya di-scope ke `kampusId` dari token TV yang divalidasi `TvPiketGuard`.

## Context
- **App:** `apps/api`
- **Ref:** `Projek/AbsenSI/06-Features/tv-piket.md` — bagian "Isi/Widget" (final, 2026-07-22)
- **Reuse WAJIB:** service Rekap Kehadiran existing (Fase 1) untuk widget #1/#2 — cek `apps/api/src/attendance/` untuk method rekap yang sudah ada, JANGAN tulis ulang logic hitung alfa

## Spec Detail

### API: `GET /tv-piket/data` (guard: `TvPiketGuard`)
Response gabungan (1 endpoint untuk semua widget, supaya frontend cukup 1 kali fetch/subscribe):
```json
{
  "kampus": { "id": 1, "nama": "Kampus 1" },
  "persentase": {
    "hadir": { "jumlah": 820, "persen": 82 },
    "izin_sakit": { "jumlah": 90, "persen": 9 },
    "alfa": { "jumlah": 90, "persen": 9 }
  },
  "siswaTidakHadir": [
    { "studentId": 1001, "nama": "Budi Santoso", "kelas": "XI-RPL-1", "status": "sakit", "keterangan": "Demam" }
  ],
  "guruBelumMulai": [
    { "teacherId": 45, "nama": "Ahmad S.", "kelas": "XI-TKJ-2", "mapel": "Produktif TKJ", "terlambatMenit": 12 }
  ],
  "guruIzin": [
    { "teacherId": 50, "nama": "Siti R.", "kategori": "sakit", "cakupan": "Seharian Penuh", "tugasSudahDiisi": false, "followUpNeeded": true }
  ]
}
```

### Logic tiap bagian
- **`persentase`** — panggil service Rekap existing dengan filter kampus (bukan filter kelas/jurusan), hari ini saja. `izin_sakit` gabungan status `izin`+`sakit` dari `attendance_records`
- **`siswaTidakHadir`** — query `attendance_records` hari ini di kampus tsb, status `izin`/`sakit`/`alfa` (alfa dihitung sesuai logic hari-wajib existing, BUKAN kolom DB — konsisten prinsip lama), join `permits` untuk `keterangan` (alasan_detail)
- **`guruBelumMulai`** — query `teaching_sessions` hari ini, `kelasId` yang `kelas.kampusId` cocok, filter: jam mulai jadwal (+ toleransi) sudah lewat DAN `startedAt IS NULL` DAN `closedAt IS NULL` (masih relevan, belum lewat total). `terlambatMenit` dihitung sama seperti `ScheduleResolverService.hitungTerlambatMenit` (reuse, JANGAN duplikat logic)
- **`guruIzin`** — query `teacher_permits` hari ini untuk guru yang jadwalnya di kampus tsb, `cakupan` = nama kelas+mapel kalau `sessionId` terisi, atau `"Seharian Penuh"` kalau null

## JANGAN
- ❌ JANGAN hitung ulang logic alfa/hadir dari nol — panggil service Rekap Fase 1 yang sudah ada dan teruji
- ❌ JANGAN hitung ulang logic `terlambatMenit` secara terpisah — reuse `ScheduleResolverService.hitungTerlambatMenit` yang sudah dibuat T042
- ❌ JANGAN buat endpoint ini bisa diakses tanpa `TvPiketGuard` — ini data siswa/guru yang cukup sensitif (nama + alasan sakit/izin), harus terproteksi meski TV dianggap "read-only publik" secara fisik
- ❌ JANGAN kembalikan data kampus LAIN selain yang terikat ke token — validasi `kampusId` dari guard, jangan terima `kampusId` dari query param request

## Files
- **Buat:** `apps/api/src/tv-piket/tv-piket.module.ts`, `.service.ts`, `.controller.ts`

## Acceptance Criteria
- [ ] `GET /tv-piket/data` dengan token kampus 1 → hanya data kampus 1, tidak bocor data kampus lain
- [ ] Angka `persentase` cocok dengan hasil Rekap Admin untuk kampus & tanggal yang sama (verifikasi 1 sumber kebenaran)
- [ ] `guruBelumMulai` HANYA muncul untuk guru yang jam mulainya sudah lewat toleransi DAN belum `started_at` — guru yang jadwalnya belum waktunya TIDAK muncul di sini
- [ ] `guruIzin` dengan `followUpNeeded: true` konsisten dengan flag yang sama di dashboard admin_jurnal (T051) — 1 sumber data, bukan 2 perhitungan terpisah

## Handoff ke T070
T070 (halaman TV Piket) konsumsi endpoint ini + subscribe Socket.IO channel `attendance:kampus:{id}` untuk update realtime.
