# T208 — API: Date Generator Minggu A/B (Menggantikan BlockWeekRange)

## Depends on
**WAJIB SETELAH T203, T206** (butuh model `OpsiJadwalMingguGenerate` dan service `OpsiJadwalService` sudah ada). Bagian dari rangkaian T203-T215.

## Objective
Endpoint backend untuk **generate otomatis daftar tanggal individual** Minggu A dan Minggu B dari 2 titik tanggal mulai (selisih PERSIS 7 hari), skip hari libur (`SchoolHoliday`) dan skip hari yang tidak ada di Alokasi Waktu terkait — menggantikan `BlockWeekRange` (rentang manual per baris).

## Konteks — Mekanisme Dikonfirmasi User (2026-08-16)

Contoh nyata dari diskusi: admin input **Minggu A mulai 20 Juli 2026** (sampai batas akhir, misal 12 Desember 2026) dan **Minggu B mulai 27 Juli 2026** (sampai batas akhir, misal 19 Desember 2026) — selisih titik mulai A dan B **SELALU 7 hari**, sehingga sistem otomatis tahu pola berselang-seling A-B-A-B... tanpa admin input blok manual satu-satu. Sistem generate SEMUA tanggal kelipatan 7 hari dari masing-masing titik mulai, DIKURANGI tanggal yang masuk `SchoolHoliday`, DIKURANGI hari yang tidak ada slot-nya di `AlokasiWaktu` terkait Opsi Jadwal ini.

## Spec Detail

### 1. Backend — endpoint generate

- `POST /opsi-jadwal/:id/generate-minggu` — HANYA valid untuk `OpsiJadwal.mode === blok` (TOLAK dengan pesan jelas kalau dipanggil untuk Opsi mode normal). Body: `{ mingguAMulai: string, mingguASelesai: string, mingguBMulai: string, mingguBSelesai: string }`.
- **VALIDASI SEBELUM generate**:
  1. `mingguBMulai` HARUS PERSIS 7 hari setelah `mingguAMulai` (ATAU SEBALIKNYA — VERIFIKASI arah mana yang wajib, REKOMENDASI: terima KEDUA arah asalkan selisih ABSOLUT 7 hari, TOLAK kalau selisihnya BUKAN 7 hari dengan pesan jelas "Tanggal mulai Minggu A dan Minggu B harus berselisih tepat 7 hari").
  2. Kedua rentang (`mingguAMulai`-`mingguASelesai`, `mingguBMulai`-`mingguBSelesai`) HARUS berada DALAM rentang tanggal `Semester` induk `OpsiJadwal` ini (TOLAK kalau keluar rentang semester, pesan jelas sebutkan rentang semester yang valid).
- **ALGORITMA GENERATE**:
  1. Loop dari `mingguAMulai`, lompat +7 hari tiap iterasi, sampai `mingguASelesai` — untuk TIAP tanggal hasil: cek `SchoolHoliday` (skip kalau tanggal itu hari libur), cek `AlokasiWaktuSlot` OpsiJadwal ini punya slot untuk `hari` tanggal itu (skip kalau alokasi waktu TIDAK punya jadwal untuk hari itu, misal alokasi tidak punya slot Sabtu dan tanggal jatuh hari Sabtu) — SISANYA masuk daftar `OpsiJadwalMingguGenerate` dengan `minggu: A`.
  2. SAMA untuk Minggu B, `minggu: B`.
  3. **REPLACE SELURUH baris `OpsiJadwalMingguGenerate` untuk Opsi Jadwal ini** (hapus semua lama kalau generate ulang, insert baru — KONSISTEN pola REPLACE PENUH T206) dalam 1 `$transaction`.
- Response: ringkasan hasil (`{mingguACount, mingguBCount, tanggalLibur SkippedCount}`) — supaya admin tahu berapa tanggal yang di-skip dan kenapa.

### 2. Backend — endpoint lihat/edit hasil generate

- `GET /opsi-jadwal/:id/minggu-generate` — list semua tanggal hasil generate, grouped by minggu A/B (untuk ditampilkan di UI T210).
- `DELETE /opsi-jadwal/:id/minggu-generate/:tanggalId` — HAPUS 1 tanggal SPESIFIK dari hasil generate (edge case: admin mau exclude 1 tanggal manual di luar hari libur resmi, misal acara sekolah mendadak) — **VERIFIKASI kebutuhan ini dengan user saat implementasi kalau ragu**, TIDAK disebutkan eksplisit di diskusi tapi MASUK AKAL sebagai fitur pelengkap — KALAU user TIDAK butuh ini, SKIP endpoint ini, JANGAN paksa buat fitur yang tidak diminta.

