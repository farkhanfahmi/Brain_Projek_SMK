---
tags:
  - task
  - bugfix
created: 2026-06-05
status: ready
---

# Task-005: Fix Import Jadwal Pengawas + Frontend Edit Form + Rekap Tanggal + Template Sesi

## Objective
Perbaiki 4 bug yang sudah diverifikasi dari kode aktual:
1. Import jadwal menyimpan `pengawas_id` dari ujian lama → pengawas gagal login
2. Form edit jadwal selalu reset ke pengawas pertama → menampilkan pengawas yang salah
3. Rekap: kolom tanggal siswa absen selalu tampilkan "hari ini" bukan tanggal filter
4. Template jadwal contoh sesi `'Sesi 1'` — seharusnya `'1'`

## Context
- **DB Tables:** `jadwal_ujians`, `pengawas`
- **API Endpoints:** `POST /jadwal-ujian/import`, `GET /rekap-admin`
- **Files:** 3 file — 1 backend controller, 1 backend controller, 1 frontend page

---

## Bug 1: Import Jadwal — `->first()` Tanpa Filter `ujian_id`

### Root Cause
File: `backend/app/Http/Controllers/JadwalUjianController.php`

```php
// Baris 221 — SALAH
$pengawas = Pengawas::where('niy', $niyPengawas)->first();

// Baris 229 — SALAH
$pengawasPengganti = Pengawas::where('niy', $niyPengganti)->first();
```

`->first()` tanpa `ujian_id` mengambil record pengawas paling lama (ID terkecil).
Akibatnya `jadwal_ujians.pengawas_id` menyimpan ID dari ujian yang berbeda.
Saat edit, `filteredProctors` hanya berisi pengawas ujian aktif → ID lama tidak ditemukan → form reset ke [0].

### Fix

```php
// Baris 221 — SESUDAH
$pengawas = Pengawas::where('niy', $niyPengawas)
    ->where('ujian_id', $request->ujian_id)
    ->first();
if (!$pengawas) {
    $errors[] = "Baris " . ($index + 2) . ": Pengawas dengan NIY '$niyPengawas' tidak ditemukan untuk ujian ini. Pastikan data Pengawas sudah diimport untuk ujian yang dipilih.";
    continue;
}

// Baris 229 — SESUDAH
if (!empty($niyPengganti)) {
    $pengawasPengganti = Pengawas::where('niy', $niyPengganti)
        ->where('ujian_id', $request->ujian_id)
        ->first();
    if (!$pengawasPengganti) {
        $errors[] = "Baris " . ($index + 2) . ": Pengawas pengganti dengan NIY '$niyPengganti' tidak ditemukan untuk ujian ini.";
        continue;
    }
}
```

**Penting:** Pesan error di-improve agar admin tahu harus import pengawas dulu sebelum import jadwal.

---

## Bug 2: Template Sesi Jadwal — `'Sesi 1'` Seharusnya `'1'`

### Root Cause
File: `backend/app/Http/Controllers/JadwalUjianController.php`, method `template()`

```php
// Baris 142 — SALAH
$sheet->setCellValue('B2', 'Sesi 1');
```

Database menyimpan sesi sebagai `'1'`, `'2'`, `'3'`. Jika admin mengikuti contoh dan input `'Sesi 1'`, sesi tersimpan sebagai string `'Sesi 1'` yang tidak cocok saat filtering.

### Fix
```php
// Header label — perjelas format
$sheet->setCellValue('B1', 'Sesi (isi angka: 1, 2, atau 3)');

// Contoh data — gunakan angka saja
$sheet->setCellValue('B2', '1');
```

---

## Bug 3: Frontend ExamSchedule — useEffect Reset Pengawas saat Edit

### Root Cause
File: `frontend-admin/src/pages/ExamSchedule.jsx`, baris 104–116

