---
tags:
  - task
  - feature
  - backend
created: 2026-06-05
status: ready
---

# Task-006: Backend Foundation — Sistem Monitoring Petugas

## Objective
Buat fondasi backend untuk fitur monitoring: migrasi DB, flag panitia, atribusi presensi, dan semua endpoint yang dibutuhkan task-007 dan task-008.

## Context
- Spec lengkap: `06-Features/feature-006-monitoring-petugas.md`
- **BACA spec itu dulu sebelum mulai**
- Jangan sentuh frontend — itu scope task-007 dan task-008

---

## 1. Migrations

### Migration A: tambah flag ke tabel `panitia`

```php
// File baru: database/migrations/xxxx_add_monitoring_flags_to_panitia_table.php
Schema::table('panitia', function (Blueprint $table) {
    $table->boolean('is_pds')->default(false)->after('can_scan');
    $table->boolean('is_keliling')->default(false)->after('is_pds');
});
```

### Migration B: tambah atribusi ke tabel `presensi_pesertas`

```php
// File baru: database/migrations/xxxx_add_scan_attribution_to_presensi_pesertas_table.php
Schema::table('presensi_pesertas', function (Blueprint $table) {
    $table->unsignedBigInteger('scanned_by_panitia_id')->nullable()->after('panitia_id');
    $table->string('scan_keterangan', 255)->nullable()->after('scanned_by_panitia_id');
    $table->foreign('scanned_by_panitia_id')
          ->references('id')->on('panitia')
          ->onDelete('set null');
});
```

Jalankan: `php artisan migrate`

---

## 2. Model Updates

### `app/Models/Panitia.php`
Tambahkan ke `$fillable`:
```php
'is_pds', 'is_keliling'
```

### `app/Models/PresensiPeserta.php`
Tambahkan ke `$fillable`:
```php
'scanned_by_panitia_id', 'scan_keterangan'
```

Tambahkan relasi:
```php
public function scannedByPanitia()
{
    return $this->belongsTo(Panitia::class, 'scanned_by_panitia_id');
}
```

---

## 3. PanitiaController — Toggle Endpoints

Tambahkan 2 method baru di `app/Http/Controllers/PanitiaController.php`:

```php
public function togglePds(Request $request, $id)
{
    $panitia = Panitia::findOrFail($id);
    
    // Jika akan diaktifkan, pastikan is_keliling tidak aktif
    if (!$panitia->is_pds && $panitia->is_keliling) {
        return response()->json([
            'message' => 'Panitia ini sudah ditandai sebagai Petugas Keliling. Nonaktifkan dulu sebelum mengaktifkan PDS.'
        ], 422);
    }
    
    $panitia->update(['is_pds' => !$panitia->is_pds]);
    return response()->json(['message' => 'Status PDS diperbarui.', 'data' => $panitia]);
}

public function toggleKeliling(Request $request, $id)
{
    $panitia = Panitia::findOrFail($id);
    
    // Jika akan diaktifkan, pastikan is_pds tidak aktif
    if (!$panitia->is_keliling && $panitia->is_pds) {
        return response()->json([
            'message' => 'Panitia ini sudah ditandai sebagai PDS/Team IT. Nonaktifkan dulu sebelum mengaktifkan Keliling.'
        ], 422);
    }
    
    $panitia->update(['is_keliling' => !$panitia->is_keliling]);
    return response()->json(['message' => 'Status Keliling diperbarui.', 'data' => $panitia]);
}
```

Tambahkan ke `routes/api.php` (dalam group `auth:sanctum`):
```php
Route::patch('/panitia/{id}/toggle-pds', [PanitiaController::class, 'togglePds']);
Route::patch('/panitia/{id}/toggle-keliling', [PanitiaController::class, 'toggleKeliling']);
```

---

## 4. Modifikasi Login Response

Di endpoint login NIY (`PresensiService` atau controller yang handle `/login-niy`), pastikan response menyertakan flag panitia:

