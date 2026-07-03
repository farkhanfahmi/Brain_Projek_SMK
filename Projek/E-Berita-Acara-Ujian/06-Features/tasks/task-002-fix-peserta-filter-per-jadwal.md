---
tags:
  - task
  - bugfix
created: 2026-06-04
status: ready
---

# Task-002: Fix Filter Peserta — Hanya Tampilkan Siswa Sesuai Jadwal Hari Ini

## Objective
Perbaiki bug di mana halaman scan pengawas dan menu Manual Kehadiran menampilkan SEMUA siswa di ruangan tersebut lintas semua hari, padahal seharusnya hanya siswa yang terjadwal di jadwal spesifik hari itu yang ditampilkan.

## Context
- **Modul:** AssignmentService + ExamReportController (rekapAdmin)
- **DB Tables:** `jadwal_peserta` (pivot), `peserta_ujians`, `jadwal_ujians`
- **API Endpoints:** `GET /get-assignment`, `GET /rekap-admin`, `GET /peserta-all`
- **Frontend yang terdampak:** `frontend/` (scan pengawas), `frontend-admin/` (ManualAttendance)

## Root Cause (Sudah Dianalisis — Jangan Analisis Ulang)

### Fakta dari Database
```
peserta_ujians: 3.919 records — semua punya ruang_id, sesi, ujian_id ✅
jadwal_peserta: 8.002 rows — pivot per-jadwal PER hari berisi data
```

### Bug di `AssignmentService::getParticipants()`
```php
// LOGIKA SALAH — pivot dipakai sebagai FALLBACK, bukan PRIMARY
if (!empty($ruangIds)) {
    $peserta = PesertaUjian::where('ujian_id', $ujianId)
        ->whereIn('ruang_id', $ruangIds)   // ← room filter, TANPA tanggal
        ->get();                            // ← selalu return hasil (133 siswa/room)
}

// Ini TIDAK PERNAH JALAN karena query atas tidak pernah kosong:
if ($peserta->isEmpty()) {
    // pivot fallback — seharusnya ini yang jadi PRIMARY
}
```

`peserta_ujians` tidak punya kolom tanggal → filter by `ruang_id` saja = semua siswa di room itu lintas semua hari.

### Bug di `ExamReportController::rekapAdmin()` (untuk Manual Kehadiran)
Sama persis — filtering by `ruang_id` dari jadwal tanggal yang dipilih, tapi tetap return semua siswa di room tersebut.

---

## LANGKAH PERTAMA WAJIB — Investigasi Data Sebelum Fix

**Sebelum menulis kode apapun**, jalankan query ini untuk memahami struktur data pivot:

```bash
docker exec mariadb mariadb -u root berita_acara_ujian_baru -e "
-- Apakah pivot berisi data yang benar?
SELECT 
  j.id as jadwal_id,
  j.ujian_id,
  j.ruang_id,
  j.sesi,
  DATE(j.mulai_ujian) as tanggal,
  COUNT(jp.peserta_ujian_id) as jumlah_peserta_di_pivot
FROM jadwal_ujians j
LEFT JOIN jadwal_peserta jp ON j.id = jp.jadwal_ujian_id
GROUP BY j.id
ORDER BY tanggal, j.ruang_id, j.sesi
LIMIT 50;
"
```

```bash
docker exec mariadb mariadb -u root berita_acara_ujian_baru -e "
-- Jadwal mana yang SUDAH punya pivot dan berapa jumlahnya?
SELECT 
  COUNT(DISTINCT jadwal_ujian_id) as jadwal_dengan_pivot,
  COUNT(*) as total_pivot_rows
FROM jadwal_peserta;
"
```

```bash
docker exec mariadb mariadb -u root berita_acara_ujian_baru -e "
-- Sample: siapa siswa yang ada di pivot untuk jadwal tertentu?
SELECT jp.jadwal_ujian_id, jp.peserta_ujian_id, p.nama, p.nomor_peserta, p.sesi
FROM jadwal_peserta jp
JOIN peserta_ujians p ON jp.peserta_ujian_id = p.id
LIMIT 20;
"
```

**Laporkan hasilnya di CONTEXT.md bagian 'Masalah Ditemukan' sebelum melanjutkan ke fix.**

---

## Spec Fix

### Skenario A: Pivot berisi data per-jadwal yang benar (jumlah siswa per jadwal masuk akal)
**→ Fix: Balik logika priority di `getParticipants()` — pivot jadi PRIMARY**

