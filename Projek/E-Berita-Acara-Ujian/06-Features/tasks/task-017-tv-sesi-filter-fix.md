---
tags:
  - task
  - bugfix
  - backend
  - frontend-tv
created: 2026-06-08
status: ready
---

# Task-017: Fix TV Dashboard — `current_sesi` + Filter Sesi 3 Panel

## Prerequisite
Task ini HARUS selesai sebelum task-018 (performance) karena fix sesi adalah prasyarat
agar filter benar-benar efektif.

## Root Cause
`DashboardService::getAttendanceStats()` tidak pernah mengembalikan field:
- `current_sesi` — sesi yang sedang berjalan (NOW between mulai_ujian & ujian_berakhir)
- `is_last_sesi` — apakah ini sesi terakhir hari ini
- `all_sesi` — semua sesi hari ini

Akibatnya di `useAttendanceData.js`: `sesiInfo.current` selalu `null` → semua filter sesi
tidak berjalan → semua panel TV menampilkan akumulasi semua sesi (bukan sesi aktif).

---

## Fix 1 — Backend: Tambah `current_sesi` ke `getAttendanceStats`

**File:** `backend/app/Services/DashboardService.php` — method `getAttendanceStats` (atau sejenisnya)

Cari method yang dipanggil oleh route `GET /dashboard/attendance-stats` dan tambahkan logic:

```php
// Tentukan sesi yang sedang aktif (NOW antara mulai_ujian dan ujian_berakhir, hari ini)
$now   = now();
$today = today();

$aktiveSesiQuery = \App\Models\JadwalUjian::where('ujian_id', $ujianId)
    ->whereDate('mulai_ujian', $today)
    ->where('mulai_ujian', '<=', $now)
    ->where('ujian_berakhir', '>=', $now)
    ->orderBy('mulai_ujian')
    ->first();

$currentSesi = $aktiveSesiQuery?->sesi ?? null;

// Semua sesi hari ini (sudah mulai)
$allSesi = \App\Models\JadwalUjian::where('ujian_id', $ujianId)
    ->whereDate('mulai_ujian', $today)
    ->where('mulai_ujian', '<=', $now)
    ->distinct()
    ->orderBy('sesi')
    ->pluck('sesi')
    ->filter()
    ->values()
    ->toArray();

// Apakah semua sesi sudah selesai (is_last_sesi)?
$adaSesiBerikutnya = \App\Models\JadwalUjian::where('ujian_id', $ujianId)
    ->whereDate('mulai_ujian', $today)
    ->where('mulai_ujian', '>', $now)
    ->exists();
$isLastSesi = !$adaSesiBerikutnya && !empty($allSesi);
```

Tambahkan ke array return:
```php
'current_sesi'  => $currentSesi,
'is_last_sesi'  => $isLastSesi,
'all_sesi'      => $allSesi,
```

**Catatan:** Jika method sudah return banyak field, cukup tambahkan 3 field ini ke array
yang sudah ada. Jangan refactor seluruh method.

---

## Fix 2 — Backend: `getAttendanceByCampus` filter attendance by sesi aktif

**File:** `backend/app/Services/DashboardService.php` — method `getAttendanceByCampus`

Saat ini method tidak menerima parameter `$sesi`. Tambahkan parameter dan terapkan ke
query counting kehadiran peserta:

```php
// SEBELUM signature:
public function getAttendanceByCampus(?string $ujianId, ?string $date, ?string $sesi = null): array

// Di dalam method, pada query yang menghitung siswa hadir:
// Cari query presensi / hadir dan tambahkan filter sesi jika ada.
// Contoh (sesuaikan dengan kode existing):
if ($sesi) {
    $presensisQuery->whereHas('pesertaUjian', fn($q) => $q->where('sesi', $sesi));
    // ATAU jika pakai join langsung:
    // ->where('peserta_ujians.sesi', $sesi)
}
```

**Juga update route DashboardController::attendanceByCampus** untuk terima & pass parameter `sesi`.

---

## Fix 3 — Backend: `getAttendanceByClass` filter attendance by sesi aktif

**File:** `backend/app/Services/DashboardService.php` — method `getAttendanceByClass`

Saat ini `$sesi` diterima tapi hanya dipakai untuk filter daftar pengawas, BUKAN untuk
counting kehadiran. Terapkan juga ke query attendance count:

