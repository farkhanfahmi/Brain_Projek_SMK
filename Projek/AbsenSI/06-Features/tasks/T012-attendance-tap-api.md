# T012 — Attendance API: Core Tap Logic

## Depends on
T007 (cards harus bisa dicari by UID), T010 (schedules/threshold harus ada), T011 (kiosk UI sudah ada untuk test)

## Objective
Implementasi endpoint `/attendance/tap` — logika inti pemrosesan tap RFID termasuk debounce, tap-1/tap-2+, perhitungan terlambat, dan dispatch event ke BullMQ.

## Context
- **App:** `apps/api`
- **Tables:** `cards`, `students`, `teachers`, `attendance_records`, `schedules`
- **Guard:** `KioskGuard`
- **ADR:** ADR-005 (terlambat dihitung dari tap gerbang), ADR-006 (event ke BullMQ)
- **Ref:** `Projek/AbsenSI/06-Features/absensi-gerbang.md` — baca SELURUH file

## Spec Detail

### Install BullMQ:
```
pnpm add bullmq @nestjs/bullmq --filter api
```

### `POST /attendance/tap` (dilindungi `KioskGuard`):

**Request body:**
```json
{
  "uid": "A1B2C3D4",
  "client_uuid": "nanoid-unik-dari-kiosk",
  "kiosk_id": "gerbang-1"
}
```

**Alur pemrosesan (urutan wajib diikuti):**

```
1. Cek client_uuid di attendance_records
   → kalau sudah ada: return 200 OK dengan data record yang ada (idempotent, tidak proses ulang)

2. Cari card by uid
   → tidak ditemukan: return result "rejected_unknown"

3. Cek card.status
   → 'inactive': return result "rejected_inactive"

4. Tentukan person (student atau teacher dari card)

5. Kalau student: cek students.locked_at
   → tidak null: return result "rejected_locked"

6. CEK DEBOUNCE:
   Query tap_events: kartu yang sama (card_id), scanned_at > now() - 30 detik
   → kalau ada: return result "rejected_duplicate"
   (Catat: ini cek dari tap_events, bukan attendance_records)

7. Tentukan ini tap-1 atau tap-2+:
   Query attendance_records: (student_id atau teacher_id), tanggal = hari ini
   
   JIKA tidak ada record hari ini (tap-1):
   - Hitung status: bandingkan server timestamp dengan jam masuk dari schedules
     * Siswa: jam masuk sekolah hari itu (type: jam_sekolah, hari: hari ini)
     * Guru: jam mengajar pertama hari itu - threshold_terlambat_menit
   - Buat attendance_record baru:
     waktu_masuk = scanned_at, status = 'hadir'/'terlambat'
   
   JIKA sudah ada record hari ini (tap-2+):
   - UPDATE record yang ada: waktu_pulang = scanned_at, pulang_via = 'tap'
   
8. Simpan ke tap_events (result: 'accepted', attendance_record_id = record yang dibuat/diupdate)

9. Dispatch event ke BullMQ:
   queue: 'attendance'
   job: 'attendance.recorded'
   payload: { personId, personType, tapType: 'masuk'/'pulang', timestamp: scanned_at, kioskId }

10. Return response ke kiosk:
```json
{
  "result": "accepted",
  "tap_type": "masuk",
  "person": { "nama": "Budi Santoso", "identifier": "XI-RPL-1" },
  "waktu": "07:32:45",
  "status": "terlambat"
}
```

### Aturan penting:
- `scanned_at` = **timestamp server saat request diterima**, bukan dari request body
- Semua DB write dalam 1 transaksi Prisma (attendance_record + tap_event)
- BullMQ dispatch di luar transaksi (fire-and-forget, jangan block response)

### BullMQ setup:
- Queue `attendance` — konsumer hanya stub logging untuk sekarang (Fase 3 akan pasang WA notif)
- Setup `AttendanceProcessor` yang hanya `console.log` event di Fase 1

## JANGAN
- ❌ JANGAN gunakan timestamp dari request body kiosk sebagai `scanned_at` — selalu server timestamp
- ❌ JANGAN buat tap-3 diabaikan — tap-3+ tetap UPDATE waktu_pulang ke tap terakhir
- ❌ JANGAN lupa cek debounce SEBELUM cek tap-1/tap-2 — urutan penting
- ❌ JANGAN skip idempotency check (`client_uuid`) — ini krusial untuk offline buffer
- ❌ JANGAN buat tap_events di task terpisah — T013 handle logging, tapi `tap_events` insert HARUS terjadi di dalam transaksi T012 juga (termasuk yang rejected)
- ❌ JANGAN throw exception untuk rejected tap — return 200 OK dengan `result` enum yang sesuai (kiosk tidak perlu catch error)

## Files
- **Buat:** `apps/api/src/attendance/attendance.module.ts`
- **Buat:** `apps/api/src/attendance/attendance.service.ts`
- **Buat:** `apps/api/src/attendance/attendance.controller.ts`
- **Buat:** `apps/api/src/attendance/processors/attendance.processor.ts` (BullMQ consumer stub)
- **Modifikasi:** `apps/api/src/app.module.ts` — import AttendanceModule, BullMQ config

## Acceptance Criteria
- [ ] Tap UID valid + kartu aktif → `attendance_record` baru dibuat, response `tap_type: masuk`
- [ ] Tap UID yang sama lagi (tap-2) → `waktu_pulang` diupdate, response `tap_type: pulang`
- [ ] Tap UID yang sama < 30 detik setelah tap sebelumnya → `rejected_duplicate`, tidak buat record baru
- [ ] Tap UID kartu nonaktif → `rejected_inactive`
- [ ] Tap UID tidak dikenal → `rejected_unknown`
- [ ] Tap siswa yang locked → `rejected_locked`
- [ ] Kirim request dengan `client_uuid` yang sama 2 kali → response sama, tidak ada record duplikat
- [ ] `scanned_at` di database berbeda dengan timestamp di request body (membuktikan server timestamp dipakai)
- [ ] BullMQ job `attendance.recorded` muncul di queue setelah tap sukses

## Handoff ke T013
T013 akan memverifikasi bahwa `tap_events` sudah selalu terisi (termasuk untuk rejected tap) dan tidak ada endpoint DELETE/UPDATE untuk tabel ini.
