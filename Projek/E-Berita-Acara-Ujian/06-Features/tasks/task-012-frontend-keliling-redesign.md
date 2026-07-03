---
tags:
  - task
  - feature
  - frontend
created: 2026-06-07
status: blocked
blocked_by: task-011
---

# Task-012: Frontend — Rewrite Keliling.jsx (One-Page) + Photo Upload UI

## Prerequisite
**task-011 harus selesai dulu.** Verifikasi endpoint berikut berjalan:
- `GET /keliling/overview?ujian_id=X` → return array `data` berisi ruang + siswa
- `POST /keliling/keterangan` dengan multipart/form-data + file opsional

## Objective
Rewrite `Keliling.jsx` menjadi satu halaman tanpa navigasi antar screen. Tambah UI upload foto di keliling dan di modal PDS (`App.jsx`).

---

## 1. Rewrite `Keliling.jsx` — One-Page Grouped View

### Hapus arsitektur 2-screen (screen 1: pilih ruang, screen 2: detail)

Ganti dengan **satu halaman scrollable** yang menampilkan semua ruang sekaligus, dikelompokkan.

### State

```jsx
const [overview, setOverview] = useState([]);      // array ruang dari API
const [loading, setLoading]   = useState(true);
const [hasActiveSession, setHasActiveSession] = useState(false);

// Per siswa: { [pesertaId]: keterangan_string }
const [keteranganMap, setKeteranganMap]   = useState({});
// Per siswa untuk "Lainnya": { [pesertaId]: teks_custom }
const [keteranganCustom, setKeteranganCustom] = useState({});
// Per siswa: file yang dipilih { [pesertaId]: File object }
const [attachmentMap, setAttachmentMap]   = useState({});
// Per siswa: loading state { [pesertaId]: boolean }
const [saving, setSaving] = useState({});
```

### Fetch Data

```jsx
const fetchOverview = async () => {
    setLoading(true);
    try {
        const res = await axios.get(`${API_BASE}/keliling/overview`, {
            params: { ujian_id: ujianId }
        });
        setOverview(res.data.data || []);
        setHasActiveSession(res.data.has_active_session);
    } catch (err) {
        Swal.fire('Error', 'Gagal memuat data.', 'error');
    } finally {
        setLoading(false);
    }
};

useEffect(() => {
    if (ujianId) fetchOverview();
}, [ujianId]);
```

### Simpan Keterangan (dengan foto opsional)

```jsx
const simpanKeterangan = async (jadwalId, pesertaId, nomorPeserta) => {
    const ket = keteranganMap[pesertaId];
    if (!ket) { Swal.fire('Pilih Keterangan', '', 'warning'); return; }

    const finalKet = ket === 'Lainnya'
        ? (keteranganCustom[pesertaId] || '').trim()
        : ket;
    if (!finalKet) { Swal.fire('Isi Keterangan', 'Tuliskan keterangan untuk "Lainnya".', 'warning'); return; }

    // Validasi foto WAJIB untuk keterangan tertentu
    const KETERANGAN_WAJIB_FOTO = ['Sakit', 'Izin', 'PKL', 'Dispen'];
    if (KETERANGAN_WAJIB_FOTO.includes(finalKet) && !attachmentMap[pesertaId]) {
        Swal.fire('Foto Wajib', `Keterangan "${finalKet}" wajib disertai foto surat bukti.`, 'warning');
        return;
    }

    setSaving(prev => ({ ...prev, [pesertaId]: true }));

    try {
        const formData = new FormData();
        formData.append('jadwal_id', jadwalId);
        formData.append('nomor_peserta', nomorPeserta);
        formData.append('panitia_id', user.id);
        formData.append('keterangan', finalKet);

        const file = attachmentMap[pesertaId];
        if (file) formData.append('attachment', file);

        await axios.post(`${API_BASE}/keliling/keterangan`, formData, {
            headers: { 'Content-Type': 'multipart/form-data' }
        });

        // Hapus siswa dari list setelah berhasil
        setOverview(prev => prev.map(ruang => {
            if (ruang.jadwal_id !== jadwalId) return ruang;
            const newBelum = ruang.siswa_belum_scan.filter(s => s.id !== pesertaId);
            // Jika tidak ada siswa belum, ruang masih tampil karena mungkin ada kondisi lain
            return { ...ruang, siswa_belum_scan: newBelum, total_belum: newBelum.length };
        }).filter(ruang =>
            ruang.total_belum > 0 ||
            ruang.pengawas_pengganti !== null ||
            ruang.status_berita_acara === 'Belum diisi'
        ));

        // Bersihkan state siswa ini
        setKeteranganMap(p => { const n = { ...p }; delete n[pesertaId]; return n; });
        setAttachmentMap(p => { const n = { ...p }; delete n[pesertaId]; return n; });
    } catch (err) {
        Swal.fire('Gagal', err.response?.data?.message || 'Gagal menyimpan.', 'error');
    } finally {
        setSaving(prev => ({ ...prev, [pesertaId]: false }));
    }
};
```

