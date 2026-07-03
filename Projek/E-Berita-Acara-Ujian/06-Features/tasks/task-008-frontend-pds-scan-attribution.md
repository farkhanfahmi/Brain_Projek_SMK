---
tags:
  - task
  - feature
  - frontend
created: 2026-06-05
status: blocked
blocked_by: task-006
---

# Task-008: Frontend — PDS Scan Modal + Atribusi di Pengawas + Toggle Admin

## Prerequisite
**task-006 harus selesai dulu.** Verifikasi:
- Login response panitia menyertakan `is_pds`
- Endpoint scan presensi return `require_keterangan: true` jika PDS tanpa keterangan
- Response daftar siswa pengawas menyertakan `scanned_by_panitia_id` + `scan_keterangan`

## Objective
Tiga perubahan frontend:
1. Halaman scan panitia: tambah popup keterangan untuk PDS/Team IT
2. Halaman pengawas: tampilkan label atribusi jika siswa diabsen oleh panitia
3. Admin panel: tambah toggle `is_pds` dan `is_keliling` di halaman Panitia

---

## Bagian 1 — Modal Keterangan untuk PDS

### Di halaman scan panitia (`frontend/src/pages/Scan.jsx` atau nama sejenis)

**Alur yang harus diubah:**

Saat ini: scan barcode → langsung kirim presensi → tampilkan hasil

Setelah perubahan:
1. Scan barcode → kirim ke backend
2. Jika response `require_keterangan: true` → **jangan tampilkan sukses dulu**, tampilkan modal
3. Modal ditampilkan dengan pilihan keterangan
4. Setelah dipilih → kirim ulang dengan `scan_keterangan` → tampilkan sukses

**Modal keterangan:**

```
┌─────────────────────────────────┐
│  📝 Tambah Keterangan           │
│  Siswa: [Nama Siswa]            │
├─────────────────────────────────┤
│  Pilih keterangan:              │
│                                 │
│  ○ Terlambat                    │
│  ○ Device Bermasalah            │
│  ○ Lainnya: [____________]      │
│                                 │
│  [Batal]        [Konfirmasi]    │
└─────────────────────────────────┘
```

**State tambahan:**
```jsx
const [showKeteranganModal, setShowKeteranganModal] = useState(false);
const [pendingSiswa, setPendingSiswa] = useState(null);    // data siswa yang menunggu konfirmasi
const [keteranganPDS, setKeteranganPDS] = useState('');
const [keteranganCustom, setKeteranganCustom] = useState('');
```

**Logic:**
```jsx
const handleScanResult = async (barcodeValue) => {
    const res = await axios.post('/api/presensi', {
        kode_peserta: barcodeValue,
        panitia_id: user.id,
    });
    
    if (res.data.require_keterangan) {
        // PDS mode: tampilkan modal sebelum konfirmasi
        setPendingSiswa(res.data.siswa);
        setShowKeteranganModal(true);
        return;
    }
    
    // Normal: langsung tampilkan sukses
    showSuccess(res.data);
};

const handleKonfirmasiPDS = async () => {
    const ket = keteranganPDS === 'Lainnya' ? keteranganCustom : keteranganPDS;
    if (!ket) { alert('Pilih keterangan dulu.'); return; }
    
    const res = await axios.post('/api/presensi', {
        kode_peserta:    pendingSiswa.kode_peserta,
        panitia_id:      user.id,
        scan_keterangan: ket,
    });
    
    setShowKeteranganModal(false);
    setPendingSiswa(null);
    setKeteranganPDS('');
    setKeteranganCustom('');
    showSuccess(res.data);
};
```

---

## Bagian 2 — Atribusi di Halaman Pengawas

### Di komponen yang menampilkan daftar siswa pengawas

Cari di `frontend/src/pages/Pengawas.jsx` atau komponen daftar siswa.

**Tampilan normal (scan oleh pengawas sendiri):**
```
✅ Ahmad Fauzi                  [latar hijau]
   07:32
```

**Tampilan jika diabsen oleh PDS/Team IT:**
```
🟡 Ahmad Fauzi                  [latar amber/orange]
   07:32
   Diabsen oleh PDS/Team IT — Terlambat
```

**Implementasi:**
```jsx
// Di mapping setiap siswa:
const isDiabsenPanitia = siswa.presensi?.scanned_by_panitia_id != null;
const scanKeterangan = siswa.presensi?.scan_keterangan;

return (
    <div className={`student-row ${isDiabsenPanitia ? 'bg-amber-50 border-amber-200' : 'bg-green-50 border-green-200'}`}>
        <span className="student-name">{siswa.nama}</span>
        <span className="waktu">{siswa.presensi?.waktu_datang}</span>
        {isDiabsenPanitia && (
            <span className="text-xs text-amber-700 mt-1">
                Diabsen oleh PDS/Team IT{scanKeterangan ? ` — ${scanKeterangan}` : ''}
            </span>
        )}
    </div>
);
```