```jsx
// MASALAH: useEffect ini berjalan saat proctors di-load ulang
// Saat handleEdit dipanggil → ujian_id berubah → fetchDependentData dipanggil (async)
// Proctors load → useEffect ini berjalan → pengawas_id yang sudah diset benar di-reset ke [0]
useEffect(() => {
    if (formData.ujian_id && filteredProctors.length > 0) {
        const isMainProctorValid = filteredProctors.some(p => p.id == formData.pengawas_id);
        if (!isMainProctorValid) {
            setFormData(prev => ({ ...prev, pengawas_id: filteredProctors[0].id })); // ← MASALAH
        }
    }
}, [formData.ujian_id, proctors]);
```

### Fix
Tambahkan guard `editMode` di awal useEffect:

```jsx
useEffect(() => {
    // JANGAN reset pilihan pengawas saat sedang mode edit
    // Mode edit sudah meng-set pengawas_id dari data jadwal yang ada
    if (editMode) return;

    if (formData.ujian_id && filteredProctors.length > 0) {
        const isMainProctorValid = filteredProctors.some(p => p.id == formData.pengawas_id);
        const isSubProctorValid = formData.pengawas_pengganti_id === '' ||
            filteredProctors.some(p => p.id == formData.pengawas_pengganti_id);

        if (!isMainProctorValid) {
            setFormData(prev => ({ ...prev, pengawas_id: filteredProctors[0].id }));
        }
        if (!isSubProctorValid) {
            setFormData(prev => ({ ...prev, pengawas_pengganti_id: '' }));
        }
    }
}, [formData.ujian_id, proctors, editMode]); // ← tambahkan editMode ke dependency array
```

**Penting:** Tambahkan `editMode` ke dependency array agar React tidak complain (exhaustive-deps rule).

---

## Bug 4: Rekap — Kolom Tanggal Siswa Absen Selalu `now()`

### Root Cause
File: `backend/app/Http/Controllers/ExamReportController.php`, method `rekapAdmin()`

Cari baris dengan pola ini di bagian mapping `$dataPeserta`:
```php
// SALAH — baris ~501
'tanggal' => $presensi && $presensi->waktu_datang
    ? \Carbon\Carbon::parse($presensi->waktu_datang)->toDateString()
    : now()->toDateString(),  // ← hardcode "hari ini", abaikan parameter $tanggal
```

Untuk siswa yang absen (tidak ada presensi), kolom tanggal selalu menampilkan tanggal hari ini meski admin sedang melihat rekap untuk tanggal lain.

### Fix
```php
// BENAR
'tanggal' => $presensi && $presensi->waktu_datang
    ? \Carbon\Carbon::parse($presensi->waktu_datang)->toDateString()
    : ($tanggal ? $tanggal : now()->toDateString()),  // ← pakai parameter $tanggal
```

---

## Bug 5 (Nice-to-Have): Disable "Semua Tanggal" di Manual Kehadiran

### Root Cause (Code Smell)
`keyBy('kode_peserta')` di `rekapAdmin()` mode tanpa filter tanggal akan menimpa record siswa yang hadir di beberapa hari — hanya presensi TERAKHIR yang tersisa.

### Fix
Di `frontend-admin/src/pages/ManualAttendance.jsx`, hapus opsi "Semua Tanggal":

```jsx
// SEBELUM: dropdown tanggal bisa kosong (= semua tanggal)
{availableDates.length === 0 && <option value="">Semua Tanggal</option>}

// SESUDAH: wajib pilih tanggal, tidak ada opsi "Semua"
// Jika availableDates kosong, tampilkan pesan "Tidak ada tanggal tersedia"
{availableDates.length === 0 
    ? <option value="" disabled>Tidak ada tanggal tersedia</option>
    : null
}
```

Dan di `fetchRekap`, add guard:
```jsx
const fetchRekap = useCallback(async () => {
    if (!selectedUjian || !tanggal) {  // ← ini sudah ada, pastikan tetap ada
        setPesertaList([]);
        return;
    }
    ...
```