### Tampilan

```jsx
return (
    <div className="min-h-screen bg-slate-50">
        {/* Header tetap */}
        <div className="sticky top-0 z-10 bg-white border-b border-slate-100 px-4 py-3 flex items-center justify-between">
            <div>
                <h1 className="font-black text-slate-900 text-base">Monitoring Keliling</h1>
                <p className="text-xs text-slate-400">
                    {hasActiveSession ? 'Sesi aktif + sebelumnya' : 'Semua sesi hari ini'}
                </p>
            </div>
            <div className="flex gap-2">
                <button onClick={fetchOverview} className="p-2 rounded-xl hover:bg-slate-100">
                    {/* ikon refresh */}
                </button>
                <button onClick={onLogout} className="p-2 rounded-xl hover:bg-red-50 text-red-500">
                    {/* ikon logout */}
                </button>
            </div>
        </div>

        {/* Body */}
        <div className="p-4 space-y-4 max-w-lg mx-auto pb-20">
            {loading && <p className="text-center text-slate-400 py-8">Memuat data...</p>}

            {!loading && overview.length === 0 && (
                <div className="text-center py-12">
                    <p className="text-emerald-600 font-bold text-lg">✅ Semua ruang sudah beres!</p>
                    <p className="text-slate-400 text-sm mt-1">Tidak ada ruang yang perlu perhatian.</p>
                </div>
            )}

            {overview.map(ruang => (
                <RuangCard
                    key={ruang.jadwal_id}
                    ruang={ruang}
                    keteranganMap={keteranganMap}
                    setKeteranganMap={setKeteranganMap}
                    keteranganCustom={keteranganCustom}
                    setKeteranganCustom={setKeteranganCustom}
                    attachmentMap={attachmentMap}
                    setAttachmentMap={setAttachmentMap}
                    saving={saving}
                    onSimpan={simpanKeterangan}
                />
            ))}
        </div>
    </div>
);
```

### Komponen `RuangCard`

```jsx
const RuangCard = ({ ruang, keteranganMap, setKeteranganMap, keteranganCustom, setKeteranganCustom,
                     attachmentMap, setAttachmentMap, saving, onSimpan }) => {

    const KETERANGAN_OPTIONS = ['Alfa', 'Sakit', 'Izin', 'PKL', 'Dispen', 'Lainnya'];

    return (
        <div className="bg-white rounded-2xl border border-slate-100 shadow-sm overflow-hidden">
            {/* Header ruang */}
            <div className="p-4 border-b border-slate-100">
                <div className="flex justify-between items-start">
                    <div>
                        <h2 className="font-black text-slate-900">{ruang.ruang} — Sesi {ruang.sesi}</h2>
                        <p className="text-xs text-slate-400">{ruang.nama_mapel}</p>
                    </div>
                    {ruang.total_belum > 0 && (
                        <span className="text-xs font-bold bg-red-100 text-red-700 px-2 py-1 rounded-full">
                            {ruang.total_belum} belum
                        </span>
                    )}
                </div>

                <div className="mt-2 space-y-1 text-sm">
                    <p className="text-slate-600">
                        <span className="text-slate-400">Pengawas:</span> {ruang.pengawas || '-'}
                    </p>
                    {ruang.pengawas_pengganti && (
                        <p className="text-amber-700 font-medium text-xs bg-amber-50 px-2 py-1 rounded-lg">
                            ⚠️ Pengganti: {ruang.pengawas_pengganti}
                        </p>
                    )}
                    <p className={`text-xs font-bold ${ruang.status_berita_acara === 'Sudah disubmit' ? 'text-emerald-600' : 'text-red-600'}`}>
                        BA: {ruang.status_berita_acara === 'Sudah disubmit' ? '✅ Sudah disubmit' : '⚠️ Belum diisi'}
                    </p>
                </div>
            </div>

            {/* Daftar siswa belum scan */}
            <div className="divide-y divide-slate-50">
                {ruang.siswa_belum_scan.length === 0 ? (
                    <p className="text-xs text-slate-400 text-center py-3">
                        ✅ Semua siswa hadir
                    </p>
                ) : (
                    ruang.siswa_belum_scan.map(siswa => (
                        <SiswaRow
                            key={siswa.id}
                            siswa={siswa}
                            jadwalId={ruang.jadwal_id}
                            keterangan={keteranganMap[siswa.id] || ''}
                            setKeterangan={(val) => setKeteranganMap(p => ({ ...p, [siswa.id]: val }))}
                            keteranganCustom={keteranganCustom[siswa.id] || ''}
                            setKeteranganCustom={(val) => setKeteranganCustom(p => ({ ...p, [siswa.id]: val }))}
                            attachment={attachmentMap[siswa.id] || null}
                            setAttachment={(file) => setAttachmentMap(p => ({ ...p, [siswa.id]: file }))}
                            isSaving={!!saving[siswa.id]}
                            onSimpan={() => onSimpan(ruang.jadwal_id, siswa.id, siswa.nomor_peserta)}
                            options={KETERANGAN_OPTIONS}
                        />
                    ))
                )}
            </div>
        </div>
    );
};
```

