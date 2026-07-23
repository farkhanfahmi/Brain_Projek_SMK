# T048 — UI: Dashboard Guru — Halaman Utama (Jadwal Hari Ini)

## Depends on
T041 (API jadwal hari ini), T046 (API izin guru — untuk status tombol Izin)

## Objective
Buat halaman utama dashboard guru yang menampilkan jadwal mengajar hari ini, dengan 2 tombol per slot ("Mulai Mengajar" / "Izin") yang aktif/nonaktif sesuai gating dari backend.

## Context
- **App:** `apps/web`
- **Route:** `/guru` (halaman utama guru setelah login — **beda** dari `/riwayat` yang sudah ada di Fase 1/T021, ini halaman BARU yang jadi landing page guru, T021 tetap ada sebagai sub-halaman)
- **Role:** `guru`
- **Ref:** `Projek/AbsenSI/06-Features/dashboard-guru-jurnal.md` — bagian "Konsep Inti" poin 1-5
- **⚠️ Ref WAJIB dibaca sebelum menulis 1 baris kode UI:** `Projek/AbsenSI/06-Features/design-system/MASTER.md` dan seluruh companion file (`01-colors.md`, `02-typography.md`, `03-components.md`, `04-layout-spacing.md`) — **semua warna, radius, spacing, tipografi di task ini WAJIB pakai token dari situ, JANGAN karang warna/ukuran sendiri.** Detail token yang relevan sudah dipetakan di bagian "Konten halaman" di bawah, tapi kalau ada pattern yang tidak tercakup di sini, rujuk balik ke design system, jangan pakai default Tailwind/shadcn generik.

## Spec Detail

### Layout `/guru`
- Extend `apps/web/app/(guru)/layout.tsx` yang sudah ada dari T021 — tambah menu baru "Jurnal Mengajar" di sidebar guru (link ke `/guru`), menu "Riwayat Kehadiran" (T021) tetap ada
- Sidebar & top bar HARUS mengikuti spec existing (`03-components.md` bagian Sidebar & Top Bar) — kalau `(guru)/layout.tsx` dari T021 sudah dibangun sesuai design system, task ini HANYA menambah 1 nav item baru dengan pola yang identik (ikon 20px stroke `--color-text-secondary`, radius 16px, active state full orange `--color-primary` + label putih) — jangan buat gaya nav item baru yang beda dari item lain di sidebar yang sama
- `/guru` jadi default redirect setelah login untuk role `guru` (bukan langsung ke `/riwayat` seperti sebelumnya — cek routing existing dan update default redirect)

### Konten halaman
- Header: `text-h1` (24px/bold, `--color-text-primary`) "Jadwal Mengajar Hari Ini — [tanggal]" + badge status tap gerbang di sampingnya, pakai spec **Badge/Delta Pill** dari `03-components.md` (`radius-full`, padding 2px 8px, `text-caption` 12px/600):
  - Sudah tap gerbang → `--color-success-bg` bg + `--color-success-text` text, label "Sudah Tap Gerbang"
  - Belum tap gerbang → `--color-danger-bg` bg + `--color-danger-text` text, label "Belum Tap Gerbang" (bukan kuning — sistem token ini cuma punya 2 warna status: success/danger, TIDAK ada warna kuning/warning terpisah, lihat `01-colors.md`)
- List card per sesi (urut jam) — tiap card pakai **Global Card** spec (`03-components.md`): `radius-xl` (24px), padding 24px, `shadow-card`, bg `--color-bg-surface` putih, gap antar card 24px (`space-6`)
  - Nama kelas + mapel + jam: kelas+mapel di `text-h3` (16px/600 `--color-text-primary`), jam di `text-body` (14px/400 `--color-text-secondary`) — misal "XI-RPL-1 — Pemrograman Web" lalu baris kedua "07:00-08:30"
  - Badge status sesi — pakai Badge/Delta Pill yang sama, dengan mapping warna **hanya dari palet yang tersedia** (tidak ada token biru/abu terpisah untuk status, jadi gunakan kombinasi yang konsisten):
    - "Belum Waktunya" → bg `--color-bg-surface-subtle`, text `--color-text-secondary` (netral, bukan status positif/negatif)
    - "Bisa Dimulai" → `--color-success-bg`/`--color-success-text` (status positif — guru bisa bertindak)
    - "Sedang Berlangsung" → `--color-primary-soft` bg + `--color-primary` text (pakai warna aksen oranye, BUKAN biru — sistem ini cuma 1 warna aksen, `01-colors.md` poin 2: "Orange is a spotlight... Do not introduce a second accent color")
    - "Selesai" → bg `--color-bg-surface-subtle`, text `--color-text-tertiary` (paling pudar, menandakan tidak perlu aksi lagi)
    - "Diizinkan" → bg `--color-primary-soft` + text `--color-primary-hover` (variasi tint oranye yang sama, bukan warna baru)
  - Tombol **"Mulai Mengajar"** — Primary Button spec (`03-components.md`): `radius-full`, bg `--color-primary`, text putih, `14px/600`. Enabled hanya kalau `bisa_mulai: true` dari API; disabled state → turunkan opacity + `cursor-not-allowed`, dengan tooltip alasan (misal "Belum tap gerbang", "Belum waktunya", dst — mapping pesan error dari kode gagal)
  - Tombol **"Izin"** — Secondary Button spec (`03-components.md`): bg putih, border `1px solid --color-border-subtle`, `radius-full`. Enabled hanya kalau `sudah_diizinkan: true` DAN `tugas_sudah_diisi: false`; kalau `sudah_diizinkan: false` tombol ini disabled dengan tooltip "Belum ada izin dari Admin Jurnal"; kalau tugas sudah diisi, tombol berubah jadi "Lihat Tugas Titipan" (mode view, bukan submit lagi — tapi tetap bisa dibuka untuk revisi, lihat T046 soal submit ulang)