**Catatan:** Tampilkan ini **saat refresh** — tidak perlu real-time/polling.

---

## Bagian 3 — Toggle Admin di Halaman Panitia

### Di `frontend-admin/src/pages/Panitia.jsx`

**Tambahkan dua toggle button per baris panitia**, mirip dengan toggle `can_scan` yang sudah ada.

**Tampilan per baris:**
```
[Nama]  [NIY]  [Jabatan]  [can_scan ●]  [PDS ○]  [Keliling ○]  [Edit] [Hapus]
```

**Handler:**
```jsx
const handleTogglePds = async (panitiaId) => {
    try {
        const res = await axios.patch(`/api/panitia/${panitiaId}/toggle-pds`);
        // Refresh data panitia
        fetchInitialData();
        Swal.fire('Berhasil', res.data.message, 'success');
    } catch (err) {
        Swal.fire('Gagal', err.response?.data?.message || 'Gagal toggle PDS', 'error');
    }
};

const handleToggleKeliling = async (panitiaId) => {
    try {
        const res = await axios.patch(`/api/panitia/${panitiaId}/toggle-keliling`);
        fetchInitialData();
        Swal.fire('Berhasil', res.data.message, 'success');
    } catch (err) {
        Swal.fire('Gagal', err.response?.data?.message || 'Gagal toggle Keliling', 'error');
    }
};
```

**Visual toggle:**
- `is_pds = true` → badge/pill berwarna merah/orange dengan teks "PDS ✓"
- `is_keliling = true` → badge/pill berwarna biru dengan teks "Keliling ✓"
- Keduanya false → badge abu-abu "PDS" dan "Keliling" (off)

---

## Berita Acara — Catatan Tambahan PDS

Di halaman berita acara pengawas (atau preview laporan), tampilkan `catatan_pelaksanaan_full` dari response API, **bukan** `catatan_pelaksanaan` biasa.

```jsx
// Ganti:
<p>{laporan.catatan_pelaksanaan}</p>

// Dengan:
<p>{laporan.catatan_pelaksanaan_full || laporan.catatan_pelaksanaan}</p>
```

`catatan_pelaksanaan_full` adalah computed field dari backend yang sudah include append PDS.

---

## Files yang Diubah

| File | Aksi |
|------|------|
| `frontend/src/pages/Scan.jsx` (atau nama scan panitia) | MODIFIKASI — tambah modal keterangan PDS |
| `frontend/src/pages/Pengawas.jsx` (atau komponen daftar siswa) | MODIFIKASI — tampilan atribusi |
| `frontend/src/context/AuthContext.jsx` (atau login handler) | MODIFIKASI — routing `is_pds` → scan page |
| `frontend-admin/src/pages/Panitia.jsx` | MODIFIKASI — toggle PDS + Keliling |
| Halaman berita acara/laporan pengawas | MODIFIKASI — pakai `catatan_pelaksanaan_full` |

---

## Acceptance Criteria

**PDS Modal:**
- [ ] Panitia `is_pds = true` scan siswa → muncul modal sebelum konfirmasi
- [ ] Modal punya pilihan: Terlambat, Device Bermasalah, Lainnya (+ input teks)
- [ ] "Lainnya" dipilih → input teks wajib diisi sebelum konfirmasi
- [ ] Batal di modal → presensi tidak tersimpan, bisa scan ulang
- [ ] Konfirmasi → presensi tersimpan dengan keterangan

**Atribusi Pengawas:**
- [ ] Siswa yang diabsen pengawas → tampil hijau, tidak ada label tambahan
- [ ] Siswa yang diabsen PDS → tampil amber/orange + teks "Diabsen oleh PDS/Team IT — [keterangan]"
- [ ] Tampil setelah refresh (tidak perlu real-time)

**Toggle Admin:**
- [ ] Toggle PDS aktif → badge merah muncul di baris panitia tersebut
- [ ] Toggle Keliling aktif → badge biru muncul
- [ ] Jika PDS aktif, klik Keliling → muncul pesan error dari backend (422)
- [ ] Toggle bisa dibalik (klik lagi → nonaktif)

**Berita Acara:**
- [ ] Jika ada siswa yang diabsen PDS → catatan_pelaksanaan di preview/laporan punya append format ` | Nama Keterangan`
- [ ] Jika tidak ada PDS scan → catatan tampil normal tanpa append

- [ ] Update CONTEXT.md setelah selesai



eddit