```php
// File: backend/app/Services/AssignmentService.php
// Method: getParticipants()

private function getParticipants(int $ujianId, array $ruangIds, array $sesiList, array $jadwalIds)
{
    // PRIMARY: gunakan pivot table (mapping siswa ke jadwal spesifik)
    if (!empty($jadwalIds)) {
        $peserta = PesertaUjian::whereHas('jadwalUjians', function ($q) use ($jadwalIds) {
            $q->whereIn('jadwal_ujians.id', $jadwalIds);
        })->get();

        if ($peserta->isNotEmpty()) {
            return $peserta;
        }
    }

    // FALLBACK: jika pivot kosong untuk jadwal ini, pakai ruang + sesi
    // (untuk ujian lama yang belum punya data pivot)
    if (!empty($ruangIds)) {
        $query = PesertaUjian::where('ujian_id', $ujianId)
            ->whereIn('ruang_id', $ruangIds);

        if (!empty($sesiList)) {
            $query->where('sesi', $sesiList);
            // HAPUS orWhereNull('sesi') — terlalu broad
        }

        return $query->get();
    }

    return collect();
}
```

### Skenario B: Pivot KOSONG atau tidak punya data per-hari yang benar
**→ Laporkan ke CONTEXT.md, jangan fix kode dulu. Kita perlu diskusi dengan user tentang bagaimana data seharusnya distruktur.**

---

## Fix untuk `rekapAdmin()` (Manual Kehadiran)

Setelah fix `getParticipants()`, ada bagian serupa di `ExamReportController::rekapAdmin()`.

Cari bagian ini di `backend/app/Http/Controllers/ExamReportController.php`:
```php
// Sekitar baris 450-475 (bagian filtering peserta berdasarkan tanggal)
$pivotPesertaIds = DB::table('jadwal_peserta')
    ->whereIn('jadwal_ujian_id', $jadwalIds)
    ->pluck('peserta_ujian_id')->toArray();
```

Jika Skenario A berlaku: ubah query peserta agar **prioritaskan pivot** juga:
```php
if (!empty($pivotPesertaIds)) {
    // Jika ada data pivot, gunakan pivot saja — ini yang paling akurat per-tanggal
    $pesertas = PesertaUjian::whereIn('id', $pivotPesertaIds)->with(['ruang'])->get();
} else {
    // Fallback: filter by ruang + sesi dari jadwal hari itu
    // (logika existing yang sudah ada)
    $pesertas = PesertaUjian::where('ujian_id', $ujianId)
        ->whereIn('ruang_id', $ruangIds)
        // ...
}
```

---

## Fix untuk `getPesertaAll()` (Endpoint panitia)

Di `ExamReportController::getPesertaAll()`, ada filter by `pengawas_id` yang juga gunakan pivot:
```php
$jadwals = JadwalUjian::where('ujian_id', $ujianId)
    ->where(function($q) use ($pengawasId) { ... })->get();
$jadwalIds = $jadwals->pluck('id');
$pesertaIds = DB::table('jadwal_peserta')->whereIn('jadwal_ujian_id', $jadwalIds)->pluck('peserta_ujian_id');
$pesertaQuery->whereIn('id', $pesertaIds);
```
Bagian ini sudah benar — jika pivot ada, hasilnya benar. Tidak perlu diubah.

---

## Files
- **Modifikasi:** `backend/app/Services/AssignmentService.php` — method `getParticipants()` (PRIMARY FIX)
- **Modifikasi:** `backend/app/Http/Controllers/ExamReportController.php` — method `rekapAdmin()` (SECONDARY FIX)
- **Jangan sentuh:** Frontend, migration, model, seeder

## Acceptance Criteria

- [ ] Investigasi pivot selesai dan hasil dilaporkan di CONTEXT.md
- [ ] Skenario (A atau B) sudah ditentukan sebelum kode ditulis
- [ ] **Jika Skenario A:** `GET /get-assignment?ujian_id=X&pengawas_id=Y` hanya return siswa yang ada di `jadwal_peserta` untuk jadwal hari itu, bukan semua siswa di ruangan
- [ ] **Jika Skenario A:** `GET /rekap-admin?ujian_id=X&tanggal=Y` hanya return siswa yang dijadwalkan hari Y, bukan semua siswa di room yang dipakai hari Y
- [ ] `orWhereNull('sesi')` sudah dihapus dari `getParticipants()`
- [ ] Fallback ke room-based masih bekerja jika pivot kosong untuk jadwal tertentu
- [ ] CONTEXT.md diupdate dengan hasil

## Validasi Claudian
- [x] Root cause sudah confirmed dengan data database
- [x] Fix tidak akan break ujian yang tidak punya pivot (ada fallback)
- [x] Scope kecil: 2 file PHP, masing-masing ~10-15 baris perubahan
- [x] Tidak ada konflik dengan ADR yang ada
