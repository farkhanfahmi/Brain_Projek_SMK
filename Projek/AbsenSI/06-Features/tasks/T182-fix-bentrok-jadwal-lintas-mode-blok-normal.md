# T182 — API: Fix Deteksi Bentrok Jadwal Guru Lintas Mode Blok/Normal (Bug T159)

## Depends on
Tidak ada dependency teknis wajib. **Ditemukan saat audit kritis T158+T159 (2026-08-14)** — bug nyata di kode yang SUDAH LIVE, prioritas tinggi karena berdampak langsung ke guru (bentrok jadwal fisik lolos tanpa peringatan sistem).

## Objective
Perbaiki `ensureNoBentrok()` (`schedules.service.ts`) supaya bentrok jadwal guru TERDETEKSI BENAR ketika salah satu jadwal berasal dari kelas **mode normal** (`minggu: null`) dan yang lain dari kelas **mode blok** (`minggu: "A"`/`"B"`) — SAAT INI kombinasi ini LOLOS dari pengecekan bentrok padahal secara kalender nyata keduanya bentrok setiap minggu.

## Context — Bug Ditemukan Saat Audit (2026-08-14)

Audit kritis pasca-eksekusi T158+T159 menemukan celah interaksi yang TIDAK terlihat saat masing-masing task ditulis/dikerjakan terpisah:

`schedules.service.ts` (baris ~354): 
```ts
const mingguCocok = c.minggu === minggu || c.minggu === "setiap_minggu" || minggu === "setiap_minggu";
```

**Sebelum T159**: kelas mode normal SELALU dapat `minggu: "setiap_minggu"` (nilai literal), jadi kondisi ini benar — jadwal normal otomatis "cocok" (bentrok) dengan jadwal manapun.

**SETELAH T159** (`resolveMinggu()`, `schedules.service.ts:280-297`): kelas mode normal SEKARANG dapat `minggu: null` (BUKAN lagi `"setiap_minggu"` — perubahan makna field yang disengaja untuk T159, sudah benar untuk tujuan LAINNYA). TAPI perbandingan `mingguCocok` di `ensureNoBentrok()` **TIDAK IKUT DIPERBARUI** untuk mengenali `null` sebagai "berlaku setiap minggu" — hanya string literal `"setiap_minggu"` yang dikenali.

**Skenario kegagalan konkret**: Guru X dijadwalkan mengajar Kelas A (mode normal, `minggu: null`) hari Senin jam 1-2, DAN Kelas B (mode blok, `minggu: "A"`) hari Senin jam 1-2 yang SAMA (Minggu A). Perbandingan `null === "A"` → false, tidak ada sisi yang `"setiap_minggu"` → `mingguCocok = false` → **bentrok TIDAK terdeteksi**, `create()`/`update()` Schedule kedua BERHASIL tanpa penolakan — padahal secara kalender nyata, di setiap Minggu-A, guru itu punya 2 kelas di jam yang sama secara fisik.

## Spec Detail

### 1. Backend — perbaiki logic `mingguCocok`

- `apps/api/src/core/schedules/schedules.service.ts`, `ensureNoBentrok()` — ganti kondisi `mingguCocok` supaya `null` (mode normal, berlaku SETIAP minggu) diperlakukan SAMA seperti `"setiap_minggu"` — KEDUANYA berarti "cocok dengan minggu apa pun":
  ```ts
  const isEverySelf = minggu === "setiap_minggu" || minggu === null || minggu === undefined;
  const isEveryOther = c.minggu === "setiap_minggu" || c.minggu === null;
  const mingguCocok = isEverySelf || isEveryOther || c.minggu === minggu;
  ```
  (SESUAIKAN sintaks persis dengan tipe TypeScript yang berlaku di file ini — PASTIKAN semantik: kalau SALAH SATU sisi "berlaku setiap minggu" (null ATAU literal "setiap_minggu"), maka OTOMATIS cocok/bentrok, terlepas nilai sisi lainnya).

### 2. Audit titik lain yang mungkin punya bug SERUPA

- **WAJIB grep MENYELURUH** semua tempat lain di codebase yang membandingkan `Schedule.minggu` (selain `ensureNoBentrok`) — kemungkinan ada logic serupa di `ScheduleResolverService`, `TeachingSessionsService`, atau tempat lain yang MUNGKIN punya asumsi lama "minggu selalu terisi string, tidak pernah null" sebelum T159. Kalau ditemukan pola serupa yang BELUM diperbarui, PERBAIKI dengan pola yang SAMA (KONSISTEN, jangan cuma tambal 1 titik).
- **CATATAN dari audit**: `ScheduleResolverService.filterByKelasModeJadwal()` (`schedule-resolver.service.ts:155-168`) SUDAH BENAR menangani null vs blok dengan baik (dikonfirmasi audit terpisah) — method itu TIDAK perlu diubah, HANYA `ensureNoBentrok()` yang bug. TETAP verifikasi ulang saat implementasi, jangan asumsi dari laporan audit saja.

