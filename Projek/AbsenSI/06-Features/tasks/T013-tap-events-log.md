# T013 — Tap Events Logging (Verifikasi & Hardening)

## Depends on
T012 (tap_events sudah ditulis di dalam transaksi tap)

## Objective
Verifikasi bahwa tap_events sudah ter-log dengan benar untuk SEMUA skenario tap (termasuk yang ditolak), dan pastikan tidak ada celah untuk modifikasi log dari luar.

## Context
- **App:** `apps/api`
- **Tables:** `tap_events` (insert-only)
- **ADR:** ADR-020 (tap_events = immutable forensic log)

## Spec Detail

### Yang perlu diverifikasi dari T012:
Setiap call ke `POST /attendance/tap` harus menghasilkan TEPAT 1 baris di `tap_events`, termasuk:
- Tap accepted → `result: accepted`, `attendance_record_id` terisi
- Tap rejected (semua jenis) → `result: rejected_*`, `attendance_record_id: null`
- Tap duplicate (debounce) → `result: rejected_duplicate`

### Yang perlu ditambahkan di task ini:

**1. API read-only untuk admin:**
- `GET /tap-events` (akses: `super_admin`) — untuk keperluan audit forensik
  - Filter: `kiosk_id`, `card_uid`, `result`, `from`, `to`
  - Pagination
  - Include: nama pemilik kartu (kalau card_id ada)

**2. Verifikasi tidak ada endpoint modifikasi:**
- Scan seluruh `AttendanceController` dan `TapEventController` — pastikan tidak ada `PATCH` atau `DELETE` untuk `tap_events`
- Tambahkan komentar `// INSERT ONLY — do not add UPDATE or DELETE endpoints` di service

**3. Admin UI — halaman Audit Log Tap:**
- Route `/admin/audit/tap` (sederhana, bukan dashboard utama)
- Tabel: Waktu | UID | Pemilik | Kiosk | Hasil
- Filter: kiosk, rentang tanggal, hasil (accepted/rejected)
- Ini halaman forensik — desain sederhana, fungsional, tidak perlu fancy

### Verifikasi manual yang wajib dilakukan setelah task selesai:
Lakukan 5 skenario tap dengan hardware reader fisik dan verifikasi di database:
1. Tap kartu valid → cek `tap_events` ada 1 baris result=accepted
2. Tap kartu tidak dikenal → cek ada baris result=rejected_unknown
3. Tap 2x dalam 10 detik → cek tap kedua result=rejected_duplicate
4. Tap kartu nonaktif → cek result=rejected_inactive
5. Tap siswa terkunci → cek result=rejected_locked

## JANGAN
- ❌ JANGAN buat endpoint `DELETE /tap-events` atau `PATCH /tap-events` dalam kondisi apapun
- ❌ JANGAN buat fitur "hapus log lama" via UI — kalau perlu cleanup, itu dilakukan via ETL ke data warehouse (ADR-013), bukan delete dari aplikasi
- ❌ JANGAN skip pengujian dengan hardware fisik — debounce 30 detik tidak bisa ditest akurat dengan simulasi keyboard biasa

## Files
- **Buat:** `apps/api/src/attendance/tap-event.controller.ts` (GET endpoint saja)
- **Buat:** `apps/web/app/(admin)/audit/tap/page.tsx`
- **Modifikasi:** `apps/api/src/attendance/attendance.service.ts` — tambahkan komentar INSERT ONLY

## Acceptance Criteria
- [ ] Setelah 10 tap skenario berbeda, semua ada di `tap_events`
- [ ] `GET /tap-events?result=rejected_duplicate` return hanya tap yang kena debounce
- [ ] Tidak ada route `PATCH` atau `DELETE` untuk tap_events di seluruh codebase (`grep -r "tap-events" apps/api/src --include="*.ts"` tidak boleh return method DELETE/PATCH)
- [ ] Halaman `/admin/audit/tap` menampilkan data dengan filter berfungsi