### Komponen `SiswaRow`

```jsx
const SiswaRow = ({ siswa, keterangan, setKeterangan, keteranganCustom, setKeteranganCustom,
                    attachment, setAttachment, isSaving, onSimpan, options }) => (
    <div className="p-3 space-y-2">
        {/* Nama + nomor */}
        <div>
            <p className="font-bold text-slate-900 text-sm">{siswa.nama}</p>
            <p className="text-xs text-slate-400 font-mono">{siswa.nomor_peserta} · {siswa.kelas}</p>
        </div>

        {/* Dropdown keterangan */}
        <select
            value={keterangan}
            onChange={e => setKeterangan(e.target.value)}
            className="w-full text-sm rounded-xl border-slate-200 py-2 px-3"
        >
            <option value="">-- Pilih Keterangan --</option>
            {options.map(opt => <option key={opt} value={opt}>{opt}</option>)}
        </select>

        {/* Input teks untuk "Lainnya" */}
        {keterangan === 'Lainnya' && (
            <input
                type="text"
                placeholder="Tuliskan keterangan..."
                value={keteranganCustom}
                onChange={e => setKeteranganCustom(e.target.value)}
                className="w-full text-sm rounded-xl border-slate-200 py-2 px-3"
            />
        )}

        {/* Upload foto (opsional) */}
        <div className="flex items-center gap-2">
            <label className="flex-1 flex items-center gap-2 px-3 py-2 rounded-xl border border-dashed border-slate-300 cursor-pointer hover:border-indigo-400 text-xs text-slate-400">
                <input
                    type="file"
                    accept="image/*,application/pdf"
                    capture="environment"
                    className="hidden"
                    onChange={e => setAttachment(e.target.files?.[0] || null)}
                />
                {attachment
                    ? <span className="text-indigo-600 font-medium truncate">📎 {attachment.name}</span>
                    : <span>📷 Upload foto/surat (opsional)</span>
                }
            </label>
            {attachment && (
                <button onClick={() => setAttachment(null)} className="text-red-400 text-xs px-2">✕</button>
            )}
        </div>

        {/* Tombol simpan */}
        <button
            onClick={onSimpan}
            disabled={!keterangan || isSaving}
            className="w-full py-2 rounded-xl text-sm font-bold bg-indigo-600 text-white disabled:opacity-40"
        >
            {isSaving ? 'Menyimpan...' : 'Simpan'}
        </button>
    </div>
);
```

---

## 2. Modifikasi Modal PDS di `App.jsx` — Tambah Upload Foto

Di modal PDS yang sudah ada (sekitar baris 1078), tambahkan:

### State tambahan
```jsx
const [pdsAttachment, setPdsAttachment] = useState(null);
```

### Reset saat modal ditutup/selesai
```jsx
// Saat menutup modal atau selesai konfirmasi, tambahkan:
setPdsAttachment(null);
```

### UI upload di dalam modal (setelah pilihan keterangan, sebelum tombol Konfirmasi)

