# T181 — API: Fix Race Condition di `JamPelajaranService.update()` — Cakupan Tingkat vs Aktivasi

## Depends on
Tidak ada dependency teknis wajib. **Ditemukan saat audit kritis T158+T159 (2026-08-14)** — celah race condition, probabilitas kejadian rendah (butuh 2 admin melakukan aksi bersamaan dalam window sempit) tapi dampaknya nyata kalau terjadi (aktivasi "yatim" — mengarah ke opsi yang sudah tidak valid untuk tingkat itu).

## Objective
`JamPelajaranService.update()` — validasi "tolak ubah cakupan tingkat kalau opsi ini SEDANG AKTIF di tingkat yang mau dihapus dari cakupan" WAJIB dilakukan DI DALAM transaksi yang sama dengan mutasi, BUKAN dibaca sebagai snapshot SEBELUM transaksi terpisah — supaya tidak ada window race dengan `activate()` yang berjalan bersamaan.

## Context — Bug Ditemukan Saat Audit (2026-08-14)

`apps/api/src/jam-pelajaran/jam-pelajaran.service.ts` (baris ~114-163) — `update()` SAAT INI:
1. Baca `before = findOne(id)` (termasuk `before.aktivasi`) — snapshot LUAR transaksi.
2. Validasi: kalau cakupan baru tidak lagi mencakup tingkat yang ADA di `before.aktivasi` → tolak.
3. KALAU lolos — jalankan `$transaction` TERPISAH yang benar-benar mengubah `semuaTingkat`+`tingkatScopes`+slot.

**Skenario kegagalan konkret** (TOCTOU — time-of-check-to-time-of-use):
- T=0: Admin A buka form edit opsi "Jam Reguler" (saat ini `semuaTingkat: true`, AKTIF untuk tingkat X dan XI) — mau ubah jadi cakupan khusus XII saja. Validasi baca `before.aktivasi` = [X, XI] — TIDAK ADA konflik terdeteksi KARENA validasi ini mengecek terhadap cakupan BARU vs aktivasi LAMA, TAPI baru dieksekusi di step berikutnya.
- T=1 (SEBELUM Admin A commit): Admin B memanggil `activate(optionId, "XI", ...)` — LOLOS validasi `activate()` (opsi ini MASIH `semuaTingkat: true` di database saat itu, jadi valid mencakup XI) — `JamPelajaranAktivasi` tingkat XI di-upsert menunjuk opsi ini.
- T=2: Transaksi Admin A commit — opsi berubah jadi `semuaTingkat: false`, cakupan HANYA XII.
- **Hasil akhir**: `JamPelajaranAktivasi` untuk tingkat XI MASIH menunjuk ke opsi yang SEKARANG hanya mencakup XII — aktivasi "yatim"/tidak valid, TAPI sistem tidak sadar (tidak ada re-validasi setelah fakta).
- **Dampak**: `getActiveForTingkat("XI")`/`resolveJamSchedule()` untuk tingkat XI akan TETAP "berhasil" resolve jam dari opsi yang seharusnya sudah tidak berlaku untuk XI — kelas/guru tingkat XI dapat jam yang SALAH tanpa sistem menolak, PERSIS kondisi yang spec T158 minta dicegah (`T158.md` Edge Cases) tapi lolos karena race.

## Spec Detail

### 1. Backend — pindahkan validasi ke DALAM transaksi

- `apps/api/src/jam-pelajaran/jam-pelajaran.service.ts`, `update()` — REFAKTOR supaya SELURUH proses (baca aktivasi terkini + validasi + mutasi) terjadi DALAM 1 `$transaction`:
  ```ts
  return this.prisma.$transaction(async (tx) => {
    const currentAktivasi = await tx.jamPelajaranAktivasi.findMany({ where: { optionId: id } });
    // VALIDASI: untuk setiap currentAktivasi.tingkat, pastikan tingkat itu MASIH akan
    // tercakup oleh cakupan BARU (semuaTingkat baru, ATAU tingkat baru ada di tingkatScopes baru)
    // — kalau ADA yang tidak lagi tercakup, throw ConflictException SEBELUM mutasi apa pun.
    // ... baru lakukan mutasi opsi + tingkatScopes + slot di transaksi YANG SAMA ini.
  });
  ```
- **PERTIMBANGAN KUNCI**: Prisma `$transaction` (interactive, dengan callback `tx`) di MySQL default isolation level `REPEATABLE READ` — VERIFIKASI apakah ini SUDAH CUKUP untuk mencegah race ini, atau perlu row-lock eksplisit (`SELECT ... FOR UPDATE` via `tx.$queryRaw` pada baris `JamPelajaranAktivasi` terkait `optionId` ini) supaya `activate()` yang berjalan BERSAMAAN benar-benar terblokir/menunggu sampai transaksi `update()` ini selesai — JANGAN asumsi isolation level default otomatis aman tanpa verifikasi, MySQL REPEATABLE READ TIDAK selalu mencegah race SEMACAM INI (baca-lalu-tulis di baris BEDA tabel/kondisi, bukan baca-tulis baris yang sama).

