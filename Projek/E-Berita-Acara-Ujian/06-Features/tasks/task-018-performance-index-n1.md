---
tags:
  - task
  - performance
  - backend
created: 2026-06-08
status: ready
blocked_by: task-017
---

# Task-018: Performance — DB Index + Non-Sargable Query + N+1 Fix

## Objective
Perbaiki tiga sumber lambat: missing index, `whereDate()` non-sargable, N+1 queries.
Target: setiap scan barcode < 300ms; refresh TV (5.000 siswa) < 500ms.

## Context
- Eksekusi SETELAH task-017 selesai
- Jangan ubah logic bisnis — ini pure performance fix
- Buat migration baru, JANGAN edit migration lama

---

## Fix 1 — Migration: Tambah Index yang Hilang

Buat file migration baru: `xxxx_add_performance_indexes.php`

```php
Schema::table('presensi_pesertas', function (Blueprint $table) {
    // Composite index untuk query utama scan: WHERE kode_peserta=? AND created_at RANGE
    $table->index(['kode_peserta', 'created_at'], 'idx_presensi_kode_tanggal');
    // Index untuk filter ujian + tanggal (getPresensiToday)
    $table->index(['ujian_id', 'created_at'], 'idx_presensi_ujian_tanggal');
});

Schema::table('pengawas', function (Blueprint $table) {
    $table->index('niy', 'idx_pengawas_niy');
});

Schema::table('panitia', function (Blueprint $table) {
    $table->index('niy', 'idx_panitia_niy');
});
```

Verifikasi `peserta_ujians.nomor_peserta` sudah ada index (biasanya unique constraint = index
otomatis). Jika belum, tambahkan:
```php
Schema::table('peserta_ujians', function (Blueprint $table) {
    // Cek dulu: SHOW INDEX FROM peserta_ujians WHERE Key_name LIKE '%nomor%'
    // Tambah hanya jika belum ada:
    $table->index('nomor_peserta', 'idx_peserta_nomor');
});
```

Jalankan: `php artisan migrate`

---

## Fix 2 — Ganti `whereDate()` dengan Range Query (Sargable)

`whereDate('kolom', $tanggal)` menghasilkan `WHERE DATE(kolom) = '...'` — fungsi pada kolom
membuat index tidak terpakai. Ganti ke range query.

Pattern penggantian (berlaku di SEMUA file berikut):

```php
// SEBELUM (non-sargable):
->whereDate('created_at', $today)
->whereDate('waktu_datang', $today)

// SESUDAH (sargable — gunakan Carbon):
->where('created_at', '>=', $today->copy()->startOfDay())
->where('created_at', '<',  $today->copy()->addDay()->startOfDay())
```

**File yang harus diupdate:**

| File | Method | Pola lama |
|------|---------|-----------|
| `PresensiService.php` | `handleScanPeserta` + `handleLoginNiy` | `whereDate('created_at', $today)` |
| `ExamReportController.php` | `getPresensiToday` | `whereDate('created_at', $today)` |
| `ExamReportController.php` | `getPresensiPengawasToday` | `whereDate('waktu_datang', $today)` |
| `DashboardService.php` | `getAttendanceStats` + `getAttendanceByCampus` + `getAttendanceStudents` | semua `whereDate(...)` |

**Catatan Carbon:** Pastikan `$today` adalah Carbon instance (`Carbon::today()` atau `now()->startOfDay()`), bukan string.

---

## Fix 3 — Hilangkan N+1 di `getPresensiToday`

**File:** `backend/app/Http/Controllers/ExamReportController.php` — method `getPresensiToday`

Saat ini: `Panitia::find($p->scanned_by_panitia_id)` dipanggil satu kali per baris di dalam `map()`.

```php
// SEBELUM (N+1):
->map(function ($p) {
    $arr = $p->toArray();
    $arr['scanned_by_panitia_nama'] = $p->scanned_by_panitia_id
        ? (\App\Models\Panitia::find($p->scanned_by_panitia_id)?->nama ?? null)
        : null;
    return $arr;
})

// SESUDAH (1 query):
// Sebelum map, kumpulkan semua panitia ID yang dibutuhkan:
$panitiaIds = $hadir->pluck('scanned_by_panitia_id')->filter()->unique()->values();
$panitiaMap = \App\Models\Panitia::whereIn('id', $panitiaIds)->get()->keyBy('id');

// Di dalam map:
->map(function ($p) use ($panitiaMap) {
    $arr = $p->toArray();
    $arr['scanned_by_panitia_nama'] = $p->scanned_by_panitia_id
        ? ($panitiaMap->get($p->scanned_by_panitia_id)?->nama ?? null)
        : null;
    return $arr;
})
```