```php
// Jika user adalah panitia, tambahkan ke response:
'user' => [
    'id'           => $panitia->id,
    'name'         => $panitia->name,
    'niy'          => $panitia->niy,
    'can_scan'     => $panitia->can_scan,
    'is_pds'       => $panitia->is_pds,       // ← BARU
    'is_keliling'  => $panitia->is_keliling,   // ← BARU
    ...
]
```

Cek file: cari di mana `/login-niy` atau `loginNiy` dihandle dan tambahkan field ini.

---

## 5. Endpoint Baru: Keliling

Buat file baru `app/Http/Controllers/KelilingController.php`:

```php
<?php

namespace App\Http\Controllers;

use App\Models\JadwalUjian;
use App\Models\Panitia;
use App\Models\PresensiPeserta;
use App\Models\PesertaUjian;
use App\Models\LaporanUjian;
use Illuminate\Http\Request;
use Illuminate\Support\Facades\DB;

class KelilingController extends Controller
{
    /**
     * Daftar jadwal yang sesi-nya sedang aktif (NOW between mulai-selesai, hari ini)
     */
    public function jadwalAktif(Request $request)
    {
        $request->validate(['ujian_id' => 'required|exists:ujians,id']);

        $jadwals = JadwalUjian::with(['ruang', 'pengawas', 'pengawasPengganti'])
            ->where('ujian_id', $request->ujian_id)
            ->whereDate('mulai_ujian', today())
            ->where('mulai_ujian', '<=', now())
            ->where('ujian_berakhir', '>=', now())
            ->get()
            ->map(function ($j) {
                $laporan = LaporanUjian::where('jadwal_ujian_id', $j->id)->first();
                return [
                    'jadwal_id'            => $j->id,
                    'nama_mapel'           => $j->nama_mapel,
                    'sesi'                 => $j->sesi,
                    'mulai_ujian'          => $j->mulai_ujian,
                    'ujian_berakhir'       => $j->ujian_berakhir,
                    'ruang'                => $j->ruang?->nama_ruang,
                    'ruang_id'             => $j->ruang_id,
                    'pengawas'             => $j->pengawas?->name,
                    'pengawas_pengganti'   => $j->pengawasPengganti?->name,
                    'status_berita_acara'  => $laporan ? 'Sudah disubmit' : 'Belum diisi',
                ];
            });

        return response()->json($jadwals);
    }

    /**
     * Siswa yang belum terscan untuk jadwal tertentu
     */
    public function siswaBelumScan(Request $request)
    {
        $request->validate(['jadwal_id' => 'required|exists:jadwal_ujians,id']);

        $jadwalId = $request->jadwal_id;

        // Ambil ID peserta yang sudah ada presensi di jadwal ini
        $sudahScanIds = PresensiPeserta::where('jadwal_ujian_id', $jadwalId)
            ->pluck('peserta_ujian_id')
            ->toArray();

        // Ambil peserta di pivot yang BELUM scan
        $belumScan = DB::table('jadwal_peserta')
            ->join('peserta_ujians', 'jadwal_peserta.peserta_ujian_id', '=', 'peserta_ujians.id')
            ->where('jadwal_peserta.jadwal_ujian_id', $jadwalId)
            ->whereNotIn('jadwal_peserta.peserta_ujian_id', $sudahScanIds)
            ->select('peserta_ujians.id', 'peserta_ujians.nama', 'peserta_ujians.nomor_peserta', 'peserta_ujians.kode_peserta')
            ->orderBy('peserta_ujians.nama')
            ->get();

        // Info jadwal + pengawas + status BA
        $jadwal = JadwalUjian::with(['ruang', 'pengawas', 'pengawasPengganti'])
            ->find($jadwalId);
        $laporan = LaporanUjian::where('jadwal_ujian_id', $jadwalId)->first();

        return response()->json([
            'jadwal' => [
                'id'                   => $jadwal->id,
                'nama_mapel'           => $jadwal->nama_mapel,
                'sesi'                 => $jadwal->sesi,
                'ruang'                => $jadwal->ruang?->nama_ruang,
                'pengawas'             => $jadwal->pengawas?->name,
                'pengawas_niy'         => $jadwal->pengawas?->niy,
                'pengawas_pengganti'   => $jadwal->pengawasPengganti?->name,
                'status_berita_acara'  => $laporan ? 'Sudah disubmit' : 'Belum diisi',
            ],
            'siswa_belum_scan' => $belumScan,
            'total'            => $belumScan->count(),
        ]);
    }

    /**
     * Simpan keterangan tidak hadir dari petugas keliling
     */
    public function simpanKeterangan(Request $request)
    {
        $request->validate([
            'jadwal_id'          => 'required|exists:jadwal_ujians,id',
            'peserta_ujian_id'   => 'required|exists:peserta_ujians,id',
            'panitia_id'         => 'required|exists:panitia,id',
            'keterangan'         => 'required|string|max:255',
        ]);

        // Pastikan panitia ini is_keliling
        $panitia = Panitia::findOrFail($request->panitia_id);
        if (!$panitia->is_keliling) {
            return response()->json(['message' => 'Akses ditolak.'], 403);
        }

        // Cek apakah sudah ada presensi
        $existing = PresensiPeserta::where('jadwal_ujian_id', $request->jadwal_id)
            ->where('peserta_ujian_id', $request->peserta_ujian_id)
            ->first();

        if ($existing) {
            return response()->json(['message' => 'Siswa ini sudah memiliki presensi.'], 422);
        }

        $presensi = PresensiPeserta::create([
            'jadwal_ujian_id'       => $request->jadwal_id,
            'peserta_ujian_id'      => $request->peserta_ujian_id,
            'panitia_id'            => $request->panitia_id,
            'scanned_by_panitia_id' => $request->panitia_id,
            'keterangan'            => $request->keterangan,  // Alfa/Sakit/Izin/dll
            'scan_keterangan'       => null,                   // ini untuk PDS, bukan keliling
            'status'                => 'tidak_hadir',
            'waktu_datang'          => null,
        ]);

        return response()->json(['message' => 'Keterangan berhasil disimpan.', 'data' => $presensi]);
    }
}
```

