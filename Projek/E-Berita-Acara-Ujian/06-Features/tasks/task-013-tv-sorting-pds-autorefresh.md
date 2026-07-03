---
tags:
  - task
  - bugfix
  - frontend
  - frontend-tv
created: 2026-06-07
status: ready
---

# Task-013: TV Sorting + PDS Auto-Refresh + Pengawas Filter Fix

## Objective
3 perbaikan di 2 file: `frontend-tv/src/hooks/useAttendanceData.js` + `frontend/src/App.jsx`.
Tidak ada perubahan backend, tidak ada migration.

## Context
- Spec fitur: `06-Features/feature-006-monitoring-petugas.md`
- Jangan sentuh file lain di luar yang disebutkan

---

## Fix 1 — TV: Peserta Belum Hadir — Sesi Aktif + Sebelumnya

**File:** `frontend-tv/src/hooks/useAttendanceData.js`

**Problem:** Saat ada sesi aktif, `absentStudents` hanya menampilkan siswa dari sesi aktif saja (`sesiFilter = currentSesi`). User ingin lihat siswa belum hadir dari sesi aktif PLUS sesi-sesi sebelumnya.

**Fix:** Fetch absent students dengan `sesiFilter = null` (semua sesi), lalu filter client-side agar hanya sesi ≤ currentSesi yang tampil. Sort: sesi tertinggi (aktif) di atas, lalu alfabetis.

```js
// SEBELUM (sekitar baris 78-86)
try {
    const res = await getAttendanceStudents(activeUjianId, todayStr, '', '', sesiFilter)
    const allAbsent = (res.data || []).filter(s => s.status === 'Tidak Hadir')
    allAbsent.sort((a, b) => a.nama?.localeCompare(b.nama))
    setAbsentStudents(allAbsent)
} catch (err) {
    console.error('Absent students error:', err)
    setAbsentStudents([])
}

// SESUDAH
try {
    // Fetch semua (null = tanpa filter sesi) untuk dapat aktif + sebelumnya
    const res = await getAttendanceStudents(activeUjianId, todayStr, '', '', null)
    const currentSesiNum = sesiInfo.current ? Number(sesiInfo.current) : null

    const allAbsent = (res.data || []).filter(s => {
        if (s.status !== 'Tidak Hadir') return false
        if (!currentSesiNum) return true // tidak ada sesi aktif → tampilkan semua
        // Hanya sesi yang sudah berlangsung (≤ sesi aktif)
        const studentSesi = s.sesi ? Number(s.sesi) : null
        return studentSesi === null || studentSesi <= currentSesiNum
    })

    // Sort: sesi aktif (tertinggi) di atas, lalu sebelumnya, lalu alfabetis
    allAbsent.sort((a, b) => {
        const sA = a.sesi ? Number(a.sesi) : 0
        const sB = b.sesi ? Number(b.sesi) : 0
        if (sA !== sB) return sB - sA // sesi lebih tinggi = lebih atas
        return (a.nama || '').localeCompare(b.nama || '')
    })

    setAbsentStudents(allAbsent)
} catch (err) {
    console.error('Absent students error:', err)
    setAbsentStudents([])
}
```

---

## Fix 2 — TV: Kehadiran Per Ruang — Sort Descending (Paling Sedikit di Atas)

**File:** `frontend-tv/src/hooks/useAttendanceData.js`

**Problem:** "Kehadiran Per Kampus & Ruang" tidak di-sort. User ingin ruangan dengan kehadiran paling sedikit tampil di paling atas.

**Fix:** Sort `classResults` per kampus setelah fetch. Field yang digunakan untuk sort adalah jumlah siswa hadir.

Pertama, cek field apa yang ada di response `getAttendanceByClass`. Biasanya ada `hadir` atau `present` atau `total_hadir`. Verifikasi dengan `console.log(classResults)` jika tidak yakin.

```js
// SEBELUM (sekitar baris 74-75)
setClassData(classResults)

// SESUDAH — sort setiap kampus by least hadir ascending
// Ganti nama field 'hadir' jika di response menggunakan nama berbeda (cek console)
Object.keys(classResults).forEach(campus => {
    classResults[campus].sort((a, b) => {
        const hadirA = a.hadir ?? a.present ?? a.total_hadir ?? 0
        const hadirB = b.hadir ?? b.present ?? b.total_hadir ?? 0
        return hadirA - hadirB // ascending = paling sedikit di atas
    })
})
setClassData(classResults)
```

**Catatan:** `campusData` (Kehadiran Per Kampus summary) tidak perlu di-sort karena hanya 2 kampus (Kampus 1 dan Kampus 2).

---

## Fix 3 — Pengawas: Auto-Refresh Presensi Agar PDS Scan Muncul Otomatis

**File:** `frontend/src/App.jsx`

**Problem:** `fetchPresensi` (ambil data hadir via `/presensi-today`) hanya dipanggil saat:
- Page load
- `formData.ujian_id` berubah
- Pengawas scan peserta sendiri

Akibatnya: jika PDS scan siswa dari device lain, pengawas tidak tahu sampai manual refresh.

**Fix:** Tambahkan `setInterval` polling setiap 30 detik. Harus berhenti saat unmount.

Cari `useEffect` yang sudah ada untuk `fetchPresensi`, lalu tambahkan interval di dalamnya. Tambahkan di setelah `fetchInitData` + `fetchPresensi` sudah ada (sekitar baris 256-259):

```jsx
// SESUDAH (ganti/tambahkan setelah useEffect yang sudah ada)
useEffect(() => {
    fetchInitData().finally(() => setLoading(false));
    fetchPresensi();

    // Auto-refresh presensi setiap 30 detik agar scan PDS/Keliling muncul otomatis
    const interval = setInterval(() => {
        fetchPresensi();
    }, 30000);

    return () => clearInterval(interval);
}, [fetchInitData]); // fetchPresensi sengaja tidak di dep array agar tidak re-register tiap render
```

**Catatan:** `fetchPresensi` ada di `useCallback` dengan dependency `[formData.ujian_id]`. Polling ini akan selalu menggunakan versi terbaru karena `useCallback` referensi stabil.

---

## Files yang Diubah

| File | Aksi |
|------|------|
| `frontend-tv/src/hooks/useAttendanceData.js` | Fix 1: absent students sesi aktif+sebelumnya + Fix 2: sort per ruang |
| `frontend/src/App.jsx` | Fix 3: auto-refresh presensi setiap 30 detik |

**Tidak ada file backend, migration, atau model yang diubah.**

---

## Acceptance Criteria

**TV Monitor:**
- [ ] "Peserta Belum Hadir": jika ada sesi 1 dan sesi 2 aktif, menampilkan siswa belum hadir dari KEDUA sesi
- [ ] "Peserta Belum Hadir": siswa sesi aktif (sesi lebih tinggi) di atas, lalu sesi sebelumnya, lalu alfabetis
- [ ] "Peserta Belum Hadir": TIDAK menampilkan siswa dari sesi yang BELUM mulai (future session)
- [ ] "Kehadiran Per Kampus & Ruang": ruang dengan siswa hadir paling sedikit berada di baris pertama

**Pengawas Auto-refresh:**
- [ ] Setelah PDS scan siswa di device lain, pengawas view menampilkan siswa tersebut sebagai hadir (amber) dalam ≤ 35 detik tanpa perlu manual refresh
- [ ] Auto-refresh tidak menyebabkan halaman jump/scroll reset yang mengganggu

- [ ] Update CONTEXT.md setelah selesai
