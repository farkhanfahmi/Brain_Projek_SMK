# T179 — API+Web: Tombol "Eksekusi Semua" per Section Dashboard Piket

## Depends on
Tidak ada dependency teknis. Independen. **T180 (bulk "Sudah Pulang" khusus section Tidak Absen Pulang) adalah KASUS KHUSUS dari task ini** — REKOMENDASI kerjakan T179 dulu (infrastruktur bulk generic per section), T180 menyusul sebagai penerapan spesifik section itu dengan requirement tambahan (waktuPulang).

## Objective
Setiap section di dashboard piket (Board Semua Siswa, Belum Kembali, Tidak Absen Pulang, Perlu Ditinjau, Terkunci) mendapat tombol **"Eksekusi Semua"** yang menjalankan aksi utama section itu untuk SEMUA baris yang sedang tampil sekaligus — bukan klik satu-satu.

## Context — Temuan Riset (2026-08-14)

`apps/web/src/app/(piket)/piket/piket-board-view.tsx` — **5 section terkonfirmasi**, masing-masing dengan aksi individual dan endpoint SAAT INI HANYA single-id (TIDAK ADA satupun endpoint bulk existing):

| Section | Aksi individual sekarang | Endpoint (single-id) |
|---|---|---|
| Board Semua Siswa | Izin/Sakit, Cetak Surat Masuk (terlambat) | `POST /permits`, `POST /late-entry-slips` |
| Belum Kembali | Sudah Kembali, Dianggap Pulang | `PATCH /permits/:id/confirm-kembali`, `PATCH /permits/:id/set-pulang` |
| Tidak Absen Pulang | Klarifikasi (dialog 2 sub-aksi) | `POST /attendance/:record_id/konfirmasi-pulang-retroaktif`, `POST /permits/tandai-izin-tidak-kembali` |
| Perlu Ditinjau | Kunci Murid | `POST /students/:id/lock` |
| Murid Terkunci | Buka Kunci | `POST /students/:id/unlock` |

**Section "Board Semua Siswa"** — aksinya (Izin/Sakit, Cetak Surat) BUKAN aksi seragam 1-tombol (perlu isi alasan/kategori per siswa) — **TIDAK COCOK untuk bulk sederhana**, KECUALIKAN dari scope task ini (LAPORKAN sebagai temuan, bukan diam-diam skip). Task ini fokus ke **4 section** yang aksinya benar-benar 1-klik-seragam: Belum Kembali (Sudah Kembali), Tidak Absen Pulang (lihat T180 terpisah, ADA syarat khusus soal waktuPulang), Perlu Ditinjau (Kunci), Murid Terkunci (Buka Kunci).

## Spec Detail

### 1. Backend — endpoint bulk baru per aksi (menerima array ID)

Untuk SETIAP endpoint di tabel atas (KECUALI 2 endpoint section "Tidak Absen Pulang" yang scope-nya di T180) — tambah versi BULK:

- `PATCH /permits/confirm-kembali-bulk` — body `{permitIds: number[]}` — service loop panggil `confirmKembali()` PER id DALAM TRANSAKSI (atau `Promise.all` kalau tidak butuh all-or-nothing — PUTUSKAN saat implementasi: REKOMENDASI **best-effort** bukan all-or-nothing, karena 1 baris gagal (misal race condition data sudah berubah) TIDAK BOLEH menggagalkan baris lain yang valid — return `{successCount, failedCount, errors: [{permitId, reason}]}` KONSISTEN pola `ImportReport` yang sudah ada di proyek).
- `POST /students/lock-bulk` — body `{studentIds: number[], alasan?: string}` (kalau `lock()` single butuh alasan, cek dulu signature-nya) — SAMA pola best-effort+report.
- `POST /students/unlock-bulk` — body `{studentIds: number[]}` — SAMA pola.
- Guard SAMA seperti endpoint single (`PiketOnDutyGuard` dkk, TIDAK diubah) — REUSE, JANGAN buat guard baru.
- `@LogActivity` — WAJIB tercatat PER BARIS yang berhasil (bukan 1 log untuk seluruh bulk — supaya riwayat/audit tetap presisi per siswa, KONSISTEN prinsip granularitas activity_log yang sudah ada di proyek).

### 2. Frontend — tombol "Eksekusi Semua" per section

- `piket-board-view.tsx` — di HEADER masing-masing dari 3 section (Belum Kembali, Perlu Ditinjau, Murid Terkunci — Tidak Absen Pulang di T180 terpisah) — tambah tombol "Eksekusi Semua" (label spesifik per section: "Konfirmasi Semua Sudah Kembali", "Kunci Semua", "Buka Kunci Semua" — JANGAN generic "Eksekusi Semua" di UI, itu cuma nama task, LABEL HARUS JELAS aksinya apa).
- Klik tombol → **WAJIB dialog konfirmasi** (aksi bulk berdampak besar, KONSISTEN pola dialog konfirmasi yang sudah ada untuk aksi tunggal beresiko seperti Revoke Kartu) — sebutkan JUMLAH baris yang akan terpengaruh ("Konfirmasi X siswa sudah kembali?").
- Setelah eksekusi — tampilkan **hasil ringkas** (berapa berhasil, berapa gagal + alasan kalau ada) — JANGAN silent success/fail, admin perlu tahu kalau ada baris yang gagal diproses.
- Refresh section itu setelah selesai (REUSE pola refresh existing di board — Socket.IO update ATAU manual refetch, konsisten yang sudah ada).
- Section KOSONG (tidak ada baris) — tombol "Eksekusi Semua" DISABLED/TIDAK TAMPIL (tidak ada gunanya).

