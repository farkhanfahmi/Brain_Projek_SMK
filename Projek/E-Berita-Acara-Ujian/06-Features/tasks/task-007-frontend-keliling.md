---
tags:
  - task
  - feature
  - frontend
created: 2026-06-05
status: blocked
blocked_by: task-006
---

# Task-007: Frontend — Halaman Petugas Keliling

## Prerequisite
**task-006 harus selesai dulu.** Verifikasi endpoint berikut berjalan sebelum mulai:
- `GET /keliling/jadwal-aktif?ujian_id=X` → return jadwal aktif
- `GET /keliling/siswa-belum-scan?jadwal_id=X` → return siswa + info ruang
- `POST /keliling/keterangan` → simpan keterangan

## Objective
Buat halaman baru di `frontend/` (app pengawas/panitia) untuk petugas keliling. Login dengan NIY scan → otomatis diarahkan ke halaman ini (bukan halaman pengawas).

---

## Routing — Modifikasi Login Flow

Di file routing/auth frontend (cari di `AuthContext.jsx` atau `App.jsx`), tambahkan kondisi setelah login berhasil:

```jsx
const handleLoginSuccess = (userData) => {
    if (userData.is_keliling) {
        navigate('/keliling');
    } else if (userData.is_pds || userData.can_scan) {
        navigate('/scan'); // existing panitia/scan page
    } else {
        navigate('/pengawas'); // existing pengawas page
    }
};
```

Tambahkan route baru di `App.jsx`:
```jsx
<Route path="/keliling" element={<KelilingPage />} />
```

---

## Komponen: `Keliling.jsx`

Buat file baru `frontend/src/pages/Keliling.jsx` (sesuaikan path dengan struktur folder yang sudah ada).

### State yang dibutuhkan
```jsx
const [ujianId, setUjianId] = useState(null);           // dari login context
const [jadwalAktif, setJadwalAktif] = useState([]);      // list jadwal aktif
const [selectedJadwal, setSelectedJadwal] = useState(null); // jadwal yang dipilih
const [jadwalDetail, setJadwalDetail] = useState(null);  // info jadwal + pengawas + BA
const [siswaBelumScan, setSiswaBelumScan] = useState([]); // list siswa
const [keteranganMap, setKeteranganMap] = useState({});  // { peserta_id: keterangan }
const [loading, setLoading] = useState(false);
const [saving, setSaving] = useState({});               // { peserta_id: boolean }
```

### Tampilan — Screen 1: Pilih Ruang

```
┌─────────────────────────────────────┐
│  🔍 Monitoring Keliling             │
│  Sesi aktif: [Nama Ujian]           │
├─────────────────────────────────────┤
│  Pilih Ruangan:                     │
│                                     │
│  ┌──────────────────────────────┐   │
│  │ R.01 — Sesi 1 (07:30-09:30) │   │
│  │ Mapel: Bahasa Indonesia      │   │
│  └──────────────────────────────┘   │
│  ┌──────────────────────────────┐   │
│  │ R.02 — Sesi 1               │   │
│  └──────────────────────────────┘   │
│  ...                                │
│                                     │
│  [Tidak ada sesi aktif saat ini]    │
│  (tampil jika jadwalAktif kosong)   │
└─────────────────────────────────────┘
```

### Tampilan — Screen 2: Detail Ruang

```
┌─────────────────────────────────────┐
│  ← Kembali   R.01 — Sesi 1         │
├─────────────────────────────────────┤
│  📋 INFO RUANG                      │
│  Mapel: Bahasa Indonesia            │
│  Jam: 07:30 – 09:30                 │
│  Pengawas: [Nama Pengawas]          │
│  Pengganti: [Nama] (jika ada)       │
│  Berita Acara: ✅ Sudah disubmit    │
│               atau ⚠️ Belum diisi   │
├─────────────────────────────────────┤
│  👥 SISWA BELUM HADIR (5 siswa)     │
│  [Refresh]                          │
│                                     │
│  Ahmad Fauzi (001)                  │
│  [Alfa ▼]  [Simpan]                 │
│                                     │
│  Budi Santoso (002)                 │
│  [Sakit ▼] [Simpan]                 │
│  ...                                │
└─────────────────────────────────────┘
```

### Dropdown keterangan per siswa

Pilihan:
- **Alfa** (tidak hadir tanpa keterangan)
- **Sakit**
- **Izin**
- **Lainnya** → tampilkan input teks bebas

### Logic penyimpanan

```jsx
const simpanKeterangan = async (pesertaId) => {
    const ket = keteranganMap[pesertaId];
    if (!ket) return;
    
    setSaving(prev => ({ ...prev, [pesertaId]: true }));
    try {
        await axios.post('/api/keliling/keterangan', {
            jadwal_id:        selectedJadwal.jadwal_id,
            peserta_ujian_id: pesertaId,
            panitia_id:       user.id,  // dari login context
            keterangan:       ket,
        });
        // Hapus dari list setelah berhasil
        setSiswaBelumScan(prev => prev.filter(s => s.id !== pesertaId));
    } catch (err) {
        alert(err.response?.data?.message || 'Gagal menyimpan.');
    } finally {
        setSaving(prev => ({ ...prev, [pesertaId]: false }));
    }
};
```

---

## UX Notes

- **Tombol kembali** dari detail ruang ke list ruang (tanpa reload halaman)
- **Refresh button** di detail ruang untuk reload data siswa (bukan auto-polling)
- Jika siswa berhasil disimpan → hilang dari list langsung (optimistic removal)
- Jika semua siswa sudah punya keterangan → tampilkan "✅ Semua siswa sudah tercatat"
- Tampilan konsisten dengan gaya existing `frontend/` (pakai komponen/CSS yang sudah ada)

---

## Files yang Dibuat / Diubah

| File | Aksi |
|------|------|
| `frontend/src/pages/Keliling.jsx` | BARU |
| `frontend/src/App.jsx` (atau file routing) | MODIFIKASI — tambah route `/keliling` |
| `frontend/src/context/AuthContext.jsx` (atau login handler) | MODIFIKASI — routing berdasarkan `is_keliling` |

---

## Acceptance Criteria

- [ ] Panitia dengan `is_keliling = true` login via NIY → otomatis masuk halaman `/keliling`
- [ ] Panitia non-keliling → tidak bisa akses `/keliling` (redirect ke halaman mereka)
- [ ] List ruang menampilkan hanya ruang dengan sesi aktif saat ini (bukan semua ruang)
- [ ] Jika tidak ada sesi aktif → tampil pesan "Tidak ada sesi aktif saat ini"
- [ ] Klik ruang → tampil info pengawas + status berita acara + list siswa belum scan
- [ ] Dropdown keterangan "Lainnya" → muncul input teks
- [ ] Simpan keterangan → siswa hilang dari list
- [ ] Siswa yang sudah scan (apapun yang scan) tidak muncul di list
- [ ] Update CONTEXT.md setelah selesai
