---
tags:
  - task
  - bugfix
created: 2026-06-04
status: ready
---

# Task-003: Fix Pivot Generation saat Import + Fix Template Sesi

## Objective
Perbaiki 2 bug yang ditemukan di alur import peserta:
1. Pivot `jadwal_peserta` di-assign ke SEMUA jadwal di ruang+sesi, bukan hanya jadwal di hari yang sesuai `waktu_ujian` peserta
2. Template download menampilkan contoh sesi `'Sesi 1'` tapi database menyimpan `'1'` — menyesatkan admin

## Context
- **DB:** `peserta_ujians.waktu_ujian`, `jadwal_peserta`, `jadwal_ujians`
- **Files:** `backend/app/Http/Controllers/PesertaUjianController.php`
- **Metode yang diubah:** `importCsv()`, `store()`, `update()`, `downloadTemplate()`

## Root Cause

### Bug 1 — Pivot assigned ke semua jadwal ruang+sesi
```php
// SEKARANG — di importCsv(), store(), update():
$matchingScheduleQuery = JadwalUjian::where('ujian_id', $request->ujian_id)
    ->where('ruang_id', $ruangId);
if (!empty($sesi)) {
    $matchingScheduleQuery->where('sesi', $sesi);
}
// TIDAK ADA filter tanggal → siswa masuk ke 5 hari sekaligus
$peserta->jadwalUjians()->sync($matchingScheduleIds);
```

### Bug 2 — Template sesi mismatch
```php
// downloadTemplate() saat ini:
$sheet->setCellValue('F2', 'Sesi 1');  // ← SALAH: database simpan '1' bukan 'Sesi 1'
```

---

## Spec Fix

### Fix 1: Method `importCsv()` — tambah filter tanggal dari waktu_ujian

Lokasi: `PesertaUjianController.php`, di dalam loop foreach, setelah `$peserta = updateOrCreate(...)`.

**Ganti bagian:**
```php
// SEBELUM
$matchingScheduleQuery = JadwalUjian::where('ujian_id', $request->ujian_id)
    ->where('ruang_id', $ruangId);
if (!empty($sesi)) {
    $matchingScheduleQuery->where('sesi', $sesi);
}
$matchingScheduleIds = $matchingScheduleQuery->pluck('id')->toArray();
$peserta->jadwalUjians()->sync($matchingScheduleIds);
```

**Dengan:**
```php
// SESUDAH
$matchingScheduleQuery = JadwalUjian::where('ujian_id', $request->ujian_id)
    ->where('ruang_id', $ruangId);

if (!empty($sesi)) {
    $matchingScheduleQuery->where('sesi', $sesi);
}

// KUNCI FIX: Jika peserta punya waktu_ujian (ujian praktek individual),
// hanya assign ke jadwal di tanggal yang sesuai.
// Jika tidak punya waktu_ujian (ujian teori), assign ke semua jadwal di ruang+sesi (behavior lama).
if (!empty($waktuUjian)) {
    $matchingScheduleQuery->whereDate('mulai_ujian', date('Y-m-d', strtotime($waktuUjian)));
}

$matchingScheduleIds = $matchingScheduleQuery->pluck('id')->toArray();
$peserta->jadwalUjians()->sync($matchingScheduleIds);
```

### Fix 2: Method `store()` — tambah filter tanggal dari waktu_ujian

Lokasi: `PesertaUjianController.php`, method `store()`.

**Ganti bagian:**
```php
// SEBELUM
$query = JadwalUjian::where('ujian_id', $validated['ujian_id'])
    ->where('ruang_id', $validated['ruang_id'] ?? null);
if (!empty($validated['sesi'])) {
    $query->where('sesi', $validated['sesi']);
}
$peserta->jadwalUjians()->sync($query->pluck('id'));
```

**Dengan:**
```php
// SESUDAH
$query = JadwalUjian::where('ujian_id', $validated['ujian_id'])
    ->where('ruang_id', $validated['ruang_id'] ?? null);
if (!empty($validated['sesi'])) {
    $query->where('sesi', $validated['sesi']);
}
// Filter by tanggal jika waktu_ujian tersedia
if (!empty($validated['waktu_ujian'])) {
    $query->whereDate('mulai_ujian', date('Y-m-d', strtotime($validated['waktu_ujian'])));
}
$peserta->jadwalUjians()->sync($query->pluck('id'));
```

### Fix 3: Method `update()` — sama seperti store()

Terapkan fix yang sama persis seperti Fix 2 di method `update()`.

### Fix 4: Method `downloadTemplate()` — perbaiki contoh sesi

```php
// SEBELUM
$sheet->setCellValue('F2', 'Sesi 1');

// SESUDAH
$sheet->setCellValue('F2', '1');
```

Juga perbaiki komentar header kolom F agar lebih jelas:
```php
// SEBELUM
$sheet->setCellValue('F1', 'Sesi');

// SESUDAH
$sheet->setCellValue('F1', 'Sesi (isi angka: 1, 2, atau 3)');
```

