---
tags:
  - task
  - bugfix
  - backend
  - frontend
created: 2026-06-07
status: ready
---

# Task-015: Post-Testing Bug Fixes (6 Bug)

## Ditemukan Dari Pengujian
Bug ditemukan saat testing dengan ujian "tesstt" kelas X TM 1.

---

## Fix 1 — PHP Upload Limit (ROOT CAUSE semua upload gagal)

**Root cause:** `upload_max_filesize = 2M` di PHP config, foto kamera HP biasanya 3–8MB.
Laravel validasi `max:5120` (5MB) tidak pernah dieksekusi karena PHP menolak duluan.
Error "The Attachment / Foto Pelanggaran failed to upload" = Laravel translate untuk PHP's `UPLOAD_ERR_INI_SIZE`.

**Fix:** Buat file baru `backend/public/.user.ini`:

```ini
upload_max_filesize = 10M
post_max_size = 15M
```

Setelah buat file: restart server (`php artisan serve` atau web server).

---

## Fix 2+3 — Backend: `getPresensiToday` — struktur baru + nama scanner

**File:** `backend/app/Http/Controllers/ExamReportController.php` (method `getPresensiToday`, ~baris 448)

**Problem (2):** Response tidak berisi nama panitia yang scan → frontend tidak bisa tampilkan "Diabsen [nama]".

**Problem (3):** Keliling absent presensi (waktu_datang=null) tidak terlihat pengawas → pengawas tidak tahu siswa sudah dicatat Alfa/Sakit oleh keliling.

**Fix:** Ubah `getPresensiToday` agar return object `{ hadir, keliling_absen }` bukan array flat:

```php
public function getPresensiToday(Request $request)
{
    $today = now()->startOfDay();
    $baseQuery = \App\Models\PresensiPeserta::whereDate('created_at', $today);

    if ($request->has('ujian_id') && $request->ujian_id) {
        $baseQuery->where('ujian_id', $request->ujian_id);
    }

    // Presensi hadir — waktu_datang terisi + nama scanner
    $hadir = (clone $baseQuery)
        ->whereNotNull('waktu_datang')
        ->orderBy('updated_at', 'desc')
        ->get()
        ->map(function ($p) {
            $arr = $p->toArray();
            $arr['scanned_by_panitia_nama'] = $p->scanned_by_panitia_id
                ? (\App\Models\Panitia::find($p->scanned_by_panitia_id)?->nama ?? null)
                : null;
            return $arr;
        });

    // Keliling absent — waktu_datang null, ada keterangan
    $kelilingAbsen = (clone $baseQuery)
        ->whereNull('waktu_datang')
        ->whereNotNull('scan_keterangan')
        ->get()
        ->map(fn($p) => [
            'kode_peserta'    => $p->kode_peserta,
            'scan_keterangan' => $p->scan_keterangan,
        ]);

    return response()->json([
        'hadir'          => $hadir,
        'keliling_absen' => $kelilingAbsen,
    ]);
}
```

---

## Fix 4 — Frontend App.jsx: konsumsi response baru + PDS attribution + Device Bermasalah

**File:** `frontend/src/App.jsx`

### 4a. State baru `kelilingAbsen`

Tambahkan state di dekat deklarasi state lainnya (sekitar baris 30):
```jsx
const [kelilingAbsen, setKelilingAbsen] = useState([]);
```

### 4b. Update `fetchPresensi` — konsumsi struktur response baru

```jsx
// Cari baris yang ada: .then(res => setScannedStudents(res.data))
// Ganti dengan:
.then(res => {
    // Response baru: { hadir: [...], keliling_absen: [...] }
    // Fallback untuk backward-compat jika response masih array
    if (Array.isArray(res.data)) {
        setScannedStudents(res.data);
        setKelilingAbsen([]);
    } else {
        setScannedStudents(res.data.hadir || []);
        setKelilingAbsen(res.data.keliling_absen || []);
    }
})
```

### 4c. Update `absentInRoom` — tambah keliling keterangan

```jsx
// Cari: const absentInRoom = assignedStudents.filter(s =>
//     !scannedStudents.some(sc => sc.kode_peserta === s.nomor_peserta)
// );
// Ganti dengan:
const absentInRoom = assignedStudents
    .filter(s => !scannedStudents.some(sc => sc.kode_peserta === s.nomor_peserta))
    .map(s => {
        const kelilingKet = kelilingAbsen.find(k => k.kode_peserta === s.nomor_peserta);
        return { ...s, keliling_keterangan: kelilingKet?.scan_keterangan ?? null };
    });
```

### 4d. Tampilkan nama scanner di row "Siswa Hadir"

Di baris render siswa hadir (sekitar baris 1361-1364), ganti text "Diabsen PDS/Team IT":

```jsx
// Sebelum:
Diabsen PDS/Team IT{s.scan_keterangan ? ` — ${s.scan_keterangan}` : ''}

// Sesudah:
Diabsen {s.scanned_by_panitia_nama || 'PDS/Team IT'}{s.scan_keterangan ? ` — ${s.scan_keterangan}` : ''}
```

### 4e. Tampilkan keterangan keliling di row "Siswa Belum Hadir"

Di render row siswa absent (sekitar baris 1392–1455), cari tempat nama siswa ditampilkan
dan tambahkan badge keterangan keliling:

```jsx
{/* Tambahkan setelah nama/nomor siswa */}
{s.keliling_keterangan && (
    <span className="text-[9px] font-bold bg-orange-100 text-orange-700 px-1.5 py-0.5 rounded border border-orange-200">
        {s.keliling_keterangan}
    </span>
)}
```

### 4f. Fix `handleKonfirmasiPDS` — tambah `fetchPresensi` + skip modal Device Bermasalah

