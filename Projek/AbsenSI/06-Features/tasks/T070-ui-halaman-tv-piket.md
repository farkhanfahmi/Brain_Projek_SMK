# T070 — UI: Halaman TV Piket

## Depends on
T068 (auth token), T069 (API data), `DataTableCard`/`ActivityFeedCard`/`StatusBadge` (T058/T059, sudah ada di `packages/ui`)

## Objective
Buat halaman `/tv-piket/[kampusId]` — bento grid full-screen (bar persentase + 2 kolom tengah + 1 baris bawah), auto-scroll untuk daftar siswa tidak hadir, update realtime via Socket.IO.

## Context
- **App:** `apps/web`
- **Route:** `/tv-piket/[kampusId]` (BUKAN `/tv-piket` polos — 1 route per kampus, device TV dikonfigurasi eksplisit ke kampusnya via URL yang dibuka di browser kiosk mini-PC)
- **Ref:** `Projek/AbsenSI/06-Features/tv-piket.md` — bagian "Konsep" (layout ASCII final) dan "Isi/Widget" (final, 2026-07-22). **WAJIB baca ulang** `Projek/AbsenSI/06-Features/design-system/*.md` sebelum menulis UI — semua token warna/radius/spacing dari situ, JANGAN karang sendiri

## Spec Detail

### Setup Halaman
- Token TV dibaca dari `?token=...` di URL SEKALI saat load pertama, simpan ke `localStorage` (pola SAMA seperti kiosk `?device=TOKEN`, ADR-021) — reload berikutnya baca dari localStorage kalau tidak ada query param baru
- Layout **fullscreen, no navbar/sidebar** — mirip `/tv` (TV Kepsek) existing, BUKAN layout admin biasa. Cek `apps/web/app/tv/layout.tsx` existing (TV Kepsek Fase 1) sebagai referensi pola layout fullscreen, buat layout terpisah untuk `/tv-piket` dengan struktur serupa

### Struktur Bento Grid (sesuai layout ASCII final di spec)
1. **Baris atas — Bar Persentase** (full-width, 1 baris tinggi ~80-100px): 3 segmen proporsional (`flex`, `width` masing-masing sesuai `persen`), warna sesuai token final (`success` untuk hadir, `status-shipped` untuk izin/sakit, `danger` untuk alfa), angka persentase besar (`text-h3` bold, warna putih) di tengah tiap segmen
2. **Baris tengah — 2 kolom**: kiri (~60% lebar) `DataTableCard` "Siswa Tidak Hadir" (kolom: Nama, Kelas, `StatusBadge` status, Keterangan), kanan (~40% lebar) `ActivityFeedCard` "Guru Belum Mulai Mengajar"
3. **Baris bawah — full-width**: `ActivityFeedCard` "Guru Izin Hari Ini"

### Auto-Scroll (Widget Siswa Tidak Hadir)
- Kalau jumlah baris melebihi area yang terlihat, list auto-scroll ke bawah perlahan (misal `scrollTop` increment via `setInterval`/`requestAnimationFrame`, kecepatan santai — total 1 layar penuh per ~8 detik), jeda ~2 detik di posisi paling atas dan paling bawah sebelum mengulang dari atas
- TIDAK ADA kontrol manual (tidak ada tombol pause/scroll manual) — device TV tidak punya input, semua otomatis

### Realtime
- Subscribe Socket.IO channel `attendance:kampus:{kampusId}` (channel yang SAMA dipakai Dashboard Piket kerja) — begitu ada event tap/perubahan status masuk, re-fetch `GET /tv-piket/data` (bukan optimistic update parsial, karena datanya agregat lintas banyak sumber — lebih aman re-fetch utuh daripada coba hitung ulang persentase di client)
- Fallback: kalau koneksi Socket.IO putus, polling `GET /tv-piket/data` tiap 60 detik sebagai jaring pengaman (pola sama seperti kesepakatan T048 untuk halaman non-kritis-realtime — TV ini agak lebih penting jadi tetap utamakan Socket.IO, polling cuma fallback)

## JANGAN
- ❌ JANGAN buat halaman ini bisa diakses tanpa token valid — proteksi via `TvPiketGuard` di backend (T068/T069), FE juga redirect ke halaman error kalau `401` dari API
- ❌ JANGAN pakai warna di luar token yang ditentukan (`success`/`status-shipped`/`danger`) untuk bar persentase — TIDAK ada "kuning" generik, ikuti pemetaan final di `tv-piket.md`
- ❌ JANGAN buat kontrol interaktif apapun (tombol, link, form) — TV ini PASSIVE display, tidak ada elemen yang bisa "diklik" secara fungsional
- ❌ JANGAN buat auto-scroll terlalu cepat (bikin pusing dilihat sambil lewat) atau terlalu lambat (informasi lama tidak sempat terganti sebelum data refresh) — sesuaikan kecepatan supaya nyaman dibaca sekilas
- ❌ JANGAN reuse komponen custom baru untuk Data Table/Activity Feed — WAJIB pakai `DataTableCard`/`ActivityFeedCard`/`StatusBadge` generik dari `packages/ui` (T058/T059)

## Files
- **Buat:** `apps/web/app/tv-piket/[kampusId]/layout.tsx` (fullscreen, no nav)
- **Buat:** `apps/web/app/tv-piket/[kampusId]/page.tsx`
- **Buat:** `apps/web/app/tv-piket/[kampusId]/components/bar-persentase.tsx`
- **Buat:** `apps/web/lib/use-tv-piket-data.ts` — hook fetch + Socket.IO subscribe + polling fallback
- **Buat:** `apps/web/lib/use-auto-scroll.ts` — hook auto-scroll generik untuk list panjang (bisa dipakai ulang kalau ada kebutuhan serupa nanti)

## Acceptance Criteria
- [ ] Buka `/tv-piket/1?token=xxx` → halaman fullscreen tampil, token tersimpan localStorage
- [ ] Reload halaman tanpa query param → tetap berfungsi (token dari localStorage)
- [ ] Bar persentase menampilkan 3 segmen proporsional dengan warna & angka sesuai data API
- [ ] Daftar siswa tidak hadir dengan >10 baris → auto-scroll berjalan tanpa interaksi
- [ ] Tap siswa baru di kiosk kampus tsb → TV update dalam beberapa detik (verifikasi via Playwright: 2 browser context, satu simulasi tap siswa via API, satu observer di halaman TV)
- [ ] Akses `/tv-piket/1` tanpa token / dengan token yang sudah di-revoke → halaman error jelas, bukan crash/blank

## Handoff
Setelah T068-T070 selesai, TV Piket siap dipasang di lorong — device mini-PC/TV browser dikonfigurasi 1x dengan URL `/tv-piket/{kampusId}?token={token dari T068}`, tidak perlu re-konfigurasi lagi kecuali token di-revoke.
