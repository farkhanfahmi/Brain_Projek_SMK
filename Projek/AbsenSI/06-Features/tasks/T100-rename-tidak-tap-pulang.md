# T100 — Rename Menyeluruh "Tidak Tap Pulang" → "Tidak Absen Pulang"

## Depends on
Tidak ada — murni rename (teks + identifier kode), TIDAK mengubah logic/behavior sama sekali.

## Objective
Ganti semua penyebutan "Tidak Tap Pulang" (label UI, komentar, nama variabel/fungsi/tipe/endpoint) menjadi "Tidak Absen Pulang" secara KONSISTEN di seluruh kode — dikonfirmasi user 2026-07-30, cakupan MENYELURUH (bukan cuma teks UI), termasuk identifier internal.

## Context
- **App:** `apps/api` + `apps/web`
- Diskusi 2026-07-30: user awalnya mengira "banyak siswa tidak tap pulang" adalah bug soal parameter jam pulang — sudah diverifikasi lewat data live BUKAN bug (mekanisme tap murni berbasis urutan/count, tidak ada logic jam sama sekali; 44% siswa pada 2026-07-29 memang secara fisik tidak tap kedua kalinya, dikonfirmasi lewat cross-check `tap_events`). Diskusi berlanjut ke ide besar "jam pulang harus sesuai jadwal kelas" — **DITUNDA**, dicatat terpisah di T101 (06-Features/tasks/T101-validasi-jam-pulang-jadwal-kelas.md) (blocked, butuh data jadwal per kelas yang belum diisi). **T100 ini HANYA rename, TIDAK ADA perubahan logic.**

## Kapan Siswa Masuk Kategori Ini (referensi, TIDAK berubah oleh task ini)
Dari `attendance.service.ts` method `tidakTapPulangKemarin()` (akan di-rename, lihat di bawah): siswa masuk daftar ini kalau (1) `tanggal` = KEMARIN persis, (2) `waktuPulang` masih null (tidak pernah tap kedua kalinya sama sekali), (3) `studentId` terisi (bukan guru), (4) kelasnya di kampus yang sama dengan piket yang login. Ini query LIVE (dihitung ulang tiap dashboard dibuka), bukan dari job terjadwal.

## Spec Detail — Semua Titik yang Perlu Diubah

### Backend (`apps/api`)
- `apps/api/src/attendance/attendance.service.ts`:
  - Baris ±318 (komentar) — "Antrian Klarifikasi 'Tidak Tap Pulang' (T025)" → "Antrian Klarifikasi 'Tidak Absen Pulang'"
  - Baris ±323 — method `tidakTapPulangKemarin(kampusId)` → `tidakAbsenPulangKemarin(kampusId)`
  - Baris ±348 (komentar) — referensi ke method lama, update
  - Baris ±349 — method `countTidakTapPulangKemarin(kampusId)` → `countTidakAbsenPulangKemarin(kampusId)`
- `apps/api/src/attendance/attendance.controller.ts`:
  - Baris ±126 (komentar) — "Tidak Tap Pulang Kemarin" → "Tidak Absen Pulang Kemarin"
  - Baris ±134 — destructure `tidakTapPulang` → `tidakAbsenPulang`
  - Baris ±136 — panggilan `countTidakTapPulangKemarin` → `countTidakAbsenPulangKemarin`
  - Baris ±140 — response key `tidakTapPulang` → `tidakAbsenPulang` (**PERHATIAN**: ini response JSON yang dikonsumsi frontend — cek semua caller sebelum ubah key, lihat `use-piket-notification-counts.ts` di bawah)
  - Baris ±173 (komentar) — update
  - Baris ±174 — route `@Get("tidak-tap-pulang-kemarin")` → `@Get("tidak-absen-pulang-kemarin")` (**PERHATIAN**: perubahan URL endpoint, cek semua caller frontend sebelum ubah — lihat `piket/page.tsx` di bawah)
  - Baris ±177-178 — nama method controller `tidakTapPulangKemarin` → `tidakAbsenPulangKemarin`
- `apps/api/src/permits/permits.controller.ts`:
  - Baris ±63 (komentar) — "Resolusi antrian klarifikasi 'Tidak Tap Pulang' (T025)" → "Resolusi antrian klarifikasi 'Tidak Absen Pulang'"

