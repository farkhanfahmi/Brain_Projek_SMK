---
tags:
  - task
  - feature
  - backend
created: 2026-06-07
status: ready
---

# Task-011: Backend — Keliling Overview Endpoint + Photo Upload Foundation

## Objective
Buat backend untuk redesign halaman keliling (one-page grouped) dan tambah fitur upload foto pada presensi keliling + PDS.

## Context
- Spec fitur: `06-Features/feature-006-monitoring-petugas.md`
- **Jangan sentuh:** Frontend — itu scope task-012
- **Jangan refactor** di luar scope task

---

## 1. Migration: Tambah `attachment_path` ke `presensi_pesertas`

```php
// File baru: database/migrations/xxxx_add_attachment_to_presensi_pesertas_table.php
Schema::table('presensi_pesertas', function (Blueprint $table) {
    $table->string('attachment_path')->nullable()->after('scan_keterangan');
});
```

Jalankan: `php artisan migrate`

Tambahkan `'attachment_path'` ke `$fillable` di `PresensiPeserta` model.

---

## 2. Endpoint Baru: `GET /keliling/overview`

Gantikan dua endpoint lama (`jadwal-aktif` dan `siswa-belum-scan`) dengan **satu endpoint aggregated** yang return semua data sekaligus.

### Route
```php
Route::get('/keliling/overview', [KelilingController::class, 'overview']);
```

### Logic Scope Waktu

```
Jika ada sesi yang sedang aktif (NOW between mulai-selesai):
    → tampilkan jadwal hari ini yang sudah mulai (mulai_ujian <= NOW)
    → ini mencakup: sesi aktif + sesi sebelumnya hari ini

Jika tidak ada sesi aktif (semua selesai atau belum mulai):
    → tampilkan SEMUA jadwal hari ini (apapun waktunya)
    → kasus ini terjadi saat ujian sudah selesai semua
```

### Filter Tampil

Sebuah jadwal HANYA ditampilkan jika memenuhi MINIMAL SATU dari:
- Ada siswa yang belum terscan (`total_belum > 0`)
- Ada pengawas pengganti (`pengawas_pengganti_id IS NOT NULL`)
- Berita acara belum disubmit (`laporan NOT EXISTS`)

### Implementasi

```php
public function overview(Request $request)
{
    $request->validate(['ujian_id' => 'required|exists:ujians,id']);

    $ujianId = $request->ujian_id;
    $now     = now();
    $today   = today();

    // Tentukan scope waktu
    $hasActiveSession = JadwalUjian::where('ujian_id', $ujianId)
        ->whereDate('mulai_ujian', $today)
        ->where('mulai_ujian', '<=', $now)
        ->where('ujian_berakhir', '>=', $now)
        ->exists();

    $jadwalsQuery = JadwalUjian::with(['ruang', 'pengawas', 'pengawasPengganti', 'mataPelajaran'])
        ->where('ujian_id', $ujianId)
        ->whereDate('mulai_ujian', $today);

    if ($hasActiveSession) {
        // Aktif + sebelumnya: sudah mulai
        $jadwalsQuery->where('mulai_ujian', '<=', $now);
    }
    // Jika tidak ada sesi aktif: tidak ada filter waktu tambahan (semua hari ini)

    $jadwals = $jadwalsQuery->orderBy('mulai_ujian')->get();

    $result = [];

    foreach ($jadwals as $jadwal) {
        // Siswa belum scan hari ini untuk jadwal ini
        $sudahScanNomors = PresensiPeserta::where('ujian_id', $ujianId)
            ->whereDate('created_at', $today)
            ->pluck('kode_peserta')
            ->toArray();

        $belumScan = DB::table('jadwal_peserta')
            ->join('peserta_ujians', 'jadwal_peserta.peserta_ujian_id', '=', 'peserta_ujians.id')
            ->where('jadwal_peserta.jadwal_ujian_id', $jadwal->id)
            ->whereNotIn('peserta_ujians.nomor_peserta', $sudahScanNomors)
            ->select('peserta_ujians.id', 'peserta_ujians.nama',
                     'peserta_ujians.nomor_peserta', 'peserta_ujians.kelas')
            ->orderBy('peserta_ujians.nama')
            ->get();

        $totalBelum = $belumScan->count();
        $hasPengganti = !is_null($jadwal->pengawas_pengganti_id);

        // Cek status laporan
        $pengawasIds = array_values(array_filter([
            $jadwal->pengawas_id,
            $jadwal->pengawas_pengganti_id,
        ]));
        $sudahLaporan = !empty($pengawasIds) && LaporanUjian::where('ujian_id', $ujianId)
            ->whereIn('pengawas_id', $pengawasIds)
            ->where('mulai_ujian', $jadwal->mulai_ujian)
            ->exists();

        // Filter: tampilkan hanya jika memenuhi salah satu kondisi
        $perluTampil = $totalBelum > 0 || $hasPengganti || !$sudahLaporan;

        if (!$perluTampil) continue;

        $result[] = [
            'jadwal_id'           => $jadwal->id,
            'ruang'               => $jadwal->ruang?->nama_ruang ?? '-',
            'sesi'                => $jadwal->sesi,
            'nama_mapel'          => $jadwal->nama_mapel ?? ($jadwal->mataPelajaran?->nama_mapel ?? '-'),
            'mulai_ujian'         => $jadwal->mulai_ujian,
            'ujian_berakhir'      => $jadwal->ujian_berakhir,
            'pengawas'            => $jadwal->pengawas?->name,
            'pengawas_niy'        => $jadwal->pengawas?->niy,
            'pengawas_pengganti'  => $jadwal->pengawasPengganti?->name,
            'status_berita_acara' => $sudahLaporan ? 'Sudah disubmit' : 'Belum diisi',
            'total_belum'         => $totalBelum,
            'siswa_belum_scan'    => $belumScan,
        ];
    }

    return response()->json([
        'has_active_session' => $hasActiveSession,
        'total_ruang'        => count($result),
        'data'               => $result,
    ]);
}
```

