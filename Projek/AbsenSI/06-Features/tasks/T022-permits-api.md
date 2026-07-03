# T022 — Permits Module: API (Fase 1b)

## Depends on
T015 (activity_log harus sudah ada), T012 (attendance_records harus ada)

## Objective
Bangun API lengkap untuk modul Permits — pencatatan izin tidak masuk dan izin keluar oleh guru piket, termasuk generate kode verifikasi dan update otomatis ke attendance_records.

## Context
- **App:** `apps/api`
- **Tables:** `permits`, `attendance_records`
- **Role akses:** `guru_piket` SAJA (bukan super_admin — ADR-019)
- **ADR:** ADR-016 (permits table), ADR-019 (kewenangan eksklusif piket)
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-piket.md` — baca SELURUH file

## Spec Detail

### Install:
```
pnpm add nanoid --filter api  // untuk kode_verifikasi
```

### Endpoint 1: `POST /permits` — buat permit baru

**Body untuk `jenis: tidak_masuk`:**
```json
{
  "student_id": "xxx",
  "jenis": "tidak_masuk",
  "alasan_kategori": "sakit",  // atau "izin"
  "alasan_detail": "Demam tinggi",
  "tanggal": "2026-07-03"
}
```

**Body untuk `jenis: keluar`:**
```json
{
  "student_id": "xxx",
  "jenis": "keluar",
  "alasan_kategori": "izin",
  "alasan_detail": "Keperluan keluarga",
  "tanggal": "2026-07-03",
  "jam_keluar": "10:00",
  "jam_kembali_diharapkan": "13:00"  // optional
}
```

**Logic setelah save permits:**

Untuk `tidak_masuk`:
- Cari atau buat `attendance_records` dengan `tanggal` + `student_id`
- Update/set `status = alasan_kategori` (izin atau sakit)
- Pastikan tidak overwrite `waktu_masuk` kalau sudah ada (siswa ternyata hadir duluan)

Untuk `keluar`:
- Cari `attendance_records` hari itu — biarkan seperti adanya (status hadir tetap hadir)
- Set `permits.status_kembali = 'belum'`
- Generate `kode_verifikasi` = nanoid(8).toUpperCase() (misal "A3B7X2QP")
- Catat `approved_by = req.user.id`

**Response:** permits yang baru dibuat + `kode_verifikasi` (untuk dipakai construct URL print)

### Endpoint 2: `PATCH /permits/:id/confirm-kembali`
- Set `status_kembali = 'sudah'`, isi `kembali_dikonfirmasi_at`, `kembali_dikonfirmasi_by`
- Tidak mengubah `attendance_records`

### Endpoint 3: `PATCH /permits/:id/set-pulang`
- Set `status_kembali = 'pulang'`
- Update `attendance_records`: set `waktu_pulang = permits.jam_keluar`, `pulang_via = 'piket_izin'`

### Endpoint 4: `POST /attendance/manual-pulang`
```json
{
  "student_id": "xxx",
  "tanggal": "2026-07-03",
  "waktu_pulang": "14:30",
  "catatan": "Siswa lupa tap pulang, konfirmasi langsung ke piket"
}
```
- Update `attendance_records`: set `waktu_pulang`, `pulang_via = 'piket_izin'`
- Log ke `activity_log` dengan action `attendance.manual_pulang`

### Endpoint 5: `POST /attendance/confirm-izin-pulang/:record_id`
- Update `attendance_records.pulang_via` dari `tap` → `tap_izin_pulang`
- Log ke `activity_log` dengan action `attendance.confirm_izin_pulang`

### Validasi scope kampus:
Semua endpoint ini WAJIB memvalidasi bahwa `student.kelas.kampus_id == req.user.kampus_id`:
```typescript
// Piket kampus 1 tidak boleh input permit untuk siswa kampus 2
if (student.kelas.kampus_id !== req.user.kampus_id) {
  throw new ForbiddenException('Siswa bukan dari kampus Anda');
}
```

## JANGAN
- ❌ JANGAN izinkan `super_admin` akses endpoint ini — role guard harus strict `guru_piket` only (ADR-019)
- ❌ JANGAN buat field `akan_kembali` — sudah dihapus dari desain. Status kembali ditentukan dari `status_kembali` enum
- ❌ JANGAN skip scope validasi kampus — piket kampus 1 tidak boleh bisa buat permit untuk siswa kampus 2
- ❌ JANGAN overwrite `waktu_masuk` yang sudah ada saat buat permit `tidak_masuk`
- ❌ JANGAN generate `kode_verifikasi` untuk permit `tidak_masuk` — hanya untuk `keluar` yang dicetak

## Files
- **Buat:** `apps/api/src/permits/permits.module.ts`
- **Buat:** `apps/api/src/permits/permits.service.ts`
- **Buat:** `apps/api/src/permits/permits.controller.ts`
- **Modifikasi:** `apps/api/src/attendance/attendance.controller.ts` — tambah endpoint manual-pulang dan confirm-izin-pulang

## Acceptance Criteria
- [ ] Login sebagai `super_admin` → `POST /permits` → 403 Forbidden
- [ ] Login sebagai `guru_piket` kampus 1 → buat permit untuk siswa kampus 2 → 403 Forbidden
- [ ] `POST /permits(tidak_masuk, sakit)` → `attendance_records` hari itu status jadi `sakit`
- [ ] `POST /permits(keluar)` → `kode_verifikasi` terisi di response
- [ ] `PATCH /permits/:id/set-pulang` → `attendance_records.pulang_via` jadi `piket_izin`
- [ ] `POST /attendance/manual-pulang` → `activity_log` berisi action `attendance.manual_pulang`
- [ ] Scope kampus divalidasi di semua endpoint
