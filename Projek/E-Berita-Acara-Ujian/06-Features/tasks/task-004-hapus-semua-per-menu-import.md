---
tags:
  - task
  - feature
created: 2026-06-04
status: ready
---

# Task-004: Fitur "Hapus Semua" per Menu Import

## Objective
Tambahkan tombol "Hapus Semua Data" di setiap halaman admin yang memiliki fitur import, sehingga admin dapat mereset data salah dan import ulang tanpa harus hapus satu per satu. Tombol ini bersifat per-ujian (hanya hapus data untuk ujian yang sedang dipilih).

## Context
- **Aktor:** Admin
- **Menu terdampak:** Peserta Ujian, Pengawas, Panitia, Jadwal Ujian, Ruang
- **Keputusan:** Opsi B — tombol terpisah (bukan mode di dalam dialog import)
- **Scope hapus:** Entitas + direct dependencies saja. TIDAK hapus presensi dan laporan berita acara.

---

## Cascade Behavior per Entitas

Ini WAJIB diimplementasi sesuai tabel ini. Jangan asumsikan — FK sudah diverifikasi dari database.

| Entitas | Yang Dihapus | Yang TIDAK Dihapus | Level Bahaya |
|---------|--------------|---------------------|--------------|
| **Peserta** | `peserta_ujians` (CASCADE → `jadwal_peserta`) | `presensi_pesertas` (pakai string `kode_peserta`, bukan FK) | 🟡 Sedang |
| **Pengawas** | `pengawas` (CASCADE → `jadwal_ujians` → `jadwal_peserta`) + **CASCADE → `laporan_ujians`** + **CASCADE → `presensi_pengawas`** | - | 🔴 TINGGI |
| **Panitia** | `panitia` (CASCADE → `presensi_pengawas` untuk panitia) | `keterangan_ketidakhadiran.panitia_id SET NULL`, `catatan_ketidaktertiban.panitia_id SET NULL` | 🟡 Sedang |
| **Jadwal** | `jadwal_ujians` (CASCADE → `jadwal_peserta`) | `laporan_ujians` (tidak ada FK langsung jadwal→laporan), `presensi_pesertas` | 🟡 Sedang |
| **Ruang** | `ruangs` (SET NULL di jadwal+peserta+presensi) | Semua record tetap ada, hanya `ruang_id` jadi NULL | 🟢 Rendah |

---

## Spec Backend

### Endpoint Baru

Tambahkan 5 endpoint baru di `routes/api.php` (dalam middleware `auth:sanctum`):

```php
Route::delete('/peserta-ujian/reset', [\App\Http\Controllers\PesertaUjianController::class, 'resetByUjian']);
Route::delete('/pengawas/reset', [PengawasController::class, 'resetByUjian']);
Route::delete('/panitia/reset', [PanitiaController::class, 'resetByUjian']);
Route::delete('/jadwal-ujian/reset', [JadwalUjianController::class, 'resetByUjian']);
Route::delete('/ruang/reset', [App\Http\Controllers\RuangController::class, 'resetByUjian']);
```

### Method `resetByUjian()` per Controller

**1. PesertaUjianController — Hapus Peserta + Pivot**

```php
public function resetByUjian(Request $request)
{
    $request->validate(['ujian_id' => 'required|exists:ujians,id']);
    $ujianId = $request->ujian_id;

    // Cek apakah ada presensi aktif — untuk info ke admin, BUKAN untuk block
    $presensiCount = \App\Models\PresensiPeserta::where('ujian_id', $ujianId)->count();

    $count = \App\Models\PesertaUjian::where('ujian_id', $ujianId)->count();

    // DELETE peserta — cascade ke jadwal_peserta otomatis
    \App\Models\PesertaUjian::where('ujian_id', $ujianId)->delete();

    return response()->json([
        'message' => "Berhasil menghapus $count peserta dan pivot jadwalnya.",
        'presensi_warning' => $presensiCount > 0
            ? "Data presensi ($presensiCount record) TIDAK dihapus dan tetap tersimpan."
            : null
    ]);
}
```

**2. PengawasController — Hapus Pengawas (dengan BLOCK jika ada presensi/laporan)**

