# T180 — API+Web: Tombol "Sudah Pulang" Bulk — Section "Tidak Absen Pulang"

## Depends on
**REKOMENDASI setelah T179** (pola bulk generic best-effort+report sebaiknya sudah ada 1 preseden dari T179 sebelum diterapkan ke section ini yang punya nuansa tambahan). Independen dari T158 (Jam Pelajaran) UNTUK VERSI AWAL task ini — requirement "waktuPulang = jam terakhir pelajaran" secara EKSPLISIT ditunda ke T158 (lihat Context), task INI cukup pakai fallback null.

## Objective
Section "Tidak Absen Pulang" (siswa KEMARIN yang tap masuk tapi tidak tap pulang) — tombol **"Sudah Pulang"** untuk memulangkan SEMUA siswa di daftar itu sekaligus dalam 1 klik, TANPA perlu klarifikasi satu-satu.

## Context — Keputusan Diskusi (2026-08-14)

- Section ini (`piket-board-view.tsx:682-779`, judul "Tidak Absen Pulang **Kemarin**") — data dari `GET /attendance/tidak-absen-pulang-kemarin`, soal siswa KEMARIN (bukan hari ini). Aksi individual SEKARANG: tombol "Klarifikasi" → dialog `TidakAbsenPulangForm` dengan 2 sub-aksi terpisah (Konfirmasi Pulang Normal via `POST /attendance/:record_id/konfirmasi-pulang-retroaktif`, Tandai Izin Tidak Kembali via `POST /permits/tandai-izin-tidak-kembali`).
- **Requirement asli user**: waktu pulang untuk aksi bulk ini SEHARUSNYA otomatis mengikuti "jam terakhir pelajaran hari itu" — TAPI riset mengonfirmasi TIDAK ADA cara sistem tahu ini sekarang (fitur Jam Pelajaran T158 belum dieksekusi). **User EKSPLISIT mengarahkan**: kalau syarat ini butuh T158, PERBAIKAN itu ditulis DI T158 (bukan di task ini) — task INI diabaikan untuk bagian itu, fokus HANYA ke mekanisme bulk-nya.
- **Keputusan fallback (dikonfirmasi user)**: SEBELUM T158 tersedia, tombol bulk "Sudah Pulang" ini mengisi `waktuPulang` dengan **null** (dicatat "tidak diketahui") — KONSISTEN perilaku existing `konfirmasiPulangRetroaktif()` SAAT `waktuPulang` tidak diisi (`attendance.service.ts:707-718`, komentar DTO eksplisit: "kalau tidak diisi, dicatat sebagai 'tidak diketahui' — hanya pulang_via yang berubah"). **JANGAN pakai `new Date()`/`now()`** — section ini soal KEMARIN, waktu sekarang tidak masuk akal secara semantik sebagai jam pulang kemarin.
- Method backend `konfirmasiPulangRetroaktif()` SUDAH ADA dan SUDAH mendukung `waktuPulang` opsional persis sesuai kebutuhan fallback ini — task ini REUSE method itu, HANYA perlu versi BULK-nya (terima banyak `attendanceRecordId` sekaligus).

## Spec Detail

### 1. Backend — endpoint bulk baru

- `POST /attendance/konfirmasi-pulang-retroaktif-bulk` — body `{attendanceRecordIds: number[]}` (TIDAK ADA parameter `waktuPulang` — SELALU null/tidak diisi untuk versi bulk ini, sesuai keputusan fallback). Service: loop panggil `konfirmasiPulangRetroaktif(recordId, undefined)` PER id, **best-effort** (KONSISTEN pola T179 — 1 gagal tidak menggagalkan yang lain), return `{successCount, failedCount, errors: [{attendanceRecordId, reason}]}`.
- Guard SAMA seperti endpoint single (`PiketOnDutyGuard`), REUSE.
- `@LogActivity` PER BARIS yang berhasil (KONSISTEN T179).

### 2. Frontend — tombol bulk di section "Tidak Absen Pulang"

- `piket-board-view.tsx` — tombol **"Sudah Pulang Semua"** di header section (label eksplisit, BUKAN "Eksekusi Semua" generic).
- Dialog konfirmasi WAJIB — sebutkan jumlah siswa terpengaruh DAN **peringatan eksplisit** bahwa jam pulang akan dicatat "tidak diketahui" (bukan jam pasti) — supaya piket paham konsekuensinya sebelum klik (KONSISTEN prinsip transparansi, JANGAN sembunyikan bahwa data waktu tidak presisi).
- Setelah eksekusi — hasil ringkas (berhasil/gagal) + refresh section.
- **TIDAK MENGGANTIKAN tombol "Klarifikasi" individual yang sudah ada** — bulk ini adalah TAMBAHAN untuk kasus "piket yakin semua ini memang pulang normal, tidak perlu klarifikasi detail satu-satu", tombol single tetap ada untuk kasus yang butuh Tandai Izin Tidak Kembali (bulk TIDAK mencakup opsi itu, HANYA opsi "Konfirmasi Pulang Normal").

