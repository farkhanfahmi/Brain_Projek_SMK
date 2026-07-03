# T025 — Dashboard Piket: Monitoring & Antrian Klarifikasi

## Depends on
T022 (permits API), T016 (end-of-day job), T023 (layout piket)

## Objective
Buat section monitoring di Dashboard Piket: daftar siswa yang belum kembali dari izin keluar, dan antrian siswa yang kemarin tidak tap pulang dan perlu klarifikasi.

## Context
- **App:** `apps/web` (UI) + `apps/api` (endpoint)
- **Route:** tambahkan section di `/piket` (bukan halaman terpisah)
- **Role:** `guru_piket`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-piket.md` — Fungsi 4 & Fungsi 5 Antrian Klarifikasi

## Spec Detail

### Section 1: "Belum Kembali dari Izin" (di bawah tabel utama `/piket`)

**Data source:** `GET /permits?status_kembali=belum&jenis=keluar&kampus_id=X&tanggal=today`

Tampilkan kalau `jam_kembali_diharapkan < now()` (jam sudah lewat):
```
┌─ BELUM KEMBALI ──────────────────────────────────────┐
│ Budi Santoso · XI-RPL-1                               │
│ Keluar: 10:00 · Dijanjikan kembali: 13:00 (sudah lewat 1j 30m) │
│ Alasan: Keperluan keluarga                            │
│                         [Sudah Kembali] [Dianggap Pulang] │
└───────────────────────────────────────────────────────┘
```

Tombol aksi:
- **[Sudah Kembali]** → `PATCH /permits/:id/confirm-kembali` → card hilang dari daftar
- **[Dianggap Pulang]** → konfirmasi dialog → `PATCH /permits/:id/set-pulang` → card hilang dari daftar

Kalau `jam_kembali_diharapkan` null (tidak diset saat izin): kartu tetap muncul tapi tanpa countdown, hanya tombol aksi.

### Section 2: "Perlu Klarifikasi — Tidak Tap Pulang Kemarin"

**Data source:** `GET /attendance/missing-pulang?tanggal=kemarin&kampus_id=X`

```
┌─ TIDAK TAP PULANG KEMARIN ──────────────────────────┐
│ Citra Dewi · X-MM-1                                  │
│ Masuk: 07:20 kemarin, tidak ada tap pulang           │
│                                                       │
│ [Konfirmasi Pulang Normal]  [Catat Sebagai Izin Keluar] │
└──────────────────────────────────────────────────────┘
```

Tombol aksi:
- **[Konfirmasi Pulang Normal]** → modal: input jam pulang perkiraan + catatan → `POST /attendance/manual-pulang` → card hilang dari daftar
- **[Catat Sebagai Izin Keluar]** → modal form izin keluar retroaktif (tanggal = kemarin, jenis=keluar) → `POST /permits` + update attendance → card hilang

### Auto-refresh:
Kedua section di-refresh setiap 60 detik (polling ringan — tidak perlu Socket.IO untuk ini).

### Posisi di halaman:
```
/piket page layout:
┌─ Header ──────────────────────────────────────┐
├─ Tabel Siswa Hari Ini (T023) ─────────────────┤
├─ Section: Belum Kembali dari Izin ────────────┤  ← T025
├─ Section: Tidak Tap Pulang Kemarin ───────────┤  ← T025
└───────────────────────────────────────────────┘
```

Kalau kedua section kosong (tidak ada data) → tidak ditampilkan sama sekali (jangan tampilkan section kosong yang bikin tampilan berantakan).

## JANGAN
- ❌ JANGAN buat halaman terpisah untuk monitoring — tambahkan section di `/piket` yang sudah ada
- ❌ JANGAN trigger lock otomatis dari sini — lock hanya via aksi manual di T026
- ❌ JANGAN tampilkan siswa dari kampus lain
- ❌ JANGAN gunakan Socket.IO untuk refresh section ini — polling 60 detik sudah cukup (data tidak secritical tap realtime)

## Files
- **Modifikasi:** `apps/web/app/piket/page.tsx` — tambah dua section baru
- **Buat:** `apps/web/app/piket/components/BelumKembaliSection.tsx`
- **Buat:** `apps/web/app/piket/components/KlarifikasiPulangSection.tsx`

## Acceptance Criteria
- [ ] Siswa dengan `status_kembali=belum` dan `jam_kembali_diharapkan` sudah lewat → muncul di section "Belum Kembali"
- [ ] Klik [Sudah Kembali] → card hilang dari section
- [ ] Klik [Dianggap Pulang] → `attendance_records.pulang_via` jadi `piket_izin`, card hilang
- [ ] `GET /attendance/missing-pulang?tanggal=kemarin` → muncul di section "Tidak Tap Pulang Kemarin"
- [ ] Klik [Konfirmasi Pulang Normal] → isi jam → submit → `waktu_pulang` terisi, card hilang
- [ ] Kalau tidak ada data di kedua section → section tidak ditampilkan