```php
public function resetByUjian(Request $request)
{
    $request->validate(['ujian_id' => 'required|exists:ujians,id']);
    $ujianId = $request->ujian_id;

    // CEK DULU: apakah ada laporan atau presensi yang akan ikut terhapus?
    $laporanCount = \App\Models\LaporanUjian::where('ujian_id', $ujianId)->count();
    $presensiCount = \App\Models\PresensiPengawas::where('ujian_id', $ujianId)
        ->whereNotNull('pengawas_id')->count();

    // BLOKIR jika ada laporan — ini data permanen yang tidak boleh hilang tanpa konfirmasi eksplisit
    if ($laporanCount > 0) {
        return response()->json([
            'message' => "Tidak dapat menghapus pengawas karena ada $laporanCount Berita Acara yang sudah disubmit. Hapus laporan terlebih dahulu jika memang diperlukan.",
            'blocked' => true,
            'laporan_count' => $laporanCount
        ], 422);
    }

    // BLOKIR jika ada presensi (ujian sedang/sudah berlangsung)
    if ($presensiCount > 0) {
        return response()->json([
            'message' => "Tidak dapat menghapus pengawas karena ada $presensiCount data presensi pengawas. Ujian kemungkinan sudah/sedang berlangsung.",
            'blocked' => true,
            'presensi_count' => $presensiCount
        ], 422);
    }

    $count = \App\Models\Pengawas::where('ujian_id', $ujianId)->count();
    // CASCADE: jadwal_ujians → jadwal_peserta, laporan (kosong karena sudah dicek)
    \App\Models\Pengawas::where('ujian_id', $ujianId)->delete();

    return response()->json([
        'message' => "Berhasil menghapus $count pengawas beserta jadwalnya."
    ]);
}
```

**3. PanitiaController — Hapus Panitia (BLOCK jika ada presensi)**

```php
public function resetByUjian(Request $request)
{
    $request->validate(['ujian_id' => 'required|exists:ujians,id']);
    $ujianId = $request->ujian_id;

    $presensiCount = \App\Models\PresensiPengawas::where('ujian_id', $ujianId)
        ->whereNotNull('panitia_id')->count();

    if ($presensiCount > 0) {
        return response()->json([
            'message' => "Tidak dapat menghapus panitia karena ada $presensiCount data presensi panitia. Ujian kemungkinan sudah/sedang berlangsung.",
            'blocked' => true,
            'presensi_count' => $presensiCount
        ], 422);
    }

    $count = \App\Models\Panitia::where('ujian_id', $ujianId)->count();
    \App\Models\Panitia::where('ujian_id', $ujianId)->delete();

    return response()->json([
        'message' => "Berhasil menghapus $count panitia."
    ]);
}
```

**4. JadwalUjianController — Hapus Jadwal (BLOCK jika ada laporan)**

```php
public function resetByUjian(Request $request)
{
    $request->validate(['ujian_id' => 'required|exists:ujians,id']);
    $ujianId = $request->ujian_id;

    $laporanCount = \App\Models\LaporanUjian::where('ujian_id', $ujianId)->count();

    if ($laporanCount > 0) {
        return response()->json([
            'message' => "Tidak dapat menghapus jadwal karena ada $laporanCount Berita Acara yang sudah disubmit.",
            'blocked' => true,
            'laporan_count' => $laporanCount
        ], 422);
    }

    $count = \App\Models\JadwalUjian::where('ujian_id', $ujianId)->count();
    // CASCADE: jadwal_peserta
    \App\Models\JadwalUjian::where('ujian_id', $ujianId)->delete();

    return response()->json([
        'message' => "Berhasil menghapus $count jadwal dan pivot pesertanya."
    ]);
}
```

**5. RuangController — Hapus Ruang (aman: SET NULL, tidak cascade delete)**

```php
public function resetByUjian(Request $request)
{
    $request->validate(['ujian_id' => 'required|exists:ujians,id']);
    $ujianId = $request->ujian_id;

    $count = \App\Models\Ruang::where('ujian_id', $ujianId)->count();
    // FK behavior: SET NULL di jadwal+peserta+presensi (tidak delete)
    \App\Models\Ruang::where('ujian_id', $ujianId)->delete();

    return response()->json([
        'message' => "Berhasil menghapus $count ruang. Data jadwal dan peserta yang mereferensi ruang ini akan kehilangan info ruangan (tidak dihapus)."
    ]);
}
```

---

## Spec Frontend (frontend-admin/)

### Desain Tombol

Tambahkan tombol "🗑️ Hapus Semua" di setiap halaman manajemen yang punya import, di sebelah tombol "Import". Gunakan warna merah/danger.

**Posisi:** Di toolbar atas halaman, sejajar dengan tombol Import dan tambah manual.

**Tampilan:**
```jsx
<button
  onClick={handleResetAll}
  className="flex items-center gap-2 px-4 py-2 bg-red-50 text-red-600 border border-red-200 
             rounded-xl text-sm font-bold hover:bg-red-100 transition-colors"
>
  <Trash2 size={16} />
  Hapus Semua
</button>
```

### Dialog Konfirmasi Wajib (SweetAlert2)

Sebelum mengirim request DELETE, tampilkan dialog konfirmasi dua tahap:

```jsx
const handleResetAll = async () => {
  if (!selectedUjianId) {
    return Swal.fire('Pilih Ujian', 'Pilih ujian terlebih dahulu.', 'warning');
  }

  // Konfirmasi tahap 1
  const result = await Swal.fire({
    title: '⚠️ Hapus Semua [Entitas]?',
    html: `Ini akan menghapus <strong>semua [entitas]</strong> untuk ujian ini.<br><br>
           <span style="color:#ef4444">Aksi ini tidak dapat dibatalkan.</span>`,
    icon: 'warning',
    showCancelButton: true,
    confirmButtonColor: '#ef4444',
    confirmButtonText: 'Ya, Hapus Semua',
    cancelButtonText: 'Batal',
    input: 'checkbox',
    inputValue: 0,
    inputPlaceholder: 'Saya mengerti data ini akan dihapus permanen',
    inputValidator: (result) => {
      return !result && 'Centang konfirmasi terlebih dahulu'
    }
  });

  if (!result.isConfirmed) return;

  try {
    const res = await axios.delete('/api/[endpoint]/reset', {
      params: { ujian_id: selectedUjianId }
    });

    Swal.fire({
      icon: 'success',
      title: 'Berhasil Dihapus',
      html: res.data.message + (res.data.presensi_warning 
        ? `<br><small style="color:#f59e0b">⚠️ ${res.data.presensi_warning}</small>` 
        : ''),
    });
    
    fetchData(); // Reload tabel
  } catch (err) {
    const msg = err.response?.data?.message || 'Gagal menghapus data';
    Swal.fire({
      icon: err.response?.data?.blocked ? 'warning' : 'error',
      title: err.response?.data?.blocked ? 'Tidak Dapat Dihapus' : 'Gagal',
      text: msg
    });
  }
};
```

### Halaman yang Perlu Diupdate
Tambahkan tombol dan handler di:
- `frontend-admin/src/pages/Students.jsx` — tombol "Hapus Semua Peserta"
- `frontend-admin/src/pages/Proctors.jsx` — tombol "Hapus Semua Pengawas"
- `frontend-admin/src/pages/Panitia.jsx` — tombol "Hapus Semua Panitia"
- `frontend-admin/src/pages/ExamSchedule.jsx` — tombol "Hapus Semua Jadwal"
- `frontend-admin/src/pages/Rooms.jsx` — tombol "Hapus Semua Ruang"

Catatan: Setiap halaman sudah punya `selectedUjian` state. Gunakan itu sebagai `ujian_id` di request.

---

## Files
- **Modifikasi Backend:** `routes/api.php` (5 route baru)
- **Modifikasi Backend:** `PesertaUjianController.php`, `PengawasController.php`, `PanitiaController.php`, `JadwalUjianController.php`, `RuangController.php` (1 method baru per file)
- **Modifikasi Frontend:** `Students.jsx`, `Proctors.jsx`, `Panitia.jsx`, `ExamSchedule.jsx`, `Rooms.jsx` (tombol + handler per file)
- **Jangan sentuh:** Model, Migration, Service

## Acceptance Criteria

**Peserta:**
- [ ] Tombol "Hapus Semua" muncul di halaman Peserta Ujian
- [ ] Klik tombol → dialog konfirmasi dengan checkbox muncul
- [ ] Konfirmasi → peserta terhapus, tabel kosong
- [ ] `jadwal_peserta` untuk ujian ini ikut terhapus (cek via database)
- [ ] `presensi_pesertas` TIDAK terhapus (cek via database)
- [ ] Jika ada presensi aktif → pesan warning tampil di SweetAlert sukses

**Pengawas:**
- [ ] Jika ada laporan → request di-BLOCK (HTTP 422), pesan jelas
- [ ] Jika ada presensi pengawas → request di-BLOCK (HTTP 422), pesan jelas
- [ ] Jika tidak ada laporan + presensi → hapus berhasil, jadwal ikut terhapus

**Panitia:**
- [ ] Jika ada presensi panitia → request di-BLOCK (HTTP 422)
- [ ] Jika tidak ada presensi → hapus berhasil

**Jadwal:**
- [ ] Jika ada laporan → request di-BLOCK (HTTP 422)
- [ ] Jika tidak ada laporan → hapus berhasil, `jadwal_peserta` ikut terhapus
- [ ] `laporan_ujians` TIDAK terhapus (cek via database)

**Ruang:**
- [ ] Hapus berhasil tanpa block
- [ ] Jadwal dan peserta yang mereferensi ruang ini masih ada (ruang_id jadi NULL)

**Semua:**
- [ ] Dialog konfirmasi memiliki checkbox sebelum tombol aktif
- [ ] CONTEXT.md diupdate

## Validasi Claudian
- [x] Cascade behavior sudah diverifikasi dari FK database langsung
- [x] Block pada Pengawas dan Panitia/Jadwal melindungi data produksi
- [x] Tidak ada perubahan schema
- [x] Frontend: konfirmasi dua tahap (dialog + checkbox) mencegah klik tidak sengaja