```php
// Cari $attendanceQuery (atau query counting hadir per ruang) dan tambahkan:
if ($sesi) {
    // Filter presensi hanya siswa yang terdaftar di sesi ini
    $attendanceQuery->whereHas('pesertaUjian', fn($q) => $q->where('sesi', $sesi));
    // ATAU alternatif via jadwal: hanya presensi hari itu + sesi itu
}
```

---

## Fix 4 — Backend: `getAttendanceByKelas` filter by sesi aktif + sort terendah di atas

**File:** `backend/app/Services/DashboardService.php` — method `getAttendanceByKelas`

Dua perubahan:

**4a. Filter kelas hanya yang sesinya aktif:**
```php
// Jika ada $sesi, filter peserta hanya dari sesi itu
if ($sesi) {
    $query->where('sesi', $sesi); // atau whereHas sesuai struktur
}
```

**4b. Ubah sort dari tertinggi ke terendah:**
```php
// SEBELUM (baris ~266):
usort($results, fn($a, $b) => $b['percentage'] <=> $a['percentage']); // tertinggi dulu

// SESUDAH:
usort($results, fn($a, $b) => $a['percentage'] <=> $b['percentage']); // TERENDAH dulu
```

---

## Fix 5 — Frontend `useAttendanceData.js`: gunakan `current_sesi` yang sekarang valid

**File:** `frontend-tv/src/hooks/useAttendanceData.js`

Setelah Fix 1 backend bekerja, `sesiInfo.current` tidak lagi null. Update logika:

### 5a. Absent students — tampilkan sesi aktif + sebelumnya (bukan hanya sesi aktif)

```js
// Ubah dari (hanya sesi aktif — hasil diskusi Claude Code sebelumnya):
return studentSesi === currentSesiNum

// Menjadi (aktif + sebelumnya):
return studentSesi === null || studentSesi <= currentSesiNum
```

Juga kembalikan fetch tanpa filter sesi (agar dapat semua sesi untuk difilter di client):
```js
// Ganti:
const res = await getAttendanceStudents(activeUjianId, todayStr, '', '', currentSesiNum || null)
// Menjadi:
const res = await getAttendanceStudents(activeUjianId, todayStr, '', '', null) // ambil semua, filter di client
```

### 5b. Campus & class data — gunakan `currentSesiNum` sebagai filter

```js
// Ubah dari:
const sesiFilter = sesiInfo.isLast ? null : sesiInfo.current

// Menjadi (untuk kampus & ruang, HANYA sesi aktif):
const sesiFilterKampus = sesiInfo.current // null jika tidak ada sesi aktif (semua ditampilkan)
```

Pass ke API calls:
```js
getAttendanceByCampus(activeUjianId, todayStr, sesiFilterKampus),  // pakai sesi aktif
getAttendanceByClass(activeUjianId, todayStr, campus, sesiFilterKampus),
getAttendanceByKelas(activeUjianId, todayStr, sesiFilterKampus),
```

---

## Files yang Diubah

| File | Fix |
|------|-----|
| `backend/app/Services/DashboardService.php` | Fix 1: `current_sesi` di stats + Fix 2: campus filter + Fix 3: class filter + Fix 4: kelas filter & sort |
| `backend/app/Http/Controllers/DashboardController.php` | Pass parameter `sesi` ke `getAttendanceByCampus` |
| `frontend-tv/src/hooks/useAttendanceData.js` | Fix 5: pakai `current_sesi` yang valid, filter ≤ currentSesi |

Rebuild frontend-tv setelah selesai.

---

## Acceptance Criteria

- [ ] `GET /dashboard/attendance-stats` response berisi `current_sesi`, `is_last_sesi`, `all_sesi`
- [ ] Saat sesi 1 aktif: `current_sesi = 1`; saat sesi 2 aktif: `current_sesi = 2`
- [ ] **Peserta Belum Hadir**: menampilkan siswa dari sesi 1 + 2 jika sesi 2 sedang aktif
- [ ] **Peserta Belum Hadir**: jika hanya sesi 1 aktif, hanya tampilkan siswa sesi 1
- [ ] **Kehadiran Per Kampus & Ruang**: hanya menghitung kehadiran sesi yang aktif sekarang
- [ ] **Kehadiran Per Kelas**: hanya kelas sesi aktif, diurut persentase TERENDAH di atas
- [ ] Saat tidak ada sesi aktif (selesai semua): semua panel tampilkan akumulasi
- [ ] Update CONTEXT.md setelah selesai