---

## PENTING: Regenerasi Pivot Data yang Sudah Salah

Fix di atas hanya berlaku untuk import baru. Data pivot yang sudah ada di database masih salah (siswa di-assign ke terlalu banyak jadwal).

Setelah fix kode di atas, **jalankan query SQL ini** untuk regenerasi pivot dari data yang sudah ada:

```sql
-- STEP 1: Hapus semua pivot yang salah
DELETE FROM jadwal_peserta;

-- STEP 2: Isi ulang pivot dengan logic yang benar
-- Untuk peserta DENGAN waktu_ujian (praktek): hanya jadwal di tanggal waktu_ujian
INSERT INTO jadwal_peserta (jadwal_ujian_id, peserta_ujian_id)
SELECT j.id, p.id
FROM peserta_ujians p
JOIN jadwal_ujians j 
  ON j.ujian_id = p.ujian_id
  AND j.ruang_id = p.ruang_id
  AND (j.sesi = p.sesi OR p.sesi IS NULL)
  AND DATE(j.mulai_ujian) = DATE(p.waktu_ujian)
WHERE p.waktu_ujian IS NOT NULL
  AND p.ruang_id IS NOT NULL;

-- Untuk peserta TANPA waktu_ujian (teori): assign ke semua jadwal di ruang+sesi
INSERT INTO jadwal_peserta (jadwal_ujian_id, peserta_ujian_id)
SELECT j.id, p.id
FROM peserta_ujians p
JOIN jadwal_ujians j 
  ON j.ujian_id = p.ujian_id
  AND j.ruang_id = p.ruang_id
  AND (j.sesi = p.sesi OR p.sesi IS NULL)
WHERE p.waktu_ujian IS NULL
  AND p.ruang_id IS NOT NULL;
```

**Jalankan via:**
```bash
docker exec mariadb mariadb -u root berita_acara_ujian_baru -e "DELETE FROM jadwal_peserta;"
docker exec -i mariadb mariadb -u root berita_acara_ujian_baru << 'EOF'
INSERT INTO jadwal_peserta (jadwal_ujian_id, peserta_ujian_id)
SELECT j.id, p.id FROM peserta_ujians p
JOIN jadwal_ujians j ON j.ujian_id = p.ujian_id AND j.ruang_id = p.ruang_id
  AND (j.sesi = p.sesi OR p.sesi IS NULL) AND DATE(j.mulai_ujian) = DATE(p.waktu_ujian)
WHERE p.waktu_ujian IS NOT NULL AND p.ruang_id IS NOT NULL;
INSERT INTO jadwal_peserta (jadwal_ujian_id, peserta_ujian_id)
SELECT j.id, p.id FROM peserta_ujians p
JOIN jadwal_ujians j ON j.ujian_id = p.ujian_id AND j.ruang_id = p.ruang_id
  AND (j.sesi = p.sesi OR p.sesi IS NULL)
WHERE p.waktu_ujian IS NULL AND p.ruang_id IS NOT NULL;
EOF
```

**Setelah regenerasi, verifikasi:**
```bash
docker exec mariadb mariadb -u root berita_acara_ujian_baru -e "
SELECT j.id, DATE(j.mulai_ujian) tanggal, j.ruang_id, j.sesi,
  COUNT(jp.peserta_ujian_id) peserta_count
FROM jadwal_ujians j
LEFT JOIN jadwal_peserta jp ON j.id = jp.jadwal_ujian_id
GROUP BY j.id ORDER BY tanggal LIMIT 20;
"
```
Hasilnya harus menunjukkan peserta_count yang bervariasi per jadwal (bukan semua sama).

---

## Files
- **Modifikasi:** `backend/app/Http/Controllers/PesertaUjianController.php`
  - `importCsv()` — tambah `whereDate` filter
  - `store()` — tambah `whereDate` filter
  - `update()` — tambah `whereDate` filter
  - `downloadTemplate()` — perbaiki contoh sesi dan label kolom F
- **Jangan sentuh:** File lain

## Acceptance Criteria
- [ ] Import ulang peserta dengan `waktu_ujian = 2026-05-30` hanya masuk ke pivot jadwal tanggal 30 Mei, bukan semua hari
- [ ] Peserta tanpa `waktu_ujian` masih masuk ke semua jadwal ruang+sesi-nya (behavior teori tidak berubah)
- [ ] Template download: kolom Sesi menampilkan contoh `'1'` bukan `'Sesi 1'`
- [ ] Template download: label kolom Sesi berisi petunjuk format angka
- [ ] Regenerasi pivot dijalankan dan diverifikasi
- [ ] CONTEXT.md diupdate

## Validasi Claudian
- [x] Fix backward-compatible (tidak break peserta tanpa waktu_ujian)
- [x] Scope kecil: 1 file PHP, 4 method
- [x] Regenerasi pivot menggunakan query idempoten (safe dijalankan ulang)
- [x] Tidak ada perubahan schema/migration