## Edge Cases
- Section kosong — tombol bulk disabled/tersembunyi (KONSISTEN T179).
- Siswa di daftar yang SEBENARNYA perlu ditandai "Izin Tidak Kembali" (bukan pulang normal) — bulk ini TIDAK COCOK untuk kasus itu, piket TETAP HARUS pakai tombol "Klarifikasi" individual untuk baris itu — dialog konfirmasi bulk SEBAIKNYA sebutkan peringatan ini juga ("pastikan semua siswa di daftar ini memang pulang normal, bukan izin tidak kembali").
- **Follow-up wajib ke T158**: setelah T158 (Jam Pelajaran) selesai, endpoint/tombol ini SEHARUSNYA diperbarui untuk isi `waktuPulang` otomatis dari jam terakhir pelajaran hari itu (BUKAN lagi null) — INI DICATAT SEBAGAI CATATAN DI T158 (lihat task itu), BUKAN dikerjakan di task ini.

## Files
- **Modifikasi:** `apps/api/src/attendance/attendance.controller.ts`+`attendance.service.ts` (endpoint bulk baru), `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (tombol "Sudah Pulang Semua" + dialog).
- **Jangan sentuh:** `konfirmasiPulangRetroaktif()` single (REUSE apa adanya, TIDAK diubah signature), tombol "Klarifikasi" individual (TETAP ADA, tidak dihapus/digantikan), `tandaiIzinTidakKembali()` (di luar scope bulk ini).

## Acceptance Criteria
- [x] Section "Tidak Absen Pulang" punya tombol "Sudah Pulang Semua" di header section (hanya tampil kalau ada baris, `disabled` saat piket tidak on-duty).
- [x] Dialog konfirmasi menyebutkan jumlah murid DAN peringatan eksplisit waktu pulang "tidak diketahui" + peringatan untuk cek dulu murid yang seharusnya Izin Tidak Kembali.
- [x] Eksekusi bulk memanggil `konfirmasiPulangRetroaktif()` per baris dengan `waktuPulang: undefined` — diverifikasi EKSPLISIT via test yang membaca `call[0].data.waktuPulang` harus `undefined` untuk SEMUA panggilan, bukan cuma cek hasil akhir.
- [x] Best-effort — loop `try/catch` per id, 1 gagal (mis. `ensureYesterdayRecordInKampus` throw karena `waktuPulang` sudah terisi atau bukan kampus ini) TIDAK menghentikan loop, masuk `errors[]` dengan `reason`. Dialog frontend tampilkan ringkasan berhasil/gagal + nama murid yang gagal + alasan.
- [x] Tombol "Klarifikasi" individual TETAP ada, tidak diubah/dihapus (regresi nol) — bulk murni tombol tambahan di header section yang sama.
- [x] `activityLog.record()` manual PER BARIS berhasil (bukan `@LogActivity` decorator — endpoint bulk terima banyak id sekaligus, decorator hanya cocok 1 target route param).
- [x] Build + type-check hijau (`tsc --noEmit` bersih 2 app, `nest build`+`next build` sukses), 4 test baru menutupi skenario campuran berhasil+gagal, array kosong, dan record tidak ditemukan.

## Validasi Claudian
- [x] **Verifikasi eksplisit `waktuPulang` bukan `new Date()`** — test khusus mem-baca `mock.calls` dari `prisma.attendanceRecord.update` dan assert `data.waktuPulang` adalah `undefined` untuk SEMUA baris yang diproses (bukan cuma verifikasi tidak error) — ini titik paling kritis sesuai keputusan eksplisit user, dan lolos sejak percobaan pertama karena murni REUSE `konfirmasiPulangRetroaktif(kampusId, recordId, undefined)` apa adanya tanpa modifikasi signature.
- [x] Tombol "Klarifikasi" individual (`TidakAbsenPulangForm`, termasuk opsi "Tandai Izin Keluar Tidak Kembali") **TIDAK disentuh sama sekali** — dikonfirmasi via diff, komponen itu tetap identik, hanya ditambah komponen baru `SudahPulangSemuaForm` sejajar di `Dialog` terpisah.
- [x] Requirement "waktuPulang dari jam terakhir pelajaran" **TIDAK dikerjakan di task ini**, tetap tercatat sebagai follow-up di file task T158 (bagian "Follow-up WAJIB Setelah T158 Selesai").