Tambahkan routes (boleh di luar atau dalam `auth:sanctum` — keliling tidak pakai Sanctum karena login via NIY scan):
```php
Route::get('/keliling/jadwal-aktif', [KelilingController::class, 'jadwalAktif']);
Route::get('/keliling/siswa-belum-scan', [KelilingController::class, 'siswaBelumScan']);
Route::post('/keliling/keterangan', [KelilingController::class, 'simpanKeterangan']);
```

---

## 6. Modifikasi Endpoint Scan Peserta (untuk PDS)

Cari endpoint yang dipakai saat panitia scan siswa (kemungkinan di `ExamReportController` atau `PresensiController`). Tambahkan logic berikut:

```php
// Di method yang handle scan presensi oleh panitia:

$panitia = Panitia::find($request->panitia_id);

// Jika PDS: wajib ada keterangan
if ($panitia && $panitia->is_pds) {
    if (empty($request->scan_keterangan)) {
        return response()->json([
            'require_keterangan' => true,
            'message'            => 'Pilih keterangan sebelum konfirmasi.',
            'options'            => ['Terlambat', 'Device Bermasalah', 'Lainnya'],
        ], 200); // 200, bukan error — frontend handle ini sebagai trigger popup
    }
}

// Simpan presensi dengan atribusi
$presensi = PresensiPeserta::create([
    // ... field existing ...
    'scanned_by_panitia_id' => ($panitia && ($panitia->is_pds || $panitia->is_keliling))
                                ? $panitia->id : null,
    'scan_keterangan'       => $request->scan_keterangan ?? null,
]);
```

---

## 7. Modifikasi Response Siswa di Pengawas

Di `AssignmentService` atau endpoint yang return daftar siswa pengawas, tambahkan info presensi atribusi:

```php
// Di mapping setiap siswa, include data presensi:
'presensi' => $presensi ? [
    'status'                => $presensi->status,
    'waktu_datang'          => $presensi->waktu_datang,
    'scanned_by_panitia_id' => $presensi->scanned_by_panitia_id,
    'scan_keterangan'       => $presensi->scan_keterangan,
] : null,
```

---

## 8. Modifikasi Response Laporan (Berita Acara)

Di endpoint GET/submit laporan, tambahkan computed field `catatan_tambahan_pds`:

```php
// Di method yang return data laporan:
$catatanPds = PresensiPeserta::where('jadwal_ujian_id', $laporan->jadwal_ujian_id)
    ->whereNotNull('scan_keterangan')
    ->whereNotNull('scanned_by_panitia_id')
    ->with('pesertaUjian')
    ->get()
    ->map(fn($p) => $p->pesertaUjian->nama . ' ' . $p->scan_keterangan)
    ->join(', ');

$response['catatan_pelaksanaan_full'] = $laporan->catatan_pelaksanaan
    . ($catatanPds ? ' | ' . $catatanPds : '');
// catatan_pelaksanaan di DB tidak diubah — ini hanya di response
```

---

## Files yang Diubah / Dibuat

| File | Aksi |
|------|------|
| `database/migrations/xxxx_add_monitoring_flags_to_panitia_table.php` | BARU |
| `database/migrations/xxxx_add_scan_attribution_to_presensi_pesertas_table.php` | BARU |
| `app/Models/Panitia.php` | MODIFIKASI — fillable + relasi |
| `app/Models/PresensiPeserta.php` | MODIFIKASI — fillable + relasi |
| `app/Http/Controllers/PanitiaController.php` | MODIFIKASI — tambah 2 toggle method |
| `app/Http/Controllers/KelilingController.php` | BARU |
| `routes/api.php` | MODIFIKASI — 5 route baru |
| Endpoint scan peserta (cari lokasi exacts) | MODIFIKASI — PDS logic |
| AssignmentService atau response siswa pengawas | MODIFIKASI — include atribusi presensi |
| Endpoint laporan/berita acara | MODIFIKASI — catatan_pelaksanaan_full |

**Jangan sentuh:** Frontend, seeder, ujian/jadwal/peserta logic yang tidak terkait

---

## Acceptance Criteria

- [ ] Migration berhasil tanpa error. `panitia` punya kolom `is_pds` dan `is_keliling`. `presensi_pesertas` punya `scanned_by_panitia_id` dan `scan_keterangan`
- [ ] `PATCH /panitia/{id}/toggle-pds`: toggle benar, return 422 jika `is_keliling` sudah aktif
- [ ] `PATCH /panitia/{id}/toggle-keliling`: toggle benar, return 422 jika `is_pds` sudah aktif
- [ ] Login response panitia menyertakan `is_pds` dan `is_keliling`
- [ ] `GET /keliling/jadwal-aktif?ujian_id=X`: return hanya jadwal yang waktunya NOW antara mulai-selesai hari ini. Return array kosong jika tidak ada
- [ ] `GET /keliling/siswa-belum-scan?jadwal_id=X`: return siswa yang belum ada presensi di jadwal itu + info jadwal/pengawas/BA
- [ ] `POST /keliling/keterangan`: buat presensi status `tidak_hadir` dengan `scanned_by_panitia_id` terisi. Tolak 403 jika panitia bukan `is_keliling`. Tolak 422 jika sudah ada presensi
- [ ] Scan PDS tanpa `scan_keterangan` → return `require_keterangan: true`. Scan PDS dengan keterangan → presensi tersimpan dengan `scanned_by_panitia_id` + `scan_keterangan`
- [ ] Response daftar siswa pengawas menyertakan info atribusi presensi
- [ ] Response laporan menyertakan `catatan_pelaksanaan_full` dengan append PDS. DB tidak diubah
- [ ] Update CONTEXT.md setelah selesai