Cari `const handleKonfirmasiPDS = async () => {`, modifikasi:

```jsx
const handleKonfirmasiPDS = async () => {
    const ket = keteranganPDS === 'Lainnya' ? keteranganPDSCustom.trim() : keteranganPDS;
    if (!ket) { ... return; }
    try {
        const formDataPDS = new FormData();
        // ... existing formData setup sama ...

        const response = await axios.post(`${API_BASE}/scan-peserta`, formDataPDS, {
            headers: { 'Content-Type': 'multipart/form-data' }
        });

        setShowPdsModal(false);
        setPdsPendingSiswa(null);
        setKeteranganPDS('');
        setKeteranganPDSCustom('');
        setPdsAttachment(null);

        fetchPesertaAll();
        fetchPresensi(); // ← TAMBAHKAN: agar scanned_by_panitia_id langsung terlihat

        // Device Bermasalah = masalah teknis, bukan pelanggaran → skip modal catatan
        if (ket === 'Device Bermasalah') {
            return; // ← TAMBAHKAN: tidak buka PelanggaranModal
        }

        const peserta = response.data.peserta || {};
        setPelanggaranModalSiswa({
            nama: peserta.nama || pdsPendingSiswa.kode_peserta,
            nomor_peserta: peserta.nomor_peserta || pdsPendingSiswa.kode_peserta,
            kelas: peserta.kelas || '-',
            presensi_id: response.data.presensi_id ?? null,
        });
    } catch (err) {
        Swal.fire('Gagal', err.response?.data?.message || 'Gagal menyimpan presensi PDS.', 'error');
    } finally {
        setPdsAttachment(null);
        isProcessingScan.current = false;
    }
};
```

---

## Fix 5 — Backend: `rekapAdmin` — fix `hadir` flag + tambah `scan_keterangan` + scanner

**File:** `backend/app/Http/Controllers/ExamReportController.php` (~baris 510)

Di dalam `$dataPeserta = $pesertas->map(function($p) use ...)`, ubah return array:

```php
$presensi = $presensis->get($p->nomor_peserta);
$ket      = $keterangans->get($p->nomor_peserta);

// Hadir HANYA jika waktu_datang terisi (keliling absent = waktu_datang null = belum hadir)
$isHadir = $presensi && $presensi->waktu_datang;

// Nama panitia scanner
$scannedByNama = null;
if ($presensi && $presensi->scanned_by_panitia_id) {
    $scannedByNama = \App\Models\Panitia::find($presensi->scanned_by_panitia_id)?->nama;
}

return [
    'nomor_peserta'           => $p->nomor_peserta,
    'nama'                    => $p->nama,
    'kelas'                   => $p->kelas,
    'ruang'                   => $p->ruang ? $p->ruang->nama_ruang : '-',
    'sesi'                    => $p->sesi,
    'hadir'                   => $isHadir ? true : false,
    'keterangan'              => $ket ? $ket->keterangan : '',
    'catatan_absen'           => $ket ? $ket->catatan : '',
    'scan_keterangan'         => $presensi ? $presensi->scan_keterangan : null,
    'scanned_by_panitia_nama' => $scannedByNama,
    'waktu_datang'            => $presensi ? $presensi->waktu_datang : null,
    'waktu_pulang'            => $presensi ? $presensi->waktu_pulang : null,
    'tanggal'                 => $presensi && $presensi->waktu_datang
        ? \Carbon\Carbon::parse($presensi->waktu_datang)->toDateString()
        : ($tanggal ? $tanggal : now()->toDateString()),
];
```

---

## Fix 6 — Frontend Rekapitulasi.jsx: tampilkan scan_keterangan + scanner

**File:** `frontend-admin/src/pages/Rekapitulasi.jsx`

Cari render baris peserta (sekitar baris 334-346 dimana ada `p.keterangan`), tambahkan:

```jsx
{/* Setelah badge keterangan existing */}
{p.scan_keterangan && (
    <span className="inline-flex items-center px-2 py-0.5 rounded-full text-[10px] font-bold bg-amber-100 text-amber-800 border border-amber-200">
        {p.scan_keterangan}
        {p.scanned_by_panitia_nama ? ` · ${p.scanned_by_panitia_nama}` : ''}
    </span>
)}
```

---

## Files yang Diubah

| File | Fix |
|------|-----|
| `backend/public/.user.ini` | BARU — Fix 1: upload limit 10M |
| `backend/app/Http/Controllers/ExamReportController.php` | Fix 2+3: `getPresensiToday` + Fix 5: `rekapAdmin` |
| `frontend/src/App.jsx` | Fix 4: state, fetchPresensi, absentInRoom, row display, handleKonfirmasiPDS |
| `frontend-admin/src/pages/Rekapitulasi.jsx` | Fix 6: scan_keterangan badge + scanner nama |

---

## Acceptance Criteria

- [ ] Upload foto 3–5MB di keliling → berhasil tersimpan
- [ ] Upload foto pelanggaran 3–5MB di PDS modal → berhasil tersimpan
- [ ] Keliling tandai Alfa → siswa muncul di list "Belum Hadir" pengawas dengan badge "Alfa"
- [ ] PDS scan siswa → pengawas langsung lihat label "Diabsen [Nama PDS] — Terlambat" (tanpa perlu tunggu 30s)
- [ ] PDS scan dengan keterangan "Device Bermasalah" → modal Catatan Pelanggaran TIDAK muncul
- [ ] Rekapitulasi: siswa yang di-PDS-kan tampil keterangan "Terlambat · [Nama PDS]"
- [ ] Rekapitulasi: siswa keliling absen tampil status "Belum Hadir" (bukan "Hadir") + keterangan "Alfa"
- [ ] Update CONTEXT.md setelah selesai