### 2. Backend — pertimbangkan proteksi SIMETRIS di `activate()`

- **VERIFIKASI**: apakah `activate()` (`jam-pelajaran.service.ts:~180-208`) JUGA perlu row-lock terhadap opsi yang mau diaktifkan (supaya tidak "menang race" terhadap `update()` yang sedang mengubah cakupan opsi yang sama) — idealnya KEDUA method (`update()` DAN `activate()`) saling terkunci lewat baris yang SAMA (baik row-lock opsi ATAU row-lock aktivasi tingkat terkait) supaya urutan eksekusi APAPUN (A duluan atau B duluan) menghasilkan STATE YANG KONSISTEN (bukan cuma salah satu method yang "menang", tapi TIDAK ADA kombinasi urutan yang menghasilkan aktivasi yatim).

## Edge Cases
- Admin A dan B melakukan aksi BERBEDA (bukan `update()`+`activate()` tapi `update()`+`update()` opsi yang SAMA bersamaan) — transaksi kedua (yang commit belakangan) HARUS melihat state TERBARU dari transaksi pertama (REPEATABLE READ + retry-on-conflict kalau perlu, ATAU serialization error yang WAJAR ditangkap dan di-retry/dilaporkan sebagai "opsi ini baru saja diubah, coba lagi").
- Fix ini TIDAK BOLEH mengubah perilaku validasi untuk kasus NORMAL (tanpa race — 1 admin, tidak ada aksi bersamaan) — hasil akhirnya harus IDENTIK dengan sebelumnya untuk kasus yang sudah benar.

## Files
- **Modifikasi:** `apps/api/src/jam-pelajaran/jam-pelajaran.service.ts` (`update()`, PERTIMBANGKAN `activate()` untuk proteksi simetris).
- **Jangan sentuh:** `delete()` (SUDAH aman, ada FK backstop alami — dikonfirmasi audit, TIDAK perlu perubahan).

## Acceptance Criteria
- [x] Test concurrent (simulasi mock, bukan DB nyata): `update()` yang mengubah cakupan opsi yang lagi aktif, dijalankan dengan snapshot `before` stale yang meniru `activate()` sudah commit duluan di antara baca-snapshot dan mutasi — hasil KONSISTEN, `update()` ditolak `ConflictException` (tidak pernah menghasilkan aktivasi yatim).
- [x] Kasus normal (tanpa concurrency) — perilaku `update()` IDENTIK dengan sebelumnya (3 test regresi: cakupan tanpa aktivasi terdampak berhasil, cakupan yang masih mencakup aktivasi berhasil, cakupan yang tidak lagi mencakup tetap ditolak).
- [x] Build + type-check hijau, 7 test baru ditambahkan (`jam-pelajaran.service.spec.ts`, file baru — belum ada test sebelumnya untuk modul ini).

## Validasi Claudian
- [x] **Race direproduksi SEBELUM fix** — test "BUG (sebelum fix)/FIX (setelah fix)" ditulis dan dijalankan dulu terhadap kode lama: GAGAL (`Received promise resolved instead of rejected` — mutasi lolos padahal semestinya ditolak), membuktikan bug nyata bukan cuma laporan audit. Setelah fix diterapkan, test yang sama PASS.
- [x] **REPEATABLE READ TIDAK CUKUP, dikonfirmasi lewat analisis + fix row-lock eksplisit diterapkan** — root cause race ini murni application-level TOCTOU (baca `before` DI LUAR transaksi terpisah dari transaksi mutasi), bukan soal isolation level transaksi Prisma itu sendiri. Fix: `SELECT id FROM jam_pelajaran_options WHERE id = ? FOR UPDATE` (pola SAMA persis dengan preseden existing di `block-week-ranges.service.ts`) diambil PALING AWAL di dalam `$transaction`, SEBELUM re-read aktivasi TERKINI (bukan snapshot lama) dan mutasi — dikonfirmasi via test "row-lock diambil PALING AWAL" (urutan call: lock → read-aktivasi → mutate).
- [x] **Proteksi simetris DITAMBAHKAN di `activate()`** — row-lock SAMA (`jam_pelajaran_options` baris `id` yang sama) diambil di awal transaksi `activate()` juga, SEBELUM validasi cakupan + upsert aktivasi. Alasan: tanpa ini, `activate()` bisa "menang race" terhadap `update()` yang sedang mengubah cakupan (baca cakupan lama yang masih valid, padahal `update()` sedang mengubahnya jadi tidak valid) — dengan lock yang sama di kedua method, urutan eksekusi APAPUN (update duluan atau activate duluan) saling menunggu dan menghasilkan state konsisten. Dikonfirmasi via test "activate() mengambil row-lock yang SAMA... sebelum validasi+upsert". **`deactivate()` TIDAK diberi row-lock** — dipertimbangkan tapi disimpulkan tidak perlu: method ini hanya MENGHAPUS aktivasi (efek monoton mengurangi state), tidak pernah bisa menghasilkan aktivasi yatim baru terlepas urutan eksekusinya terhadap `update()`.