**Catatan performa:** Jika ada banyak ruang, query `$sudahScanNomors` bisa dioptimasi dengan di-fetch sekali di luar loop. Lakukan optimasi ini:

```php
// Ambil SEKALI di luar loop, bukan di dalam
$sudahScanNomors = PresensiPeserta::where('ujian_id', $ujianId)
    ->whereDate('created_at', $today)
    ->pluck('kode_peserta')
    ->toArray();

foreach ($jadwals as $jadwal) {
    // Gunakan $sudahScanNomors yang sudah di-fetch
    $belumScan = DB::table('jadwal_peserta')
        ->join(...)
        ->whereNotIn('peserta_ujians.nomor_peserta', $sudahScanNomors)
        ...
```

---

## 3. Modifikasi `simpanKeterangan` — Tambah File Upload (WAJIB untuk keterangan tertentu)

Ubah method `simpanKeterangan` di `KelilingController`:

**Aturan foto:**
- **WAJIB** jika `keterangan` adalah: `Sakit`, `Izin`, `PKL`, `Dispen` (butuh surat bukti)
- **OPSIONAL** jika `keterangan` adalah: `Alfa`, `Lainnya`

```php
public function simpanKeterangan(Request $request)
{
    $KETERANGAN_WAJIB_FOTO = ['Sakit', 'Izin', 'PKL', 'Dispen'];
    $isWajibFoto = in_array($request->keterangan, $KETERANGAN_WAJIB_FOTO);

    $request->validate([
        'jadwal_id'      => 'required|exists:jadwal_ujians,id',
        'nomor_peserta'  => 'required|string',
        'panitia_id'     => 'required|exists:panitia,id',
        'keterangan'     => 'required|string|max:255',
        'attachment'     => ($isWajibFoto ? 'required' : 'nullable') . '|file|mimes:jpg,jpeg,png,pdf|max:5120',
    ]);

    // ... validasi panitia is_keliling dan cek existing presensi (tidak berubah) ...

    $attachmentPath = null;
    if ($request->hasFile('attachment') && $request->file('attachment')->isValid()) {
        $attachmentPath = $request->file('attachment')
            ->store('presensi-attachments', 'public');
    }

    $presensi = PresensiPeserta::create([
        'kode_peserta'          => $request->nomor_peserta,
        'ujian_id'              => $jadwal->ujian_id,
        'ruang_id'              => $jadwal->ruang_id,
        'panitia_id'            => $request->panitia_id,
        'scanned_by_panitia_id' => $request->panitia_id,
        'scan_keterangan'       => $request->keterangan,
        'attachment_path'       => $attachmentPath,
        'waktu_datang'          => null,
    ]);

    return response()->json([
        'message' => 'Keterangan berhasil disimpan.',
        'data'    => $presensi,
        'attachment_url' => $attachmentPath
            ? asset('storage/' . $attachmentPath)
            : null,
    ]);
}
```

---

## 4. Modifikasi Endpoint Scan PDS — Tambah File Upload

Di `ExamReportController` (method yang handle scan presensi panitia), tambahkan handling file upload opsional:

```php
// Di dalam method scanPeserta / handleScanPresensi:
$request->validate([
    // ... validasi existing ...
    'attachment' => 'nullable|file|mimes:jpg,jpeg,png,pdf|max:5120',
]);

$attachmentPath = null;
if ($request->hasFile('attachment') && $request->file('attachment')->isValid()) {
    $attachmentPath = $request->file('attachment')
        ->store('presensi-attachments', 'public');
}

// Pass $attachmentPath ke PresensiService atau simpan langsung ke presensi
```

Di `PresensiService::processPesertaAttendance()`, tambahkan parameter `$attachmentPath = null` dan sertakan ke array create:

```php
'attachment_path' => $attachmentPath,
```

---

## 5. Fix `getPresensiToday` — Exclude Keliling Absent Records

Di `ExamReportController::getPresensiToday()` (sekitar baris 441), tambahkan filter `whereNotNull('waktu_datang')`:

```php
// SEBELUM — return semua presensi, termasuk keliling (waktu_datang = null)
$query = \App\Models\PresensiPeserta::whereDate('created_at', $today)
    ->orderBy('updated_at', 'desc');

// SESUDAH — hanya return presensi "hadir" (waktu_datang terisi)
$query = \App\Models\PresensiPeserta::whereDate('created_at', $today)
    ->whereNotNull('waktu_datang')   // ← TAMBAHKAN INI
    ->orderBy('updated_at', 'desc');
```

**Mengapa:** Presensi yang dibuat oleh petugas keliling untuk siswa absen memiliki `waktu_datang = NULL`. Tanpa filter ini, siswa absen tersebut muncul di list "Siswa Hadir" pada view pengawas — karena frontend `presentInRoom` hanya mengecek KEBERADAAN presensi, bukan isi `waktu_datang`.

---

## 6. Pastikan Storage Link Aktif

```bash
php artisan storage:link
```

Verifikasi folder `public/storage/presensi-attachments` bisa diakses.

---

## 6. Route: Hapus/Deprecate Endpoint Lama (Opsional)

Endpoint lama `GET /keliling/jadwal-aktif` dan `GET /keliling/siswa-belum-scan` bisa dipertahankan (tidak dihapus) agar tidak breaking jika ada client lain yang pakai. Tapi fokus adalah endpoint baru `GET /keliling/overview`.

---

## Files yang Diubah / Dibuat

| File | Aksi |
|------|------|
| `database/migrations/xxxx_add_attachment_to_presensi_pesertas_table.php` | BARU |
| `app/Models/PresensiPeserta.php` | MODIFIKASI — tambah `attachment_path` ke fillable |
| `app/Http/Controllers/KelilingController.php` | MODIFIKASI — tambah `overview()`, ubah `simpanKeterangan()` |
| `app/Http/Controllers/ExamReportController.php` | MODIFIKASI — tambah file upload di scan PDS |
| `app/Services/PresensiService.php` | MODIFIKASI — tambah `$attachmentPath` di processPesertaAttendance |
| `routes/api.php` | MODIFIKASI — tambah route `GET /keliling/overview` |

---

## Acceptance Criteria

- [ ] Migration berhasil, `presensi_pesertas` punya kolom `attachment_path`
- [ ] `GET /keliling/overview?ujian_id=X`:
  - [ ] Hanya return ruang yang memenuhi MINIMAL SATU dari: ada siswa belum scan, ada pengganti, BA belum diisi
  - [ ] Jika ada sesi aktif: hanya jadwal yang sudah mulai (aktif + sebelumnya)
  - [ ] Jika tidak ada sesi aktif: semua jadwal hari ini
  - [ ] Setiap jadwal menyertakan `siswa_belum_scan` array
  - [ ] Field `has_active_session` ada di response
- [ ] `POST /keliling/keterangan` dengan keterangan Sakit/Izin/PKL/Dispen TANPA file → return 422 validation error
- [ ] `POST /keliling/keterangan` dengan keterangan Sakit/Izin/PKL/Dispen DENGAN file → berhasil, `attachment_path` tersimpan
- [ ] `POST /keliling/keterangan` dengan keterangan Alfa/Lainnya tanpa file → berhasil (foto opsional)
- [ ] Scan PDS dengan file → `attachment_path` tersimpan
- [ ] Scan PDS tanpa file → tetap berhasil (foto opsional untuk PDS)
- [ ] `GET /presensi-today` tidak lagi mengembalikan presensi dengan `waktu_datang = NULL`
- [ ] Siswa absen yang dicatat keliling TIDAK muncul di list "Siswa Hadir" pengawas
- [ ] `storage:link` aktif, file bisa diakses via URL
- [ ] Update CONTEXT.md setelah selesai
