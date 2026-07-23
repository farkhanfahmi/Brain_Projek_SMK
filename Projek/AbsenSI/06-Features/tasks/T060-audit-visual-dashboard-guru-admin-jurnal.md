# T060 — Audit Visual Nyata: Dashboard Guru & Admin Jurnal (Playwright)

## Depends on
Kode T038-T053 sudah dieksekusi (backend + frontend) — **sudah ada di working tree saat task ini ditulis (2026-07-22), belum ter-commit**. Cek `git status` dulu untuk konfirmasi state terkini sebelum audit.

## Objective
Jalankan aplikasi nyata (`apps/web` + `apps/api`), buka setiap halaman dashboard guru dan admin_jurnal yang sudah dikodekan, screenshot dan verifikasi visual — karena sepanjang breakdown T038-T053 belum pernah ada verifikasi visual nyata (spec ditulis, kode dieksekusi sesi lain, tapi belum pernah dilihat langsung berjalan di browser).

## Context
- **App:** `apps/web`, `apps/api` (harus running)
- **Tool:** Playwright MCP (sudah terpasang, lihat `claude mcp list`)
- **Ref:** `Projek/AbsenSI/06-Features/design-system/*.md` (WAJIB baca ulang versi terkini sebelum audit — sudah direvisi 2026-07-22 dengan token status baru dari T057)

## Spec Detail

### Halaman yang WAJIB diaudit (berdasarkan kode yang sudah ada di working tree)

**Dashboard Guru:**
- `/guru` atau `/guru/jurnal` (landing, T048) — cek badge status tap gerbang, badge status sesi (Belum Waktunya/Bisa Dimulai/Sedang Berlangsung/Selesai/Diizinkan), tombol Mulai Mengajar/Izin
- `/guru/sesi/:sessionId` (T049) — form jurnal autosave, tabel presensi siswa, badge Hadir/Tidak Ada di Kelas
- `/guru/izin/:permitId` (T049 bagian kedua) — form tugas titipan
- `/guru/wali-kelas` (T053) — 3 tab: Ringkasan Kehadiran, Rekap Mapel, Catatan

**Dashboard Admin Jurnal:**
- `/admin-jurnal/toleransi` (T050 revisi) — form toleransi keterlambatan
- `/admin-jurnal/mapel` (T050) — CRUD mapel
- `/admin-jurnal/semester` (T054/T050) — tabel read-only semester
- `/admin-jurnal/jadwal-blok` (T056) — kalender visual minggu A/B, coverage indicator
- `/admin-jurnal/jadwal` (T050/T047) — tabel jadwal mengajar, tombol Salin dari Semester Sebelumnya
- `/admin-jurnal/izin` (T051) — form input izin + tabel riwayat + upload bukti
- `/admin-jurnal/wali-kelas` (T052) — assign wali kelas per kelas

### Untuk tiap halaman, verifikasi (pakai Playwright: `browser_navigate`, `browser_snapshot`/`browser_take_screenshot`, `browser_click`, dll):

1. **Kepatuhan token desain** — screenshot, bandingkan visual terhadap `01-colors.md`/`03-components.md`: radius card 24px, warna hanya oranye/success/danger (kecuali status-shipped/processing di tabel workflow kalau relevan), tidak ada emoji sebagai ikon, font Plus Jakarta Sans/Inter termuat
2. **Interaksi nyata** — klik tombol, isi form, submit — pastikan behavior sesuai acceptance criteria di task spec asalnya (T048-T056), bukan cuma tampilan diam
3. **Kasus edge visual** — state kosong (belum ada data), state error (validasi gagal), state loading — pastikan semuanya punya tampilan yang jelas, bukan blank/crash

### Setelah audit, jalankan detector otomatis Impeccable
```
node <impeccable-skill-base-dir>/scripts/detect.mjs --json apps/web/src/app/(guru) apps/web/src/app/(admin-jurnal)
```
Laporkan temuan detector (jangan diabaikan) sebagai tambahan ke temuan visual manual.

## JANGAN
- ❌ JANGAN anggap task "selesai" hanya dari membaca kode — WAJIB benar-benar navigasi & screenshot tiap halaman lewat Playwright, sesuai kelemahan yang sudah diakui sebelumnya ("kode yang terlihat benar" vs "terbukti bekerja")
- ❌ JANGAN perbaiki temuan langsung tanpa laporan dulu — task ini AUDIT (temukan & laporkan), perbaikan konkret jadi task/commit terpisah supaya user bisa review temuan dulu sebelum ada perubahan kode
- ❌ JANGAN skip halaman manapun di daftar atas dengan alasan "kemungkinan besar sudah benar" — precisely karena belum pernah diverifikasi visual, semua halaman punya risiko yang sama sampai terbukti sebaliknya

## Files
- Tidak ada file yang dimodifikasi di task ini (murni audit) — kecuali temuan disepakati untuk langsung diperbaiki, itu jadi task/commit terpisah

## Acceptance Criteria
- [ ] Semua 11 halaman di atas berhasil di-screenshot dalam kondisi running nyata (bukan cuma baca kode)
- [ ] Laporan temuan tertulis: per halaman, apa yang cocok dengan design system dan apa yang tidak (kalau ada)
- [ ] Detector Impeccable dijalankan, hasil JSON-nya dilampirkan/dirangkum di laporan
- [ ] Kalau ada temuan pelanggaran design system (radius salah, warna di luar token, dll) — dicatat sebagai temuan terpisah dengan file:line, BUKAN langsung diperbaiki di task ini

## Handoff
Temuan dari task ini jadi dasar task perbaikan lanjutan (bisa dinomori T061+ sesuai kebutuhan) — hanya dibuat KALAU ada temuan nyata, jangan buat task perbaikan spekulatif sebelum audit ini benar-benar jalan.
