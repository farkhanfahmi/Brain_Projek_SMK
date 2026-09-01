---
tags: [absensi, T263, dev-only, rollback]
created: 2026-08-30
status: AKTIF -- WAJIB DIROLLBACK setelah screenshot selesai
---

# T263-TEMP -- Override "Hari Wajib" Weekend (Dev-Only, Sementara)

## Kenapa Perubahan Ini Ada

Data seed T263 (kelas X GAME DEV, 22 attendance_records) sudah benar dan tersimpan di
database dev, TAPI halaman "Hari Ini" Wali Kelas tetap menampilkan "Hari ini bukan hari
sekolah (libur/akhir pekan)" karena tanggal eksekusi (2026-08-30) jatuh di hari **Minggu**.

Sistem punya logika `resolveHariWajib()` di `attendance-report.service.ts` yang SENGAJA
mengecualikan Sabtu-Minggu dari "hari wajib" -- dipakai luas (rekap, TV Piket, dashboard
Piket, kalkulasi Alfa), BUKAN bug, melainkan aturan bisnis asli.

**Opsi "tambah SchoolHoliday" TIDAK BISA dipakai** -- weekend di-skip di kode SEBELUM baris
pengecekan holiday, jadi data SchoolHoliday tidak pernah dibaca untuk kasus ini. Satu-satunya
jalan mewujudkan "hari ini dianggap wajib" adalah menyentuh source code (dev-only, dengan
flag eksplisit, TIDAK PERNAH aktif di production).

## PERSIS Apa yang Diubah (2 File)

### File 1 -- `apps/api/src/attendance/attendance-report.service.ts`

**Lokasi**: fungsi `resolveHariWajib()`, baris ~996-997 (nomor baris SEBELUM patch).

**KODE ASLI (WAJIB DIKEMBALIKAN PERSIS SEPERTI INI)**:
```typescript
        const dayOfWeek = cursor.getUTCDay(); // 0=Minggu, 6=Sabtu
        if (dayOfWeek === 0 || dayOfWeek === 6) continue;
```

**KODE SETELAH PATCH (kondisi SEKARANG, 2026-08-30)**:
```typescript
        const dayOfWeek = cursor.getUTCDay(); // 0=Minggu, 6=Sabtu
        // T263-TEMP 2026-08-30 -- flag dev-only utk keperluan screenshot "Hari Ini" Wali
        // Kelas di hari Minggu (lihat 09-t263-temp-weekend-override.md di vault Obsidian
        // utk detail lengkap + instruksi rollback). Default OFF (unset != "true"), TIDAK
        // pernah aktif di production kecuali DEV_TREAT_WEEKEND_AS_WAJIB="true" eksplisit
        // di .env -- WAJIB DIHAPUS setelah screenshot selesai, JANGAN commit permanen.
        const treatWeekendAsWajib = process.env.DEV_TREAT_WEEKEND_AS_WAJIB === "true";
        if ((dayOfWeek === 0 || dayOfWeek === 6) && !treatWeekendAsWajib) continue;
```

**CARA ROLLBACK file ini**: hapus 6 baris komentar + baris `const treatWeekendAsWajib = ...`,
kembalikan baris terakhir jadi persis `if (dayOfWeek === 0 || dayOfWeek === 6) continue;`
(hapus bagian `&& !treatWeekendAsWajib`).

### File 2 -- `apps/api/.env` (dev, BUKAN production)

**Ditambahkan di akhir file** (setelah baris `CORS_ORIGIN=...`):
```
# T263-TEMP 2026-08-30 -- dev-only override, lihat 09-t263-temp-weekend-override.md di vault Obsidian. WAJIB DIHAPUS setelah screenshot selesai.
DEV_TREAT_WEEKEND_AS_WAJIB="true"
```

**CARA ROLLBACK file ini**: hapus 2 baris di atas (baris komentar + baris `DEV_TREAT_WEEKEND_AS_WAJIB`) dari `apps/api/.env`.

## Langkah Rollback Lengkap (Urutan)

1. Buka `apps/api/src/attendance/attendance-report.service.ts`, cari komentar `T263-TEMP` --
   kembalikan ke kode asli persis seperti di atas.
2. Buka `apps/api/.env`, hapus 2 baris `T263-TEMP` + `DEV_TREAT_WEEKEND_AS_WAJIB`.
3. Restart server API dev (`pnpm --filter @absensi/api dev:swc`) supaya `.env` ke-reload --
   PENTING: NestJS watch mode auto-reload untuk perubahan .ts, TAPI TIDAK untuk perubahan
   .env -- restart proses penuh WAJIB, bukan cukup save file.
4. Verifikasi: buka halaman "Hari Ini" Wali Kelas di hari kerja (Senin-Jumat) untuk
   konfirmasi behavior normal kembali (weekend tetap ter-exclude dengan benar).
5. (Opsional, kalau mau bersih total) Hapus data dummy T263 dari database dev: kelas
   "X GAME DEV", 24 siswa terkait, akun `081358390505`, dan attendance_records terkait --
   TIDAK WAJIB kalau dev memang boleh punya data uji coba permanen, tapi dicatat sebagai
   opsi kalau user ingin dev bersih kembali.

## Dampak Selama Flag Ini Aktif (Peringatan)

SELAMA `DEV_TREAT_WEEKEND_AS_WAJIB="true"` aktif di dev:
- SEMUA fitur yang pakai `resolveHariWajib()` terpengaruh, BUKAN cuma halaman "Hari Ini"
  Wali Kelas -- termasuk Rekap Kehadiran, TV Piket, Dashboard Piket, kalkulasi Alfa.
- Kalau ada testing/screenshot fitur LAIN di dev pada hari Sabtu/Minggu selagi flag ini
  aktif, hasilnya TIDAK merepresentasikan behavior production yang sesungguhnya (weekend
  akan dihitung sebagai hari wajib, alfa bisa muncul untuk siswa yang libur wajar).
- **Ini kenapa flag ini WAJIB dicabut segera setelah screenshot selesai, bukan dibiarkan.**

## Status Saat Ini

- [x] Kode dipatch (2026-08-30)
- [x] `.env` dev ditambah flag
- [x] Server API dev direstart, flag AKTIF
- [x] Screenshot halaman "Hari Ini" sudah diambil user
- [x] **ROLLBACK SELESAI (2026-08-31)** — kode `attendance-report.service.ts` dikembalikan
  persis ke kondisi asli (baris `if (dayOfWeek === 0 || dayOfWeek === 6) continue;` tanpa
  flag apapun), 2 baris `T263-TEMP`/`DEV_TREAT_WEEKEND_AS_WAJIB` dihapus dari `apps/api/.env`,
  server API+Web direstart bersih, diverifikasi env var tidak ada sisa (`grep` = 0 match).
  Behavior sistem kembali normal — Sabtu/Minggu tidak lagi dianggap hari wajib di dev.