### Frontend (`apps/web`)
- `apps/web/src/lib/core-types.ts`:
  - Baris ±481 — interface `TidakTapPulangRow` → `TidakAbsenPulangRow` (cek SEMUA pemakai tipe ini sebelum rename, minimal `piket-board-view.tsx` dan `piket/page.tsx`)
- `apps/web/src/lib/use-piket-notification-counts.ts`:
  - Baris ±8 — field `tidakTapPulang: number` → `tidakAbsenPulang: number` (SESUAIKAN dengan perubahan response key backend di atas — kedua sisi harus berubah bersamaan, ini bukan 2 perubahan independen)
- `apps/web/src/app/(piket)/piket/page.tsx`:
  - Baris ±7 — import tipe `TidakTapPulangRow` → `TidakAbsenPulangRow`
  - Baris ±13 — destructure `tidakTapPulang` → `tidakAbsenPulang`
  - Baris ±16 — panggilan `apiFetch<TidakAbsenPulangRow[]>("/attendance/tidak-absen-pulang-kemarin")` (URL ikut berubah sesuai backend)
  - Baris ±26 — prop `initialTidakTapPulang` → `initialTidakAbsenPulang`
- `apps/web/src/app/(piket)/piket/piket-board-view.tsx`:
  - Import tipe `TidakTapPulangRow` → `TidakAbsenPulangRow`
  - Prop `initialTidakTapPulang`, state `tidakTapPulang`/`setTidakTapPulang` → `initialTidakAbsenPulang`, `tidakAbsenPulang`/`setTidakAbsenPulang`
  - `openSection` union type value `"tidak-tap-pulang"` → `"tidak-absen-pulang"` (dan semua pemakaiannya: `SummaryCard` `onClick`/`active`, kondisi render section)
  - Fungsi `handleTidakTapPulangResolved` → `handleTidakAbsenPulangResolved`
  - **Label UI** `SummaryCard label="Tidak Tap Pulang"` → `label="Tidak Absen Pulang"`
  - Komponen `TidakTapPulangSection` → `TidakAbsenPulangSection`, judul di dalamnya `<h2>Tidak Tap Pulang Kemarin</h2>` → `<h2>Tidak Absen Pulang Kemarin</h2>`
  - Komponen `TidakTapPulangForm` → `TidakAbsenPulangForm` (dialog "Klarifikasi" — isi form TIDAK berubah, cuma nama komponen; teks "Konfirmasi Pulang Normal"/"Tandai Izin Keluar Tidak Kembali" di dalamnya TIDAK perlu diubah, itu bukan bagian dari rename "tidak tap/absen pulang")

## Business Rules
- **TIDAK ADA perubahan logic/query/endpoint behavior** — murni rename identifier dan teks. Response shape JSON boleh berubah KEY-nya (`tidakTapPulang` → `tidakAbsenPulang`) tapi struktur/isi datanya identik.
- Endpoint route berubah (`/attendance/tidak-tap-pulang-kemarin` → `/attendance/tidak-absen-pulang-kemarin`) — pastikan TIDAK ada 1 sisi yang lupa diupdate (backend jalan duluan, frontend masih panggil URL lama = akan 404). Ubah backend+frontend dalam 1 commit/deploy yang sama, JANGAN deploy terpisah.

## Files
- **Modifikasi:** semua file yang disebutkan di atas (7 file total): `apps/api/src/attendance/attendance.service.ts`, `apps/api/src/attendance/attendance.controller.ts`, `apps/api/src/permits/permits.controller.ts`, `apps/web/src/lib/core-types.ts`, `apps/web/src/lib/use-piket-notification-counts.ts`, `apps/web/src/app/(piket)/piket/page.tsx`, `apps/web/src/app/(piket)/piket/piket-board-view.tsx`.
- **Jangan sentuh:** logic `tidakAbsenPulangKemarin()`/`countTidakAbsenPulangKemarin()` itu sendiri (where clause, filter tanggal/waktuPulang) — HANYA nama, bukan isi fungsi. Jangan sentuh section "Belum Kembali"/"Perlu Ditinjau"/"Siswa Terkunci" — di luar scope rename ini.

