# T043 — API: Mulai Mengajar (Start Session) dengan Validasi Geofence

## Depends on
T041 (endpoint jadwal hari ini), T042 (config toleransi)

## Objective
Buat endpoint yang dipanggil saat guru klik "Mulai Mengajar" — memvalidasi tap gerbang, jendela waktu, dan lokasi GPS (geofence kampus), lalu membuka sesi (`started_at`, hitung `terlambat_menit`, simpan lokasi).

## Context
- **App:** `apps/api`
- **Tables:** `teaching_sessions`, `kampus`, `attendance_records`
- **Role:** `guru`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Gating & Validasi Mulai Mengajar", terutama poin 3 "Geofencing Lokasi"

## Spec Detail

### API: `POST /teaching-sessions/:sessionId/start`
- Auth: JwtAuthGuard, role `guru`
- `teacher_id` dari JWT — endpoint HARUS verifikasi `teaching_sessions.teacher_id === req.user.teacher_id`, kalau tidak cocok return 403 (bukan 404 — supaya jelas ini masalah otorisasi bukan sesi tidak ada)

**Request body:**
```json
{
  "lat": -6.123456,
  "lng": 106.123456
}
```

**Urutan validasi (WAJIB berurutan persis seperti ini, stop di validasi pertama yang gagal, return error spesifik tiap tahap):**

1. **Sesi milik guru ini?** → kalau tidak, 403 `"Sesi ini bukan milik Anda"`
2. **Sesi sudah `started_at` terisi?** → kalau sudah, 409 `"Sesi sudah dimulai sebelumnya"` (idempotency guard — cegah double-start dari double-click atau retry network)
3. **Ada `teacher_permits` aktif untuk sesi/tanggal ini?** → kalau ada, 409 `"Anda sudah diizinkan tidak mengajar untuk sesi ini"`
4. **Guru sudah tap gerbang hari ini?** (cek `attendance_records`) → kalau belum, 403 `"Anda belum tap kartu di gerbang hari ini"`
5. **Jam sekarang dalam jendela jadwal?** (`jam_mulai <= sekarang <= jam_selesai` dari `schedules` terkait) → kalau belum masuk jendela, 403 `"Belum waktunya sesi ini dimulai"`; kalau sudah lewat `jam_selesai`, 403 `"Sesi ini sudah lewat waktunya"`
6. **Validasi geofence:**
   - Ambil `kampus` dari `kelas` sesi ini (`kelas.kampus_id`)
   - Kalau `kampus.radius_geofence_meter IS NULL` atau `lokasi_lat`/`lokasi_lng` kampus itu `NULL` → **skip validasi geofence** (kampus belum dikonfigurasi, jangan hard-block karena data admin belum lengkap — log warning di server, tapi izinkan lanjut)
   - Kalau kampus punya koordinat lengkap: hitung jarak Haversine antara `(lat, lng)` request dan `(kampus.lokasi_lat, kampus.lokasi_lng)`
   - Kalau jarak `>` `radius_geofence_meter` → 403 `"Lokasi Anda di luar radius sekolah"` — **HARD BLOCK, tidak ada override** (sesuai keputusan spec)
7. Semua validasi lolos → lanjut ke eksekusi

**Eksekusi (setelah semua validasi lolos):**
- Hitung `terlambat_menit` via `ScheduleResolverService.hitungTerlambatMenit(jamMulaiJadwal, now)`
- Update `teaching_sessions`: `started_at = now()`, `lokasi_lat`, `lokasi_lng`, `terlambat_menit`
- Response 200:
```json
{
  "session_id": 501,
  "started_at": "2026-07-21T07:08:00Z",
  "terlambat_menit": 0
}
```

### Perhitungan jarak Haversine
Implementasi standar (radius bumi 6371 km), fungsi murni terpisah (`libs/geo.ts` atau sejenis) supaya bisa di-unit-test tanpa mock DB:
```typescript
function hitungJarakMeter(lat1, lng1, lat2, lng2): number
```

## JANGAN
- ❌ JANGAN percaya `lat`/`lng` dari client tanpa validasi tipe data (harus number valid, range lat -90..90, lng -180..180) — reject dengan 400 kalau format aneh, JANGAN biarkan lolos ke perhitungan jarak dengan nilai sampah
- ❌ JANGAN buat jalur override manual/bypass geofence di endpoint ini — sesuai spec, hard block TANPA override adalah keputusan sadar. Kalau nanti dibutuhkan override, itu task terpisah yang harus didiskusikan ulang dengan user, JANGAN diam-diam ditambahkan sekarang
- ❌ JANGAN skip validasi geofence secara diam-diam kalau kampus PUNYA koordinat lengkap — skip HANYA kalau data kampus memang belum lengkap (null), bukan kalau lat/lng dikirim tapi di luar radius
- ❌ JANGAN ubah urutan validasi di atas — urutan ini sengaja: cek kepemilikan dulu (keamanan), baru idempotency, baru bisnis rule, baru geofence (paling mahal secara komputasi & paling sensitif UX, ditaruh terakhir supaya guru tidak dapat pesan error lokasi kalau sebenarnya masalahnya di tempat lain)

## Files
- **Buat:** `apps/api/src/common/geo.ts` — fungsi Haversine murni
- **Buat:** `apps/api/src/common/geo.spec.ts` — unit test Haversine dengan koordinat yang jaraknya sudah diketahui manual
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.service.ts` — tambah method `startSession(sessionId, teacherId, lat, lng)`
- **Modifikasi:** `apps/api/src/teaching-sessions/teaching-sessions.controller.ts` — tambah `POST /teaching-sessions/:sessionId/start`

## Acceptance Criteria
- [ ] Unit test Haversine: jarak 2 titik yang diketahui manual (misal 2 titik berjarak ~111km karena beda 1 derajat lintang) hasilnya sesuai toleransi kecil
- [ ] Start session tanpa tap gerbang dulu → 403 dengan pesan jelas
- [ ] Start session di luar jendela waktu (terlalu pagi) → 403
- [ ] Start session dengan lokasi di luar radius kampus (kampus yang SUDAH ada koordinat+radius) → 403, `teaching_sessions.started_at` TETAP null (tidak ke-update)
- [ ] Start session dengan lokasi dalam radius → 200, `started_at` terisi, `terlambat_menit` sesuai perhitungan
- [ ] Start session 2x untuk sesi yang sama → request kedua 409
- [ ] Start session untuk kampus yang `radius_geofence_meter IS NULL` → lolos validasi geofence (skip), sesi tetap bisa dimulai
- [ ] Start session untuk sesi milik guru lain → 403

## Handoff ke T044 & T045
T044 (auto-close job) akan mengandalkan `started_at` yang diisi task ini untuk menentukan sesi mana yang perlu di-closed. T045 (UI dashboard guru) memanggil endpoint ini dan harus menangani SEMUA kode error di atas dengan pesan yang jelas ke guru (termasuk minta izin lokasi browser sebelum submit).
