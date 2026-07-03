# T026 — Dashboard Piket: Lock/Unlock Siswa

## Depends on
T023 (dashboard piket layout), T012 (tap logic harus cek lock sebelum proses)

## Objective
Implementasi fitur lock/unlock siswa oleh guru piket, termasuk integrasi ke tap processing agar siswa terkunci ditolak di gerbang dengan pesan yang tepat.

## Context
- **App:** `apps/api` + `apps/web`
- **Tables:** `students` (kolom lock), `tap_events`
- **Role akses:** `guru_piket`
- **ADR:** ADR-017 (lock manual, tidak auto-lock)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-piket.md` — Fungsi 6

## Spec Detail

### API Endpoints (akses: `guru_piket`):

**POST `/students/:id/lock`**
```json
{ "locked_reason": "Izin keluar 2 hari lalu, belum ada konfirmasi" }
```
- Set: `locked_at = now()`, `locked_reason`, `locked_by = req.user.id`
- Bersihkan: `unlocked_at = null`, `unlocked_by = null`, `unlock_note = null` (reset dari unlock sebelumnya kalau ada)
- Validasi scope kampus: student.kelas.kampus_id == req.user.kampus_id
- Log ke `activity_log`: action `student.lock`

**POST `/students/:id/unlock`**
```json
{ "unlock_note": "Orang tua sudah konfirmasi via telepon, kasus diselesaikan BK" }
```
- Set: `unlocked_at = now()`, `unlocked_by = req.user.id`, `unlock_note`
- Bersihkan: `locked_at = null`, `locked_reason = null`, `locked_by = null`
- Log ke `activity_log`: action `student.unlock`

### Verifikasi T012 (tap logic) — sudah handle lock:
Pastikan di `AttendanceService.processTap()` sudah ada:
```typescript
if (student.locked_at !== null) {
  // Catat di tap_events dengan result: rejected_locked
  return { result: 'rejected_locked' };
}
```
Dan kiosk UI (T011) sudah menampilkan "Hubungi Guru Piket" untuk response ini.

### UI di Dashboard Piket (`/piket`):

**Section: "Perlu Ditinjau — Kandidat Lock"**
- Data: siswa yang punya `permits(keluar, status_kembali=belum)` dari hari-hari sebelumnya (bukan hari ini)
- Ini adalah siswa yang "berpotensi" perlu dikunci — piket review sendiri
- Tombol **[Kunci Siswa]** → modal form: textarea alasan (required) → submit

**Section: "Siswa Terkunci"**
- Data: `students` dengan `locked_at IS NOT NULL` dan `kelas.kampus_id = req.user.kampus_id`
- Per baris: nama, kelas, terkunci sejak, alasan
- Tombol **[Buka Kunci]** → modal form: textarea catatan (required) → submit

Kedua section ini bisa ditempatkan di tab terpisah di halaman `/piket` atau di sub-halaman `/piket/lock` — pilih yang lebih clean untuk layout yang sudah ada dari T023 dan T025.

### Feedback di kiosk (verifikasi T011):
Pastikan halaman kiosk sudah menampilkan:
```
❌ SISWA TERKUNCI
   Hubungi Guru Piket
```
Ini berbeda dari pesan "Kartu tidak aktif" — harus jelas berbeda supaya siswa tahu harus ke mana.

## JANGAN
- ❌ JANGAN buat auto-lock berdasarkan kondisi apapun — lock HANYA dari aksi manual piket (ADR-017)
- ❌ JANGAN buat admin bisa lock/unlock — ini eksklusif piket (ADR-019)
- ❌ JANGAN skip validasi scope kampus — piket kampus 1 tidak boleh lock siswa kampus 2
- ❌ JANGAN buat proses BK atau follow-up disciplinary di dalam sistem — itu offline, di luar scope aplikasi
- ❌ JANGAN tampilkan tombol lock di tabel utama (T023) — lock ada di section terpisah "Perlu Ditinjau"

## Files
- **Modifikasi:** `apps/api/src/core/students/students.controller.ts` — tambah endpoint lock/unlock
- **Modifikasi:** `apps/api/src/core/students/students.service.ts` — method lock/unlock
- **Modifikasi:** `apps/web/app/piket/page.tsx` — tambah section Lock/Unlock
- **Buat:** `apps/web/app/piket/components/LockSection.tsx`
- **Buat:** `apps/web/app/piket/components/TerkunciSection.tsx`

## Acceptance Criteria
- [ ] `POST /students/:id/lock` oleh piket kampus yang benar → `locked_at` terisi
- [ ] `POST /students/:id/lock` oleh piket kampus lain → 403
- [ ] `POST /students/:id/lock` oleh `super_admin` → 403
- [ ] Siswa yang terkunci tap di kiosk → response `rejected_locked`, kiosk tampilkan "Hubungi Guru Piket"
- [ ] `POST /students/:id/unlock` → `locked_at` jadi null, `unlock_note` terisi
- [ ] Setelah unlock, siswa tap di kiosk → berhasil (tidak lagi rejected_locked)
- [ ] `activity_log` mencatat `student.lock` dan `student.unlock` dengan snapshot before/after
- [ ] Section "Siswa Terkunci" di dashboard piket menampilkan siswa yang dikunci

---

## ✅ Setelah T026 selesai: Fase 1b COMPLETE

Semua 26 task Fase 1 sudah selesai. Langkah selanjutnya:
1. End-to-end testing dengan hardware fisik (reader RFID, printer thermal)
2. Review semua acceptance criteria dari T001–T026
3. Deployment ke server sekolah (lihat `Projek/AbsenSI/10-Environment.md`)
4. Edit `C:\ProjekSMK\print.php` kalau belum dilakukan di T024
5. Buka `Projek/AbsenSI/13-Backlog.md` untuk mulai planning Fase 2