## Edge Cases
- Section dengan RATUSAN baris (misal Perlu Ditinjau saat insiden lock massal, ADR terkait insiden lama) — endpoint bulk backend HARUS efisien (batch query, BUKAN N+1 query terpisah per id kalau bisa dihindari — PERTIMBANGKAN `updateMany` untuk bagian yang bisa, tapi HATI-HATI beberapa aksi (`confirmKembali`, `lock`) punya SIDE EFFECT lain (log, cek kondisi) yang TIDAK BISA murni `updateMany` — VERIFIKASI per endpoint apakah bisa dioptimalkan atau tetap loop, JANGAN korbankan correctness demi performa).
- Race condition: 2 piket klik "Eksekusi Semua" section yang sama nyaris bersamaan — best-effort report akan secara alami menangani ini (baris yang sudah diproses piket A akan gagal/skip saat diproses piket B, tercatat di report), TIDAK PERLU locking tambahan di luar itu.
- Section "Board Semua Siswa" — DIKECUALIKAN dari task ini (dijelaskan di Context), JANGAN tambahkan tombol bulk di sana tanpa desain terpisah untuk alasan per-siswa.

## Files
- **Modifikasi:** `apps/api/src/permits/permits.controller.ts`+`permits.service.ts` (endpoint bulk confirm-kembali), `apps/api/src/core/students/students.controller.ts`+`students.service.ts` (endpoint bulk lock/unlock), `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (3 tombol bulk baru + dialog konfirmasi + hasil ringkas).
- **Jangan sentuh:** endpoint single-id existing (TIDAK dihapus/diubah, bulk adalah TAMBAHAN paralel), section "Board Semua Siswa" (dikecualikan), section "Tidak Absen Pulang" (scope T180 terpisah).

## Acceptance Criteria
- [x] Section Belum Kembali ("Konfirmasi Semua Sudah Kembali"), Perlu Ditinjau ("Kunci Semua"), Murid Terkunci ("Buka Kunci Semua") masing-masing punya tombol bulk dengan label spesifik aksinya (bukan "Eksekusi Semua" generic di UI).
- [x] Klik tombol menampilkan dialog konfirmasi (`BulkActionDialog` generik reusable) dengan jumlah murid terpengaruh + field ekstra wajib untuk Kunci/Buka Kunci (alasan/catatan, sama untuk semua baris dalam 1 eksekusi).
- [x] Eksekusi bulk best-effort — loop `try/catch` per id di service, 1 baris gagal (race lawan aksi individual/piket lain) TIDAK menghentikan loop.
- [x] Hasil ringkas (berhasil/gagal + nama murid + alasan per baris gagal) ditampilkan di dialog setelah eksekusi.
- [x] `activityLog.record()` manual PER BARIS berhasil (bukan `@LogActivity` decorator — endpoint bulk terima banyak id, decorator hanya cocok 1 target route param) di ketiga service (`PermitsService`, `StudentsService`).
- [x] Section kosong — tombol bulk tidak tampil sama sekali (`items.length > 0 &&`).
- [x] Section "Board Semua Siswa" TIDAK dapat tombol bulk — dikecualikan sesuai spec (aksinya Izin/Sakit/Cetak Surat butuh input per-siswa, bukan aksi 1-klik seragam).
- [x] Build + type-check hijau (`tsc --noEmit` bersih 2 app, `nest build` sukses, `next build` sukses — sempat terganggu proses `next dev` sesi lain yang menulis folder `.next` bersamaan, diverifikasi ulang setelah sesi itu selesai). 12 test baru (`permits.service.spec.ts` baru + `students.service.spec.ts` baru) untuk skenario campuran berhasil+gagal, best-effort, array kosong.

## Validasi Claudian
- [x] **Endpoint bulk best-effort dikonfirmasi** (bukan all-or-nothing) — sesuai rekomendasi task, tidak ada penyimpangan.
- [x] **`@LogActivity` granularitas PER BARIS dikonfirmasi** — `PermitsService`/`StudentsService` masing-masing di-inject `ActivityLogService` (baru untuk `PermitsService`, `PermitsModule` ditambah `ActivityLogModule`), `activityLog.record()` dipanggil dalam loop, 1 kali per baris yang berhasil, action-name SAMA dengan endpoint single (`permit.confirm_kembali`, `student.lock`, `student.unlock`).
- [x] **Section "Board Semua Siswa" dikecualikan secara sengaja** (bukan lupa) — dilaporkan di ringkasan implementasi: aksinya (Izin/Sakit butuh alasan per-siswa, Cetak Surat butuh data per-siswa) bukan aksi seragam 1-klik yang cocok untuk pola bulk generic ini, sesuai keputusan eksplisit di Context task.
- [x] **Section "Tidak Absen Pulang" TIDAK disentuh di task ini** — sudah dikerjakan terpisah di T180 sebelum task ini (dengan syarat tambahan `waktuPulang` khusus).
- [x] **Komponen `BulkActionDialog` generik** dibuat untuk menghindari duplikasi 3x kode dialog konfirmasi+eksekusi+hasil-ringkas yang identik strukturnya, dipakai oleh ketiga section — TIDAK dipakai untuk T180 (`SudahPulangSemuaForm`) yang sudah selesai lebih dulu dan dibiarkan apa adanya (task ini tidak diminta refactor T180).
