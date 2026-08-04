# T080 — API+UI: Pagination untuk Tabel Data Besar (Siswa, Kartu, Foto, Log)

## Depends on
Tidak ada — perubahan independen di beberapa endpoint+halaman, bisa dikerjakan bertahap per halaman.

## Context
- **App:** `apps/api` + `apps/web`
- **Ref:** Ditemukan 2026-07-24 — dengan 4090 siswa & 3929 kartu ter-migrasi (T062), halaman `/siswa`, `/kartu`, `/foto` mem-fetch dan render SEMUA baris sekaligus tanpa batas. Ini memberatkan render browser (DOM node ribuan baris) meski response API sudah cepat.
- **Catatan performa terpisah:** masalah kecepatan navigasi menu (2-6 detik) yang dilaporkan user sebelumnya adalah soal **mode dev vs production** (sudah diselesaikan terpisah, lihat sesi 2026-07-24) — task ini murni soal jumlah DOM node/baris tabel, bukan solusi yang sama.

## Spec Detail

### Pola yang Sudah Ada (Preseden)
`(admin)/log/log-view.tsx` **SUDAH** pagination server-side (lihat `goToPage()`, `ActivityLogPage` type dengan field halaman). **REUSE pola ini** — jangan desain ulang dari nol.

### Endpoint yang Perlu Pagination
1. `GET /students` (`students.controller.ts` + `students.service.ts`) — tambah `page`, `pageSize` (default 50) ke `ListStudentsDto`, response jadi `{ data: Student[], total: number, page: number, pageSize: number }` (pola sama seperti `ActivityLogPage`)
2. `GET /cards` — sama, tambah pagination
3. Endpoint foto (`GET /students` + `GET /teachers` dipakai bareng di `/foto` — cek ulang, foto-view fetch keduanya) — cukup pagination di listing utamanya, TIDAK perlu untuk assign-picker dropdown

### UI
- `siswa-view.tsx`, `kartu-view.tsx` (setelah T078 split section), `foto-view.tsx` (setelah T079 split section) — tambah komponen pagination di bawah tabel: tombol Previous/Next + info "Halaman X dari Y" + total data, pola visual sama seperti yang sudah ada di `log-view.tsx`
- **PENTING:** pagination HARUS terintegrasi dengan filter (T078/T079/filter Kelas-Jurusan yang sudah ada) — ganti filter otomatis reset ke halaman 1, bukan tetap di halaman terakhir yang mungkin sudah kosong
- Default `pageSize`: 50 baris per halaman (cukup ringan untuk render, tidak terlalu banyak klik next untuk 4000+ data)

### JANGAN dari Bug T-sebelumnya (WAJIB diperhatikan)
- Filter server-side di `siswa-view.tsx` baru saja diperbaiki (2026-07-24) karena bug `useState(initialStudents)` tidak sinkron ulang saat props berubah + Next.js Router Cache butuh `router.refresh()` eksplisit setelah `router.push()`. **Terapkan pola yang SAMA untuk state pagination** — kalau pagination state juga disimpan di `useState` lokal yang di-derive dari props, WAJIB ada `useEffect` sync + `router.refresh()` di handler ganti halaman, atau bug yang sama akan muncul lagi (halaman berubah di URL tapi tabel tidak update).

## JANGAN
- ❌ JANGAN infinite scroll — user tidak minta ini, pagination klik Next/Previous eksplisit sesuai preseden `log-view.tsx`
- ❌ JANGAN pagination di client-side (`.slice()` dari array yang sudah di-fetch penuh) — data 4000+ baris TETAP harus di-fetch sesuai halaman dari server, bukan fetch semua lalu potong di browser (itu tidak menyelesaikan masalah beban)
- ❌ JANGAN ubah endpoint yang sudah dipakai fitur lain tanpa cek dulu — misal `GET /students` dipakai juga oleh `plot-siswa`, `kenaikan-massal`, dropdown assign foto, dll yang butuh SEMUA data tanpa pagination. Tambahkan pagination sebagai PARAMETER OPSIONAL (kalau `page` tidak dikirim, tetap return semua seperti sekarang) — JANGAN break existing callers

## Files
- **Modifikasi:** `apps/api/src/core/students/dto/list-students.dto.ts`, `students.service.ts`, `students.controller.ts`
- **Modifikasi:** `apps/api/src/core/cards/` (cek nama file controller/service/dto yang sesuai)
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/siswa-view.tsx`, `siswa/page.tsx`
- **Modifikasi:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (setelah/bareng T078)
- **Modifikasi:** `apps/web/src/app/(admin)/foto/foto-view.tsx` (setelah/bareng T079)
- **Reuse:** komponen pagination dari `log-view.tsx` — extract jadi komponen shared di `packages/ui` kalau dipakai di ≥3 tempat (DRY, tapi jangan over-engineer kalau ternyata beda kebutuhan)

## Acceptance Criteria
- [ ] Halaman `/siswa` dengan 4090 data hanya render ~50 baris per halaman, ada navigasi halaman
- [ ] Ganti filter (Kelas/Jurusan/Status) otomatis reset ke halaman 1
- [ ] Endpoint `GET /students` TANPA parameter `page` tetap return semua data (tidak breaking existing callers seperti plot-siswa)
- [ ] Halaman Kartu dan Foto (setelah split section T078/T079) juga punya pagination per section
- [ ] `tsc` + `jest` bersih, `pnpm turbo run build` bersih