## Acceptance Criteria
- [x] Grep ulang `Tidak Tap Pulang\|tidakTapPulang\|TidakTapPulang\|tidak-tap-pulang` di seluruh `apps/api/src` dan `apps/web/src` setelah selesai — **0 hasil**, dikonfirmasi. Tidak ada `.spec.ts` yang menyentuh nama-nama ini (hanya `attendance-report.service.spec.ts` ada di modul yang sama, tapi untuk service lain, tidak tersentuh).
- [x] Dashboard piket menampilkan label "Tidak Absen Pulang" (kartu ringkas + judul section "Tidak Absen Pulang Kemarin") — diverifikasi live via Playwright dengan akun `guru_piket` disposable, screenshot dicek manual.
- [x] Endpoint lama (`/attendance/tidak-tap-pulang-kemarin`) sudah tidak ada (404 dikonfirmasi live curl), endpoint baru (`/attendance/tidak-absen-pulang-kemarin`) berfungsi identik — return 182 baris data (sama seperti sebelum rename, hanya nama endpoint yang berubah).
- [x] Badge notifikasi (`piket-notification-counts`) — key baru `tidakAbsenPulang` dikonfirmasi via curl langsung: `{"belumKembali":12,"tidakAbsenPulang":182,"perluDitinjau":7,"terkunci":6}`. (Catatan: hook `usePiketNotificationCounts` sendiri ternyata belum dikonsumsi di UI manapun saat ini — didefinisikan tapi tidak di-import, di luar scope T100 untuk menyambungkannya.)
- [x] Build + type-check `apps/api` dan `apps/web` hijau (`tsc --noEmit` bersih kedua app, `nest build` + `next build` sukses).
- [x] Jest `apps/api` 183/183 tetap lulus (tidak ada test yang menyentuh nama-nama yang di-rename).

**Implementasi:** rename mekanis di 7 file persis sesuai spec — `attendance.service.ts` (`tidakTapPulangKemarin`→`tidakAbsenPulangKemarin`, `countTidakTapPulangKemarin`→`countTidakAbsenPulangKemarin`, komentar), `attendance.controller.ts` (route `@Get("tidak-tap-pulang-kemarin")`→`@Get("tidak-absen-pulang-kemarin")`, response key `tidakTapPulang`→`tidakAbsenPulang`, nama method), `permits.controller.ts` (komentar saja), `core-types.ts` (`TidakTapPulangRow`→`TidakAbsenPulangRow`), `use-piket-notification-counts.ts` (field `tidakTapPulang`→`tidakAbsenPulang`), `piket/page.tsx` (import tipe, destructure, URL fetch, prop), `piket-board-view.tsx` (tipe, prop/state, union type `openSection`, nama fungsi handler, label UI, nama 2 komponen `TidakTapPulangSection`/`TidakTapPulangForm`→`TidakAbsenPulangSection`/`TidakAbsenPulangForm`).

**Verifikasi live:** backend+frontend di-restart BERSAMAAN (sesuai warning spec soal endpoint yang berubah). Dibuat akun `guru_piket` disposable (`t100_verify_piket`, dinonaktifkan lagi setelah verifikasi — FK `activity_log.actor_id` mencegah hard delete). Login via Playwright ke `/piket`, kartu "Tidak Absen Pulang" tampil dengan count 182 (cocok dengan count API), klik kartu membuka section "Tidak Absen Pulang Kemarin" dengan tabel data ter-render benar.

## Validasi Claudian
- [ ] Ini MURNI rename — kalau saat eksekusi ternyata tergoda "sekalian perbaiki juga logic X", STOP, itu bukan scope T100 (lihat T101 (06-Features/tasks/T101-validasi-jam-pulang-jadwal-kelas.md) untuk perubahan logic yang didiskusikan tapi sengaja ditunda).
- [ ] Cek dulu apakah ada file `.spec.ts` yang mock/assert nama-nama ini sebelum menganggap rename selesai — test yang menyentuh nama lama akan gagal kalau tidak ikut di-update.
- [ ] Backend+frontend WAJIB deploy bersamaan (bukan 2 langkah terpisah) karena perubahan endpoint route.
