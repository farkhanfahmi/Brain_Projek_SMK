---
tags:
  - task
  - bugfix
  - backend
created: 2026-06-06
status: ready
---

# Task-009: Fix KelilingController — 3 Bug Pasca Evaluasi

## Objective
Perbaiki 2 bug kritis dan 1 bug minor di `KelilingController.php` yang ditemukan saat evaluasi task-006/007/008.

## Context
- **File:** `backend/app/Http/Controllers/KelilingController.php` — satu-satunya file yang diubah
- **Jangan sentuh:** Frontend, migration, model, controller lain
- **Jangan refactor** di luar scope bug ini

---

## Bug 1 — KRITIS: Kolom `keterangan` Tidak Ada di `presensi_pesertas`

### Root Cause
Method `simpanKeterangan()` mencoba menyimpan ke kolom `keterangan`:

```php
// SEKARANG — SALAH
$presensi = PresensiPeserta::create([
    'kode_peserta'          => $request->nomor_peserta,
    'ujian_id'              => $jadwal->ujian_id,
    'panitia_id'            => $request->panitia_id,
    'scanned_by_panitia_id' => $request->panitia_id,
    'keterangan'            => $request->keterangan,  // ← kolom TIDAK ADA di DB → SQL error
    'scan_keterangan'       => null,                   // ← harusnya diisi di sini
    'waktu_datang'          => null,
]);
```

`presensi_pesertas` tidak punya kolom `keterangan`. Model pakai `$guarded = []` → Eloquent forward ke SQL → **query error** saat keliling simpan keterangan. Keterangan Alfa/Sakit/Izin tidak pernah tersimpan.

### Fix

```php
// SESUDAH — BENAR
$presensi = PresensiPeserta::create([
    'kode_peserta'          => $request->nomor_peserta,
    'ujian_id'              => $jadwal->ujian_id,
    'ruang_id'              => $jadwal->ruang_id,      // ← Bug 2 juga difix di sini
    'panitia_id'            => $request->panitia_id,
    'scanned_by_panitia_id' => $request->panitia_id,
    'scan_keterangan'       => $request->keterangan,   // ← pindah ke kolom yang benar
    'waktu_datang'          => null,
]);
```

---

## Bug 2 — KRITIS: `ruang_id` Tidak Diset

### Root Cause
Presensi dari keliling tidak menyimpan `ruang_id`. Tabel `presensi_pesertas` punya kolom `ruang_id` (NOT NULL dengan nilai dari jadwal).

Akibatnya: siswa absen yang dicatat keliling tidak masuk rekap berbasis ruang (filter ruang di Manual Kehadiran tidak akan menemukan mereka).

### Fix
Tambahkan `'ruang_id' => $jadwal->ruang_id` ke array create (sudah ditunjukkan di Fix Bug 1 di atas). Pastikan `$jadwal` sudah di-fetch sebelum create (sudah ada di kode: `$jadwal = JadwalUjian::findOrFail($request->jadwal_id);`).

---

## Bug 3 — MINOR: Status Berita Acara Salah Jika Ada Pengawas Pengganti

### Root Cause
Di method `jadwalAktif()` dan `siswaBelumScan()`, lookup status BA pakai operator `??`:

```php
// SEKARANG — SALAH
$pengawasId = $j->pengawas_pengganti_id ?? $j->pengawas_id;
$sudahLaporan = LaporanUjian::where('ujian_id', $j->ujian_id)
    ->where('pengawas_id', $pengawasId)   // ← hanya cek SATU pengawas
    ->where('mulai_ujian', $j->mulai_ujian)
    ->exists();
```

Jika jadwal punya `pengawas_pengganti_id`, query HANYA cek laporan dari pengganti. Jika pengawas asli yang submit laporan, status tampil "Belum diisi" — misleading untuk petugas keliling.

### Fix — Terapkan di KEDUA method (`jadwalAktif` dan `siswaBelumScan`)

```php
// SESUDAH — BENAR
$pengawasIds = array_values(array_filter([
    $j->pengawas_id,
    $j->pengawas_pengganti_id,
]));
$sudahLaporan = !empty($pengawasIds) && LaporanUjian::where('ujian_id', $j->ujian_id)
    ->whereIn('pengawas_id', $pengawasIds)   // ← cek salah satu dari keduanya
    ->where('mulai_ujian', $j->mulai_ujian)
    ->exists();
```

Untuk `siswaBelumScan()`, variabel yang dipakai adalah `$jadwal` (bukan `$j`), sesuaikan namanya:
```php
$pengawasIds = array_values(array_filter([
    $jadwal->pengawas_id,
    $jadwal->pengawas_pengganti_id,
]));
$sudahLaporan = !empty($pengawasIds) && LaporanUjian::where('ujian_id', $jadwal->ujian_id)
    ->whereIn('pengawas_id', $pengawasIds)
    ->where('mulai_ujian', $jadwal->mulai_ujian)
    ->exists();
```

---

## Files yang Diubah

| File | Perubahan |
|------|-----------|
| `backend/app/Http/Controllers/KelilingController.php` | Bug 1 + Bug 2: fix `simpanKeterangan()` — `scan_keterangan` + `ruang_id` |
| `backend/app/Http/Controllers/KelilingController.php` | Bug 3: fix `jadwalAktif()` + `siswaBelumScan()` — `whereIn` kedua pengawas |

**Tidak ada file lain yang perlu diubah.**

---

## Acceptance Criteria

- [ ] `POST /keliling/keterangan` berhasil tanpa SQL error
- [ ] Record presensi tersimpan dengan `scan_keterangan` = "Alfa"/"Sakit"/"Izin"/teks custom
- [ ] Record presensi tersimpan dengan `ruang_id` yang benar (dari jadwal terkait)
- [ ] `GET /keliling/jadwal-aktif`: jadwal dengan pengawas pengganti → status BA "Sudah disubmit" jika SALAH SATU (asli atau pengganti) sudah submit
- [ ] `GET /keliling/siswa-belum-scan`: status BA sama seperti di atas
- [ ] Update CONTEXT.md setelah selesai

## Validasi Claudian
- [x] Bug 1 + 2 verified: DESCRIBE presensi_pesertas tidak ada kolom `keterangan`, ada kolom `ruang_id`
- [x] Model pakai `$guarded = []` → kolom tidak valid akan forward ke SQL → hard error
- [x] Bug 3 verified: logika `??` tidak cukup untuk OR check dua ID
- [x] Fix tidak butuh migration baru — hanya menggunakan kolom yang sudah ada
- [x] Scope: 1 file, 3 method, perubahan minimal