**Implementasi aktual:** Karena `->get()` sudah dipanggil menghasilkan Collection, bisa pakai:
```php
$records = (clone $baseQuery)->whereNotNull('waktu_datang')->orderBy('updated_at','desc')->get();
$panitiaIds = $records->pluck('scanned_by_panitia_id')->filter()->unique();
$panitiaMap = \App\Models\Panitia::whereIn('id', $panitiaIds)->get()->keyBy('id');
$hadir = $records->map(function($p) use ($panitiaMap) {
    $arr = $p->toArray();
    $arr['scanned_by_panitia_nama'] = $panitiaMap->get($p->scanned_by_panitia_id)?->nama;
    return $arr;
});
```

---

## Fix 4 — Hilangkan N+1 di `getAttendanceStudents`

**File:** `backend/app/Services/DashboardService.php` — method `getAttendanceStudents`

Saat ini: `PresensiPeserta::where(...)->first()` dipanggil satu kali **per siswa** di dalam `map()`.
Untuk 5.000 siswa = 5.000 query.

```php
// SESUDAH — 1 query sebelum loop, lookup dari collection:
$students = $query->get();
$nomorPesertaList = $students->pluck('nomor_peserta')->toArray();

// Satu query untuk semua presensi
$presensisQuery = PresensiPeserta::where('ujian_id', $ujianId)
    ->whereNotNull('waktu_datang')
    ->whereIn('kode_peserta', $nomorPesertaList);

if ($date && $date !== 'all') {
    $presensisQuery
        ->where('waktu_datang', '>=', \Carbon\Carbon::parse($date)->startOfDay())
        ->where('waktu_datang', '<',  \Carbon\Carbon::parse($date)->addDay()->startOfDay());
}

$presensis = $presensisQuery->get()->keyBy('kode_peserta'); // lookup O(1)

// Map tanpa query tambahan:
return $students->map(function ($student) use ($ujianId, $presensis, $keterangans) {
    $presensi = $presensis->get($student->nomor_peserta); // lookup dari collection
    $ket      = $keterangans->get($student->nomor_peserta);

    return [
        'id'           => $student->id,
        'nama'         => $student->nama,
        'nomor_peserta'=> $student->nomor_peserta,
        'kelas'        => $student->kelas,
        'nama_ruang'   => $student->ruang->nama_ruang ?? '-',
        'sesi'         => $student->sesi,
        'waktu_datang' => $presensi?->waktu_datang,
        'status'       => $presensi ? 'Hadir' : 'Tidak Hadir',
        'keterangan'   => $ket?->keterangan,
        'catatan'      => $ket?->catatan,
        'petugas_nama' => $ket?->panitia?->nama,
    ];
})->toArray();
```

---

## Files yang Diubah

| File | Fix |
|------|-----|
| `database/migrations/xxxx_add_performance_indexes.php` | BARU — Fix 1: 4 index baru |
| `backend/app/Services/PresensiService.php` | Fix 2: `whereDate` → range query |
| `backend/app/Http/Controllers/ExamReportController.php` | Fix 2 + Fix 3: range query + N+1 fix |
| `backend/app/Services/DashboardService.php` | Fix 2 + Fix 4: range query + N+1 fix |

---

## Acceptance Criteria

- [ ] Migration berhasil, `SHOW INDEX FROM presensi_pesertas` menampilkan `idx_presensi_kode_tanggal` dan `idx_presensi_ujian_tanggal`
- [ ] `SHOW INDEX FROM pengawas` menampilkan index pada `niy`
- [ ] `SHOW INDEX FROM panitia` menampilkan index pada `niy`
- [ ] Scan barcode: response time < 500ms (sebelumnya bisa > 1 detik)
- [ ] `GET /dashboard/attendance-students` untuk 5.000+ siswa: response < 800ms
- [ ] `GET /presensi-today`: tidak ada N+1 (satu query untuk semua panitia nama)
- [ ] Log query Laravel (jika `APP_DEBUG=true`): jumlah query `getAttendanceStudents` turun dari ribuan ke individual
- [ ] Update CONTEXT.md setelah selesai

---

## Catatan: Prioritas 4 (Server) — Diskusi Terpisah

`php artisan serve` adalah single-threaded dev server. Untuk hari-H dengan banyak device
scan bersamaan, pertimbangkan:
- **Opsi A:** PHP-FPM + Nginx (paling stabil untuk produksi)
- **Opsi B:** Laravel Octane + Swoole/RoadRunner (performa tinggi, perlu install extension)
- **Minimal:** Set `APP_DEBUG=false`, `CACHE_STORE=file`, `SESSION_DRIVER=file` di `.env` hari-H

Ini keputusan infrastruktur — diskusikan dengan user sebelum dieksekusi.