## Edge Cases
- Guru dengan 2 jadwal SAMA-SAMA mode normal (`minggu: null` keduanya) di jam sama — SUDAH bentrok terdeteksi benar (`null === null` true) SEBELUM fix ini juga — TIDAK ADA regresi untuk kasus ini, HANYA kombinasi CAMPURAN (null vs "A"/"B") yang bug.
- Guru dengan 2 jadwal mode blok BEDA minggu (`"A"` vs `"B"`) — TETAP TIDAK bentrok (benar, keduanya jalan di minggu berbeda) — fix ini TIDAK BOLEH mengubah perilaku kasus ini.
- **Data existing yang SUDAH TERLANJUR bentrok** (dibuat selama bug ini aktif, sebelum fix) — task ini TIDAK melakukan pembersihan otomatis/backfill (di luar scope, MURNI perbaikan logic ke depan) — REKOMENDASI: setelah fix dideploy, jalankan QUERY MANUAL (bukan bagian task ini, laporkan sebagai langkah lanjutan opsional ke user) untuk cari jadwal guru yang overlap lintas mode supaya bisa dicek manual apakah ada yang perlu dikoreksi.

## Files
- **Modifikasi:** `apps/api/src/core/schedules/schedules.service.ts` (`ensureNoBentrok()`, logic `mingguCocok`).
- **Jangan sentuh:** `resolveMinggu()` (perilaku `null` untuk mode normal SUDAH BENAR dan disengaja, TIDAK diubah), `ScheduleResolverService.filterByKelasModeJadwal()` (sudah benar, verifikasi ulang saja tanpa mengubah kalau memang benar).

## Acceptance Criteria
- [x] Guru dijadwalkan kelas mode normal (`minggu: null`) DAN kelas mode blok (`minggu: "A"`) di jam sama hari sama → DITOLAK `ConflictException` (bentrok terdeteksi). Diverifikasi 2 arah (normal-baru-vs-blok-existing dan blok-baru-vs-normal-existing).
- [x] Guru dijadwalkan 2 kelas mode blok BEDA minggu (`"A"` vs `"B"`) jam sama → TETAP TIDAK bentrok (regresi nol).
- [x] Guru dijadwalkan 2 kelas mode normal jam sama → TETAP bentrok terdeteksi (regresi nol, sudah benar sebelumnya).
- [x] Grep menyeluruh titik lain yang bandingkan `Schedule.minggu` — HANYA 2 titik ditemukan di seluruh `apps/api/src`: `ensureNoBentrok()` (bug, sudah diperbaiki) dan `ScheduleResolverService.filterByKelasModeJadwal()` (dikonfirmasi ULANG sudah benar — lihat detail di bawah). Tidak ada titik ketiga.
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `nest build` sukses), 4 test baru ditambahkan (2 skenario bug + 2 skenario regresi) — semua PASS setelah fix. Full suite 341/341 (naik dari 337).

## Validasi Claudian
- [x] **Bug direproduksi SEBELUM fix** — 2 test kombinasi lintas-mode ditulis & dijalankan dulu, GAGAL persis seperti diprediksi task (`Received promise resolved instead of rejected` — bentrok tidak terdeteksi), baru fix diterapkan dan test lulus.
- [x] Regresi nol dikonfirmasi via test eksplisit: blok-vs-blok beda minggu tetap TIDAK bentrok, normal-vs-normal tetap bentrok — keduanya lulus baik SEBELUM maupun SESUDAH fix (perilaku lama untuk 2 kasus ini memang sudah benar, tidak berubah).
- [x] Grep menyeluruh (`grep -rn "\.minggu\b"` + `grep -rn "setiap_minggu"` di `apps/api/src`, exclude `.spec.ts`) — hasil: HANYA `ensureNoBentrok()` dan `filterByKelasModeJadwal()` yang membandingkan `Schedule.minggu`. **`filterByKelasModeJadwal()` dikonfirmasi ULANG (bukan cuma percaya laporan audit)**: baris itu mengecek `s.kelas?.modeJadwal !== "blok"` LEBIH DULU (return true / lolos apa adanya utk kelas normal) SEBELUM sampai ke perbandingan `s.minggu === mingguAktif || s.minggu === "setiap_minggu"` — jadi baris `minggu:null` (kelas normal) tidak pernah mencapai perbandingan itu. Ditambah, `resolveMinggu()` (`schedules.service.ts`) melempar `BadRequestException` kalau kelas mode blok tapi `minggu` kosong — jadi invariant "kelas mode blok SELALU punya minggu terisi (bukan null)" terjamin di titik input, method ini memang tidak bisa kena bug yang sama. TIDAK diubah.

## Catatan pasca-implementasi
Fix diterapkan di `ensureNoBentrok()` — logic baru: `isEverySelf` (sisi jadwal baru berlaku setiap minggu: literal `"setiap_minggu"` ATAU `null`/`undefined`) OR `isEveryOther` (sisi kandidat existing berlaku setiap minggu, sama) OR nilai `minggu` identik persis. **Data existing yang mungkin sudah bentrok** (dibuat selama bug aktif, antara eksekusi T158/T159 dan fix T182 ini) belum di-query/dicek — sesuai scope task ini murni perbaikan logic ke depan. **Rekomendasi ke user**: jalankan query manual untuk cari jadwal guru overlap lintas mode di data dev/production kalau ingin memastikan tidak ada jadwal yang terlanjur bentrok sejak T159 dieksekusi (2026-08-14).