---

## Data Fix (Opsional — Lakukan Manual Jika Diperlukan)

Jadwal yang sudah terlanjur diimport dengan `pengawas_id` salah perlu diperbaiki manual.
**Jangan jalankan ini secara otomatis** — laporkan di CONTEXT.md sebagai "Masalah Ditemukan" dan biarkan user memutuskan.

Cara paling aman: gunakan fitur "Hapus Semua Jadwal" (task-004) lalu import ulang jadwal setelah bug backend ini difix. Pastikan tidak ada laporan berita acara sebelum menghapus jadwal.

---

## Files yang Diubah

| File | Perubahan |
|------|-----------|
| `backend/app/Http/Controllers/JadwalUjianController.php` | Bug 1: tambah `where('ujian_id')` di baris 221 & 229 |
| `backend/app/Http/Controllers/JadwalUjianController.php` | Bug 2: fix template sesi contoh `'Sesi 1'` → `'1'` |
| `backend/app/Http/Controllers/ExamReportController.php` | Bug 4: fix fallback tanggal siswa absen |
| `frontend-admin/src/pages/ExamSchedule.jsx` | Bug 3: tambah `if (editMode) return` di useEffect |
| `frontend-admin/src/pages/ManualAttendance.jsx` | Bug 5 (opsional): hapus opsi "Semua Tanggal" |

**Jangan sentuh:** Model, migration, route, service lain, frontend lain.

---

## Urutan Eksekusi

1. Fix Bug 1 + Bug 2 di `JadwalUjianController.php` (satu file, dua perubahan)
2. Fix Bug 4 di `ExamReportController.php`
3. Fix Bug 3 di `ExamSchedule.jsx`
4. Bug 5 di `ManualAttendance.jsx` — kerjakan hanya jika waktu cukup

---

## Acceptance Criteria

**Bug 1 (Import Jadwal):**
- [ ] Import jadwal dengan NIY pengawas yang sudah ada di ujian aktif → berhasil, `pengawas_id` merujuk ke record ujian aktif
- [ ] Import jadwal dengan NIY yang TIDAK ADA di ujian aktif → row di-skip, error message jelas: "Pastikan data Pengawas sudah diimport untuk ujian yang dipilih"
- [ ] Import jadwal dengan NIY pengganti yang tidak ada → row di-skip, error message jelas

**Bug 2 (Template Sesi):**
- [ ] Download template jadwal → kolom B header: "Sesi (isi angka: 1, 2, atau 3)", contoh data: `'1'`

**Bug 3 (Frontend Edit):**
- [ ] Klik Edit pada jadwal → form menampilkan pengawas yang BENAR (sama dengan yang di list)
- [ ] Form tidak ter-reset ke pengawas pertama setelah proctors selesai di-load
- [ ] Membuat jadwal baru (bukan edit) → useEffect masih berjalan normal (reset saat ganti ujian)

**Bug 4 (Rekap Tanggal):**
- [ ] Buka rekap dengan filter tanggal kemarin → siswa yang absen menampilkan tanggal kemarin (bukan hari ini)
- [ ] Buka rekap tanpa filter tanggal → siswa absen menampilkan null atau string kosong (bukan hari ini)

**Semua:**
- [ ] CONTEXT.md diupdate dengan hasil dan apakah ada data jadwal lama yang perlu di-reimport

## Validasi Claudian
- [x] Semua bug diverifikasi langsung dari kode aktual (bukan asumsi)
- [x] Fix Bug 3 backward-compatible: useEffect tetap berjalan untuk skenario "create new" (editMode=false)
- [x] Fix Bug 1 mengubah pesan error agar lebih informatif — admin tidak bingung kenapa row di-skip
- [x] Tidak ada perubahan schema/migration
- [x] Bug 5 bersifat opsional — tidak block task jika tidak sempat