- Auto-refresh data setiap 60 detik (polling sederhana, TIDAK perlu Socket.IO untuk halaman ini — beda dari dashboard piket yang butuh realtime instan, di sini keterlambatan render beberapa puluh detik masih acceptable karena guru aktif berinteraksi sendiri, bukan pantauan pasif)

### Klik "Mulai Mengajar"
1. Browser minta izin geolocation (`navigator.geolocation.getCurrentPosition`)
2. Kalau user tolak izin lokasi / browser tidak support → tampilkan pesan jelas "Aktifkan lokasi untuk memulai sesi mengajar", JANGAN kirim request ke backend tanpa koordinat
3. Kalau lokasi didapat → `POST /teaching-sessions/:id/start` dengan `{ lat, lng }`
4. Sukses → redirect ke halaman detail sesi (T049, `/guru/sesi/:sessionId`)
5. Gagal → tampilkan pesan error PERSIS dari response backend (bukan pesan generik) — backend sudah menyediakan pesan spesifik per kasus (lihat T043), tampilkan apa adanya ke guru

### Klik "Izin" (tombol aktif)
- Buka modal/halaman form (lihat T049 untuk detail form ini — task ini cukup buat tombol & routing-nya, form detailnya di T049 bagian kedua)

## JANGAN
- ❌ JANGAN tampilkan tombol "Mulai Mengajar" sebagai enabled lalu baru gagal saat diklik untuk kondisi yang SUDAH diketahui dari response `GET /my-today` (misal belum tap gerbang) — kalau `bisa_mulai: false`, tombol harus benar-benar disabled di UI, bukan cuma warning setelah klik
- ❌ JANGAN redirect otomatis ke halaman lain kalau guru belum tap gerbang — guru tetap boleh lihat dashboard (sesuai spec: login tidak digate, hanya "Mulai Mengajar" yang digate)
- ❌ JANGAN buat polling interval lebih pendek dari 30 detik atau pakai Socket.IO untuk halaman ini — cukup polling 60 detik, ini bukan use case realtime kritis seperti dashboard piket
- ❌ JANGAN hardcode pesan error di frontend untuk kegagalan start-session — pakai pesan dari backend response supaya konsisten 1 sumber kebenaran
- ❌ JANGAN karang warna/radius/spacing sendiri di luar token `01-colors.md`/`04-layout-spacing.md` — kalau butuh warna status yang tidak ada di palet (misal "kuning" untuk warning), JANGAN tambah warna baru sendiri, gunakan kombinasi success/danger/primary-soft yang sudah ada (lihat "Konten halaman" untuk mapping yang benar)
- ❌ JANGAN pakai emoji sebagai ikon status (✅/⚠️) — `01-colors.md`/`MASTER.md` eksplisit melarang emoji sebagai kontrol UI, pakai ikon `lucide-react` + warna badge yang sesuai
- ❌ JANGAN pakai radius kecil (di bawah 10px) atau warna aksen kedua (biru/ungu/dll) untuk badge status "Sedang Berlangsung" — sistem ini cuma 1 warna aksen (oranye), jangan menambah warna baru demi "membedakan" status

## Files
- **Modifikasi:** `apps/web/app/(guru)/layout.tsx` — tambah menu sidebar
- **Buat:** `apps/web/app/(guru)/guru/page.tsx` (atau sesuaikan struktur route existing — cek dulu apakah route group `(guru)` sudah punya konvensi path, ikuti yang sudah ada)
- **Buat:** `apps/web/app/(guru)/guru/components/sesi-card.tsx`
- **Buat:** `apps/web/lib/use-jadwal-guru.ts` — hook fetch + polling `GET /teaching-sessions/my-today`
- **Modifikasi:** logic redirect setelah login — arahkan role `guru` ke `/guru` bukan `/riwayat`

## Acceptance Criteria
- [ ] Login sebagai guru dengan jadwal hari ini → halaman `/guru` tampilkan semua sesinya, terurut jam
- [ ] Guru belum tap gerbang → semua tombol "Mulai Mengajar" disabled, badge "Belum Tap Gerbang" tampil
- [ ] Klik "Mulai Mengajar" saat browser lokasi diblokir → pesan jelas muncul, TIDAK ada request ke backend
- [ ] Klik "Mulai Mengajar" berhasil → redirect ke halaman detail sesi
- [ ] Sesi yang sudah `sudah_diizinkan: true` → tombol "Mulai Mengajar" disabled, tombol "Izin" enabled
- [ ] Data refresh otomatis tanpa reload manual (tunggu 60 detik atau trigger manual test dengan interval dipercepat saat testing)
- [ ] Semua warna yang dipakai (badge, tombol, background) cocok persis dengan token di `01-colors.md` — tidak ada hex/warna baru yang tidak terdaftar di file itu
- [ ] Card sesi pakai radius 24px (`radius-xl`) dan shadow sesuai `shadow-card` dari `04-layout-spacing.md`, bukan radius/shadow default Tailwind
- [ ] Tidak ada emoji dipakai sebagai ikon status di manapun pada halaman ini