## Edge Cases
- Generate ULANG untuk Opsi Jadwal yang SUDAH punya `JadwalSlot` dengan `minggu: A/B` terisi (jadwal sudah diisi sebelum generate ulang) — REPLACE `OpsiJadwalMingguGenerate` TIDAK mempengaruhi `JadwalSlot` yang sudah ada (keduanya model TERPISAH, `JadwalSlot.minggu` cuma tag `A`/`B`, TIDAK bergantung langsung ke baris `OpsiJadwalMingguGenerate` tertentu) — TAPI KALAU generate ulang menghasilkan pola BERBEDA (misal tanggal mulai diubah), ada RISIKO jadwal lama jadi "salah" secara semantik (kelas yang sudah diisi jadwal Minggu A ternyata sekarang di tanggal itu jatuhnya Minggu B menurut generate baru) — TAMBAHKAN WARNING di UI sebelum generate ulang KALAU Opsi Jadwal ini SUDAH punya JadwalSlot terisi ("Opsi ini sudah punya N jadwal terisi, generate ulang bisa membuat penandaan Minggu A/B tidak sinkron dengan jadwal yang sudah dibuat — lanjutkan?").
- Rentang generate SANGAT PANJANG (hampir 1 semester penuh, misal 5 bulan) — PERFORMA: pastikan loop tidak N+1 query per tanggal (batch query `SchoolHoliday` SEKALI untuk seluruh rentang, bukan query per-iterasi).

## Files
- **Modifikasi:** `apps/api/src/opsi-jadwal/opsi-jadwal.service.ts` (method `generateMingguAB()`), `apps/api/src/opsi-jadwal/opsi-jadwal.controller.ts` (endpoint baru).
- **Jangan sentuh:** `BlockWeekRangeService`/`BlockWeekRangeController` LAMA (TIDAK dihapus/diubah sampai T215).

## Acceptance Criteria
- [x] Generate dengan 2 titik mulai berselisih 7 hari — berhasil, hasil daftar tanggal individual A dan B.
- [x] Generate dengan selisih BUKAN 7 hari — ditolak dengan pesan jelas.
- [x] Tanggal yang jatuh di `SchoolHoliday` — otomatis di-skip dari hasil.
- [x] Tanggal yang hari-nya tidak ada slot di Alokasi Waktu terkait (misal Sabtu kosong) — otomatis di-skip.
- [ ] Generate ulang untuk Opsi yang SUDAH punya JadwalSlot terisi — WARNING ditampilkan (kalau diimplementasi di FE T210, dicatat sebagai dependency) — **BELUM diimplementasi, MURNI tanggung jawab FE T210** (backend T208 ini tidak punya UI, warning butuh interaksi user sebelum submit).
- [x] Performa: batch query SchoolHoliday, tidak N+1 per tanggal.
- [x] Build + type-check hijau, jest untuk skenario generate lengkap (termasuk skip libur, skip hari kosong, validasi selisih 7 hari).

## Validasi Claudian
- [x] **WAJIB verifikasi** batch query SchoolHoliday (bukan query per-tanggal dalam loop) — DIKONFIRMASI test eksplisit `expect(prisma.schoolHoliday.findMany).toHaveBeenCalledTimes(1)`, bukan asumsi.
- [x] Konfirmasi endpoint `DELETE /minggu-generate/:tanggalId` dikerjakan HANYA kalau dikonfirmasi user butuh — **DITANYAKAN via AskUserQuestion, user KONFIRMASI butuh** — endpoint dibuat.

## Status Eksekusi

**Selesai 2026-08-17 05:50**

Modifikasi `apps/api/src/opsi-jadwal/opsi-jadwal.service.ts` (bukan modul baru, sesuai spec "Files"): tambah `generateMingguAB()`, `findMingguGenerate()`, `deleteMingguGenerateTanggal()`, plus DTO baru `dto/generate-minggu-ab.dto.ts`. Controller ditambah 3 endpoint: `POST .../generate-minggu`, `GET .../minggu-generate`, `DELETE .../minggu-generate/:tanggalId`.

Detail keputusan:
- Selisih 7 hari dicek via `Math.abs()` pada selisih milidetik `mingguBMulai - mingguAMulai`, menerima KEDUA arah (A duluan atau B duluan) sesuai rekomendasi spec.
- Validasi rentang-dalam-semester dicek utk KEDUA sisi (A dan B) terpisah, pesan sebutkan rentang semester valid.
- `hari` dikonversi `getUTCDay() + 1` (KONSISTEN konvensi MySQL DAYOFWEEK 1=Minggu..7=Sabtu dipakai di seluruh modul jadwal baru), dicocokkan ke `Set<number>` hasil batch-query `AlokasiWaktuSlot` (filter `jamKe: { not: null }` — baris istirahat tidak dihitung "hari tersedia").
- REPLACE PENUH `OpsiJadwalMingguGenerate` per Opsi Jadwal (deleteMany lalu createMany dalam 1 transaksi) — dikonfirmasi urutan panggilan via test.
- Warning UI utk generate-ulang Opsi yang sudah punya `JadwalSlot` terisi TIDAK diimplementasi di sini — murni concern presentasi (perlu dialog konfirmasi sebelum submit), didelegasikan ke T210 sebagai dependency eksplisit sesuai catatan spec Edge Cases.
- Test: 14 baru untuk `generateMingguAB`/`deleteMingguGenerateTanggal` (di file yang sama dengan 21 test T206 existing → total 35 di `opsi-jadwal.service.spec.ts`) + 546 test total lulus zero-regresi.