```jsx
{/* Tambahkan setelah section pilihan keterangan */}
<div>
    <p className="text-sm font-bold text-slate-700 mb-2">Foto Bukti (Opsional)</p>
    <label className="flex items-center gap-2 px-3 py-2 rounded-xl border border-dashed border-slate-300 cursor-pointer hover:border-indigo-400 text-sm text-slate-400">
        <input
            type="file"
            accept="image/*,application/pdf"
            capture="environment"
            className="hidden"
            onChange={e => setPdsAttachment(e.target.files?.[0] || null)}
        />
        {pdsAttachment
            ? <span className="text-indigo-600 font-medium">📎 {pdsAttachment.name}</span>
            : <span>📷 Upload foto/surat (opsional)</span>
        }
    </label>
    {pdsAttachment && (
        <button onClick={() => setPdsAttachment(null)} className="text-xs text-red-400 mt-1">
            Hapus foto
        </button>
    )}
</div>
```

### Kirim FormData di `handleKonfirmasiPDS`

```jsx
const handleKonfirmasiPDS = async () => {
    const ket = keteranganPDS === 'Lainnya' ? keteranganPDSCustom.trim() : keteranganPDS;
    if (!ket) { alert('Pilih keterangan dulu.'); return; }

    const formData = new FormData();
    formData.append('kode_peserta', pdsPendingSiswa.kode_peserta);
    formData.append('panitia_id', user.id);
    formData.append('scan_keterangan', ket);
    if (pdsAttachment) formData.append('attachment', pdsAttachment);

    try {
        const res = await axios.post(`${API_BASE}/presensi`, formData, {
            headers: { 'Content-Type': 'multipart/form-data' }
        });
        // ... handle sukses sama seperti sebelumnya ...
    } catch (err) {
        // ... error handling ...
    } finally {
        setShowPdsModal(false);
        setPdsPendingSiswa(null);
        setKeteranganPDS('');
        setKeteranganPDSCustom('');
        setPdsAttachment(null);
    }
};
```

---

## Keterangan Baru yang Ditambahkan

Dropdown keterangan keliling sekarang ada 6 pilihan (sebelumnya 4):
- Alfa
- Sakit
- Izin
- **PKL** ← baru
- **Dispen** ← baru
- Lainnya (input teks bebas)

---

## Files yang Diubah

| File | Aksi |
|------|------|
| `frontend/src/Keliling.jsx` | REWRITE total — arsitektur 2-screen → 1-screen grouped |
| `frontend/src/App.jsx` | MODIFIKASI — tambah `pdsAttachment` state + UI upload di modal PDS + kirim FormData |

**Jangan sentuh:** Backend, Keliling logic di luar UI, route, model

---

## Acceptance Criteria

**Halaman Keliling:**
- [ ] Satu halaman, semua ruang yang perlu perhatian tampil tanpa navigasi
- [ ] Ruang yang semua hadir + pengawas asli + BA submit → TIDAK tampil
- [ ] Ruang dengan pengganti → tampil meski semua hadir dan BA submit
- [ ] Ruang dengan BA belum diisi → tampil meski semua hadir
- [ ] Header setiap ruang: nama ruang, sesi, pengawas, pengganti (jika ada), status BA
- [ ] Siswa belum scan: nama + nomor + kelas + dropdown + foto opsional + tombol simpan
- [ ] Dropdown punya opsi: Alfa, Sakit, Izin, PKL, Dispen, Lainnya
- [ ] "Lainnya" → muncul input teks wajib
- [ ] Upload foto: bisa pilih dari galeri atau kamera, tampil nama file, bisa dihapus
- [ ] Simpan tanpa foto → berhasil
- [ ] Simpan dengan foto → berhasil, attachment_url ada di response
- [ ] Setelah simpan → siswa hilang dari list; ruang hilang jika tidak ada lagi kondisi yang membuatnya tampil
- [ ] Refresh button → reload semua data
- [ ] Jika tidak ada ruang yang perlu perhatian → tampil pesan "✅ Semua ruang sudah beres!"
- [ ] Scroll ke bawah untuk lihat ruang berikutnya (tidak ada tab/navigasi)

**Modal PDS:**
- [ ] Setelah pilih keterangan, ada opsi upload foto opsional
- [ ] Konfirmasi tanpa foto → berhasil
- [ ] Konfirmasi dengan foto → berhasil, presensi punya attachment_path

- [ ] Update CONTEXT.md setelah selesai
