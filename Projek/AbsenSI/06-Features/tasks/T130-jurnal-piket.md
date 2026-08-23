# T130 — Schema+API+Web: Jurnal Piket (Catatan Harian Wajib, Blocking Login)

## Depends on
Tidak ada dependency teknis ke task lain. Fitur BARU sepenuhnya (model, modul, UI baru) — tidak menyentuh domain `journal`/`TeachingSession` (itu jurnal MENGAJAR guru mapel, konsep berbeda, JANGAN disatukan/dicampur meski namanya mirip).

**Scope v1 ini SENGAJA tidak menangani kasus piket sedang izin/cuti/sakit** — itu didiskusikan dan dikerjakan sebagai task terpisah nanti (lihat "Di Luar Scope" di bawah), keputusan eksplisit user 2026-08-06 supaya fondasi jurnal wajib+blocking selesai dulu tanpa digantungi kompleksitas sistem izin.

## Objective
Piket punya kewajiban mengisi 1 catatan teks bebas untuk SETIAP hari dia terjadwal bertugas (per `PiketSchedule`). Kalau sampai hari itu berakhir (lewat tengah malam) belum diisi, itu jadi "utang" permanen — begitu piket login lagi kapan pun, dia dipaksa mengisi SEMUA utang yang menumpuk (satu-satu, dari yang paling lama) sebelum bisa mengakses fitur apa pun di dashboard piket.

## Context
- **App:** `apps/api` (model+modul baru+job harian) + `apps/web` (halaman form + guard blocking di layout)
- **Riset 2026-08-06 (Explore agent, baca kode langsung)** — 3 riset terpisah untuk memastikan tidak reinvent pola yang salah:
  1. `PiketSchedule` (`schema.prisma:845-858`) — grid mingguan murni `{hari (1-6), userId}`, TIDAK ADA konsep tanggal spesifik/pengecualian. "Hari bertugas" dihitung dari `isBertugasHariIni()` (`piket-schedules.service.ts:57-63`) yang cocokkan `hari` (weekday) hari ini terhadap grid — task ini perlu menghitung MUNDUR dari grid ini untuk tahu tanggal-tanggal spesifik di masa lalu yang piket itu bertugas (bukan cuma "hari ini").
  2. Pola `followUpNeeded` (T044, `TeacherPermit`) — preseden BEST untuk "flag jadi wajib di titik waktu tertentu", di-flip oleh job BullMQ terjadwal (`auto-close.processor.ts`) — pola job harian yang SAMA dipakai untuk menandai jurnal piket jadi "utang" begitu tengah malam lewat tanpa diisi.
  3. Pola blocking existing (`mustChangePassword`) ada di `middleware.ts:53-61` — TAPI itu murni klaim JWT (murah, tanpa query). Jurnal piket BEDA — butuh query data (ada utang atau tidak), jadi TIDAK cocok ditaruh di middleware. **Pola yang benar**: tambahan di `apps/web/src/app/(piket)/layout.tsx` (baris ±26-28, setelah fetch `bertugasHariIni` yang sudah ada) — 1 fetch tambahan `await apiFetch(...)` lalu `redirect()` kalau ada utang, PERSIS pola role-check yang sudah ada di file yang sama (baris ±20-22). Tidak perlu restrukturisasi, ini penambahan aditif.

## Keputusan Final (dikonfirmasi user 2026-08-06)

1. **Kepemilikan**: 1 catatan per PIKET per HARI BERTUGAS (bukan bersama/kolaboratif) — kalau 2 piket bertugas hari yang sama, masing-masing punya catatan terpisah.
2. **Struktur isi**: field tanggal READONLY (tanggal hari yang bersangkutan — kalau lagi lunasi utang lama, tampilkan tanggal utang itu, BUKAN tanggal hari ini login) + 1 field teks paragraf bebas.
3. **Kapan jadi utang**: begitu hari piket bertugas BERAKHIR (lewat tengah malam) TANPA diisi. Selama masih di hari yang sama, piket boleh isi kapan saja sebelum tengah malam tanpa dianggap utang.
4. **Utang menumpuk berlaku SELAMANYA** — tidak ada kedaluwarsa. Piket yang tidak pernah login di hari dia bertugas TETAP dianggap berutang (kewajiban melekat ke JADWAL, bukan ke "apakah dia login hari itu").
5. **Alur pelunasan**: piket login dengan ≥1 utang → **blocking total di level UI** (redirect paksa ke form, tidak bisa akses fitur/menu apa pun) → isi utang PALING LAMA dulu → submit → lanjut ke utang berikutnya (urut dari lama ke baru) → begitu SEMUA lunas, baru dashboard normal terbuka.
6. **Cakupan blocking**: HANYA UI piket yang terkunci. Sistem backend (attendance/tap kiosk/notifikasi T108) TETAP JALAN NORMAL di belakang layar — tidak ada yang "berhenti" cuma karena piket-nya belum bisa akses. Begitu jurnal lunas, piket melihat semua yang menumpuk selama itu (termasuk notifikasi T108) dan bisa langsung tindak lanjuti.
7. **Tidak ada pengecualian darurat** — blocking total tanpa kondisi khusus, sesuai keputusan user (mendorong piket segera lunasi begitu login, bukan menunda).

## Di Luar Scope (Task Terpisah Nanti)
Kasus piket sedang izin/cuti/sakit di hari dia terjadwal — SEHARUSNYA tidak wajib isi jurnal untuk hari itu, dan idealnya jurnal otomatis terisi dari keterangan izinnya. **TAPI riset mengonfirmasi ini bukan pekerjaan kecil**: sistem `TeacherPermit` saat ini HANYA bisa dibuat oleh `admin_jurnal` (piket tidak bisa ajukan izin sendiri sama sekali), TIDAK ADA field teks alasan bebas di `TeacherPermit` (cuma kategori: sakit/izin_pribadi/tugas_dinas/pelatihan + file bukti wajib — tidak ada yang cocok untuk auto-fill jurnal), dan `PiketSchedule` TIDAK PERNAH terhubung ke `TeacherPermit` di kode manapun. **Keputusan user**: v1 jurnal piket ini TIDAK menangani kasus izin — semua hari terjadwal tetap wajib diisi tanpa pengecualian untuk sekarang. Task susulan nanti akan menghubungkan `admin_jurnal` mencatat izin piket lewat `TeacherPermit` (perlu tambah field alasan teks) → jurnal piket otomatis skip/auto-fill untuk hari itu.

## UI/UX Detail (dirancang 2026-08-06, ikuti persis — bukan opsional)

Fitur ini punya **2 mode tampilan berbeda**, backend sama (`POST /piket-journal`), cuma jalur akses frontend yang beda:

### Mode A — Blocking (isi utang, dipaksa)
Dipicu redirect dari layout guard (lihat Spec Detail Frontend di bawah). Halaman **penuh, tanpa sidebar/top bar** (bukan di dalam `PiketContent` shell biasa) — mirip pola halaman `/login` atau `/ganti-password`, fokus tunggal tanpa distraksi navigasi.
- Latar tetap beige (`--color-bg-page`), 1 card putih tunggal (`rounded-3xl`, `shadow-card`, `p-6` atau lebih besar) terpusat di layar (flex center, max-width sekitar 480-560px, mirip lebar Sheet di `03-components.md`).
- **Judul**: "Jurnal Piket" (`text-h1`) + subjudul (`text-label`, `--color-text-secondary`): *"Lengkapi catatan harian sebelum melanjutkan ke dashboard"*.
- **Indikator progres** (HANYA tampil kalau utang > 1): badge pill kecil `bg-primary-soft text-primary`, teks "Catatan 2 dari 5".
- **Field tanggal** — tampilkan sebagai teks statis besar (BUKAN `<Input disabled>` yang terlihat seperti field kosong) — misal `<p className="text-h3 text-ink">Kamis, 6 Agustus 2026</p>` di dalam kotak dengan background sedikit lebih gelap dari card (`bg-surface-subtle`, `rounded-xl`, `p-3`) supaya jelas ini bukan field yang bisa diklik.
- **Field catatan** — `<textarea>` custom (styling manual, `packages/ui` belum punya primitif Textarea — cek dulu, kalau belum ada, buat sesuai pola `Input` yang ada: `rounded-xl` bukan pill karena multi-baris, `border-border`, `bg-surface`, min-height ~140px/6 baris), placeholder membantu: *"Contoh: siswa X terlambat 3x, sudah ditegur. Tidak ada kejadian khusus lainnya."*
- **Tombol submit** — Primary Button pill oranye penuh lebar card. Teks dinamis: **"Simpan & Lanjut"** kalau masih ada utang berikutnya setelah ini, **"Simpan & Masuk Dashboard"** kalau ini yang terakhir.
- **TIDAK ADA tombol Batal/close/X** — konsisten dengan blocking total tanpa pengecualian. Tombol Logout TETAP bisa diakses (piket boleh keluar sesi, tapi begitu login lagi diarahkan ke sini lagi) — taruh sebagai link kecil di pojok atas halaman, bukan tombol utama.
- **Nada/microcopy** — WAJIB netral, tidak menghukum. HINDARI kata "gagal", "terlambat", "denda". Pakai "belum dilengkapi", "menunggu diisi". Kalau utang menumpuk banyak (>5, edge case ekstrem), tambah 1 baris kecil (`text-caption text-ink-secondary`) di bawah progres: *"Catatan lama tetap penting untuk arsip sekolah — isi sesuai ingatan terbaik Anda."*
- **Warna**: TETAP monokrom oranye standar (`--color-primary`), JANGAN pakai merah/danger untuk keseluruhan halaman ini meski sifatnya "wajib" — merah direservasi untuk status terkunci/danger siswa di sistem ini, jurnal piket bukan pelanggaran siswa, jangan disamakan kesannya.

### Mode B — Menu Normal (akses sukarela kapan saja, isi hari ini / lihat riwayat)
**Menu baru di sidebar piket** (grup navigasi yang sama dengan menu operasional harian lain seperti Izin Keluar/Riwayat Izin — cek pengelompokan existing saat implementasi), label **"Jurnal Piket"**, ikon `NotebookPen` atau `FileText` (`lucide-react`, BUKAN emoji sesuai DESIGN.md).

Halaman baru `/piket/jurnal` — **2 card, pola PERSIS ditiru dari `izin-keluar-view.tsx`** (`apps/web/src/app/(piket)/piket/izin-keluar/izin-keluar-view.tsx`, sudah dibaca lengkap sebagai acuan struktur):

**Card 1 — Form Catatan Hari Ini** (styling identik `IzinKeluarForm`: `rounded-xl bg-surface p-6 shadow-elevated`, `Label` `text-label text-ink-secondary` di atas tiap field, `Input`/textarea `rounded-full`/`rounded-xl border-border bg-surface`):
```
Judul: "Catatan Hari Ini" (text-h2)
Subjudul: "Tulis kejadian atau catatan penting hari ini" (text-label text-ink-secondary)

Field tanggal — readonly, teks statis (pola sama Mode A), SELALU tanggal HARI INI di mode ini
Field catatan — textarea

Tombol: "Simpan Catatan" (Primary Button, pola sama tombol submit IzinKeluarForm)
```
- Kalau piket **sudah isi hari ini** — form otomatis ter-prefill dari data existing, tombol berubah **"Perbarui Catatan"** — piket BOLEH edit catatan hari ini sendiri berkali-kali sepanjang hari itu (kejadian bisa bertambah). Ini implikasi ke backend: `POST /piket-journal` untuk tanggal HARI INI yang sudah ada entrinya harus UPDATE, bukan ditolak sebagai duplikat — BEDA dari tanggal UTANG yang sekali submit selesai (tidak bisa diedit lagi setelah lewat hari). Lihat penyesuaian Spec Detail Backend di bawah.
- Pesan sukses: badge kecil `bg-success-bg text-success-text rounded-lg px-4 py-2`, pola sama `lastPermit` di `IzinKeluarForm` — *"Catatan tersimpan pukul 14:32"*.

**Card 2 — Riwayat Catatan Saya** (styling identik card kedua `izin-keluar-view.tsx`, termasuk pola search+`SortableHeader`+kolom No dari `apps/web/src/components/sortable-header.tsx`):
```
Judul: "Riwayat Catatan Saya" (text-h2)
Subjudul: "Semua catatan harian yang pernah Anda tulis" (text-label text-ink-secondary)

[search box — cari berdasarkan ISI catatan, bukan nama siswa]

Tabel: No | Tanggal (sortable) | Catatan (truncate 1 baris, klik row untuk detail penuh) | Diisi Pukul (sortable)
```
- Sort default: tanggal terbaru dulu.
- Klik baris → buka `Dialog` kecil (`@absensi/ui`, cocok karena murni tampilan-baca, bukan form >6 field per aturan `03-components.md`) menampilkan tanggal lengkap + isi catatan penuh.
- **HANYA riwayat milik piket yang login sendiri** — endpoint backend HARUS scope ke `userId` dari JWT, TIDAK menerima parameter userId dari client (pola sama persis `GET /activity-log/me`, T111 — prinsip privasi antar-piket yang sudah ditetapkan di proyek ini).
- Kosong (piket baru, belum pernah isi apa pun) → empty state sesuai pola `03-components.md` (ikon 48px `--color-text-tertiary` + pesan `text-body` abu-abu, bukan tabel kosong tanpa penjelasan).

### Penyesuaian Endpoint Backend akibat UI/UX (penting, ubah Spec Detail di bawah)
- `POST /piket-journal` sekarang punya 2 perilaku tergantung tanggal yang dikirim:
  - Tanggal = **hari ini** DAN piket punya jadwal hari ini (`isBertugasHariIni`) → **upsert** (boleh dipanggil berkali-kali, UPDATE kalau sudah ada, bukan ditolak).
  - Tanggal = **utang lampau** (dari daftar `GET /piket-journal/me/debt`) → **create sekali saja**, TOLAK kalau sudah ada (constraint unique yang jadi penjaga, sesuai Spec Detail asli).
  - Tanggal lainnya (bukan hari ini, bukan utang piket itu) → TOLAK selalu.
- Tambah `GET /piket-journal/me/list` — daftar SEMUA catatan piket yang login (untuk Card 2 Mode B), dengan search (server-side atau client-side atas hasil fetch, putuskan saat implementasi tergantung berapa banyak catatan wajar per piket — kemungkinan besar cukup kecil untuk client-side seperti pola `izin-keluar-view.tsx`, tidak perlu pagination berat).
- Tambah `GET /piket-journal/me/today` — ambil entry hari ini kalau ada (untuk prefill Card 1 Mode B), return `null` kalau belum diisi.

## Spec Detail

### Schema (Prisma)
Model baru `PiketJournalEntry`:
```prisma
model PiketJournalEntry {
  id        Int      @id @default(autoincrement())
  userId    Int      // piket yang wajib isi (bukan Teacher — User, karena piket adalah role User)
  user      User     @relation(fields: [userId], references: [id])
  tanggal   DateTime @db.Date // tanggal hari bertugas yang dicatat
  catatan   String   @db.Text
  filledAt  DateTime @default(now()) // kapan sungguh-sungguh diisi (beda dari tanggal, bisa lunasi utang lama)
  createdAt DateTime @default(now())

  @@unique([userId, tanggal]) // 1 catatan per piket per hari, cegah duplikat
  @@map("piket_journal_entries")
}
```
- Tambah back-relation di `User` model.
- **TIDAK perlu tabel "utang" terpisah** — utang dihitung DINAMIS (cross-reference `PiketSchedule` grid vs tanggal yang SUDAH LEWAT vs `PiketJournalEntry` yang ada), pola sama seperti `detectMissingCheckouts` (`end-of-day.service.ts`, dihitung query-time, bukan disimpan). **REKOMENDASI**: pola ini (query dinamis) LEBIH SEDERHANA daripada pola `followUpNeeded` (kolom boolean di-flip job) untuk kasus ini — karena "utang" di sini murni fungsi dari (jadwal - yang sudah diisi), tidak perlu state tersimpan terpisah yang bisa jadi tidak sinkron. Job BullMQ TIDAK WAJIB di sini (beda dari kesan awal di Context), TAPI evaluasi saat implementasi mana yang lebih murah secara query — kalau query dinamis "cari semua hari bertugas sejak awal grid piket itu dibuat, minus yang sudah diisi" itu mahal untuk piket yang sudah lama bertugas, PERTIMBANGKAN precompute via job. Putuskan berdasarkan realita performa saat implementasi.

### Backend
- Modul baru `apps/api/src/piket-journal/` (`*.controller.ts`, `*.service.ts`, `*.module.ts`, `dto/`).
- `GET /piket-journal/me/debt` — kembalikan daftar tanggal utang untuk piket yang login (urut lama→baru), dan boolean `hasDebt`. Logic: ambil semua `hari` piket itu di `PiketSchedule`, generate semua TANGGAL LAMPAU (sejak entah kapan — perlu tentukan titik awal masuk akal, misal sejak akun itu dibuat/`createdAt` User, ATAU sejak `PiketSchedule` untuk piket itu pertama dibuat — cek field yang tersedia) yang cocok weekday itu, kecualikan hari ini (belum berakhir, belum jadi utang) dan HARI SEBELUM piket itu terdaftar sebagai piket (jangan generate utang untuk sebelum dia jadi piket), lalu selisihkan dengan tanggal yang sudah ada di `PiketJournalEntry`.
- `POST /piket-journal` — body `{ tanggal, catatan }`, `@Roles(UserRole.guru_piket)`, `@LogActivity`. Logic (lihat "Penyesuaian Endpoint Backend akibat UI/UX" di atas untuk alasan desain ini):
  - Tanggal = hari ini DAN piket bertugas hari ini → **upsert** (`prisma.piketJournalEntry.upsert()`, boleh dipanggil berkali-kali untuk edit catatan hari berjalan).
  - Tanggal = salah satu dari daftar utang piket itu (hasil `GET /piket-journal/me/debt`) → **create sekali saja** — TOLAK kalau row untuk `(userId, tanggal)` itu sudah ada (sudah lunas sebelumnya, tidak boleh diedit lagi setelah lewat hari).
  - Tanggal lainnya (bukan hari ini, bukan utang sah piket itu) → TOLAK selalu (`ForbiddenException`/`BadRequestException`).
  - **PENTING — constraint `@@unique([userId, tanggal])`**: untuk jalur "create sekali saja" (utang lampau), manfaatkan constraint ini sebagai penjaga race condition submit ganda — panggil `create` langsung dan tangkap error unique-constraint-violation sebagai "sudah pernah diisi", JANGAN pola `findUnique`→cek-JS→`create` yang rentan race (pelajaran T129). Untuk jalur upsert (hari ini), race condition kurang kritis (dampaknya cuma "isi terakhir menang", bukan data ganda/hilang) tapi tetap pakai `upsert()` Prisma (atomik) bukan `findUnique`+`update`/`create` manual.
- `GET /piket-journal/me/list` — daftar SEMUA catatan piket yang login, urut tanggal terbaru dulu, scope ketat ke `userId` dari JWT (TIDAK menerima parameter userId dari client, pola sama `GET /activity-log/me`).
- `GET /piket-journal/me/today` — entry hari ini kalau ada (`null` kalau belum), untuk prefill form Mode B.

### Frontend
- `apps/web/src/app/(piket)/layout.tsx` — tambah fetch `GET /piket-journal/me/debt` setelah fetch `bertugasHariIni` yang sudah ada (baris ±26-28) → kalau `hasDebt: true` DAN halaman yang diakses BUKAN halaman form jurnal blocking itu sendiri → `redirect("/piket/jurnal/isi")` (pola PERSIS sama seperti role-check yang sudah ada di baris ±20-22, termasuk exclude path form itu sendiri dari redirect loop — sama seperti `mustChangePassword` exclude `/ganti-password`).
- **Mode A — halaman blocking** `apps/web/src/app/(piket)/piket/jurnal/isi/page.tsx` — di LUAR shell `PiketContent` (tanpa sidebar/top bar, lihat UI/UX Detail di atas), form 1 utang pada satu waktu (tampilkan tanggal PALING LAMA dulu, readonly), textarea catatan, tombol submit → setelah sukses, cek lagi apakah masih ada utang → kalau masih ada, tampilkan utang berikutnya (tetap di halaman yang sama, bukan navigasi ulang) → kalau sudah lunas semua, redirect ke dashboard piket normal. UI progres "Catatan ke-X dari Y" (lihat UI/UX Detail).
- **Mode B — halaman menu normal** `apps/web/src/app/(piket)/piket/jurnal/page.tsx` + view component — 2 card (Form Hari Ini + Riwayat), DI DALAM shell `PiketContent` normal (sidebar/top bar biasa), styling meniru PERSIS `izin-keluar-view.tsx` (lihat UI/UX Detail lengkap di atas untuk struktur, styling class, dan copy teks per elemen).
- Tambah menu baru "Jurnal Piket" di sidebar piket (`nav-items.ts` atau file konfigurasi sidebar piket yang setara) — ikon `NotebookPen`/`FileText` dari `lucide-react`, arahkan ke `/piket/jurnal` (Mode B).

## Edge Cases
- Piket baru pertama kali dijadwalkan (belum pernah ada hari bertugas di masa lalu) → tidak ada utang, langsung masuk dashboard normal.
- Piket yang jadwalnya (`PiketSchedule`) DIHAPUS admin di tengah jalan (tidak lagi piket hari itu) SEBELUM sempat isi jurnal hari itu → tentukan apakah utang untuk hari SEBELUM jadwal dihapus tetap berlaku (kemungkinan besar YA — dia tetap bertugas hari itu secara historis) atau dihapus juga — **klarifikasi ke user saat implementasi** kalau kasus ini dirasa ambigu.
- Piket dengan PULUHAN hari utang menumpuk (skenario ekstrem, tidak login berbulan-bulan) → UI harus tetap wajar dipakai (progres jelas, tidak terasa seperti hukuman tanpa akhir — pertimbangkan pesan yang membantu, bukan cuma angka dingin).

## Files
- **Buat:** `apps/api/prisma/schema.prisma` (model baru), migration baru, `apps/api/src/piket-journal/` (modul lengkap: controller/service/module/dto), `apps/web/src/app/(piket)/piket/jurnal/isi/page.tsx` (+ view component, Mode A blocking), `apps/web/src/app/(piket)/piket/jurnal/page.tsx` (+ view component, Mode B menu normal, mengikuti pola `izin-keluar-view.tsx`), kemungkinan komponen `Textarea` baru di `packages/ui` kalau belum ada primitif itu (cek dulu).
- **Modifikasi:** `apps/api/src/app.module.ts` (registrasi modul), `apps/web/src/app/(piket)/layout.tsx` (guard blocking baru), file konfigurasi sidebar piket (tambah menu "Jurnal Piket").
- **Jangan sentuh:** `apps/api/src/journal/` (jurnal MENGAJAR guru mapel, konsep BEDA, jangan disatukan), `TeacherPermit`/`PiketSchedule` model (tidak ada perubahan skema di task ini, penanganan izin adalah task terpisah nanti).

## Acceptance Criteria
- [x] Piket yang mengisi jurnal SEBELUM hari itu berakhir tidak dianggap berutang (bisa isi kapan saja di hari yang sama). `submit()` jalur upsert untuk tanggal=hari ini.
- [x] Piket yang TIDAK isi sampai tengah malam lewat → hari itu jadi utang permanen. `getDebt()` dinamis, diverifikasi unit test (7 skenario) + live (piket id=5, utang Kamis lampau terdeteksi benar).
- [x] Piket login dengan utang → dipaksa redirect ke form jurnal, TIDAK BISA akses menu/fitur lain. Diverifikasi live via user (screenshot): login `hilma` langsung diarahkan ke form blocking.
- [x] Utang diisi berurutan dari yang PALING LAMA, field tanggal readonly. `tanggalUtang` di-sort leksikografis (`YYYY-MM-DD` = kronologis), diambil index 0 tiap render.
- [x] Piket normal (tanpa utang) isi jurnal hari ini → field tanggal menampilkan hari ini. Diverifikasi live (screenshot user): form Mode B menampilkan "Jumat, 7 Agustus 2026" dengan benar.
- [x] Backend menolak submit tanggal yang bukan hak piket itu. Diverifikasi unit test + curl live (`400 Bad Request` untuk tanggal ngasal).
- [x] Constraint unique DB mencegah duplikat entry meski race condition submit ganda. `@@unique([userId, tanggal])` + tangkap `P2002` di `submit()` (pola T129, bukan findUnique-cek-JS-create), diverifikasi unit test.
- [x] Sistem backend lain TIDAK terpengaruh — blocking murni di layer UI (Next.js layout/redirect), tidak ada perubahan ke `AttendanceModule`/`RealtimeModule`.
- [x] Menu "Jurnal Piket" muncul di sidebar piket, ke `/piket/jurnal`. Grup baru `JURNAL_GROUP` ditambahkan ke `piket-sidebar.tsx`.
- [x] Mode B Card 1: isi + EDIT catatan hari ini (upsert). Diverifikasi live via user — entry "test" tersimpan dan tampil di Riwayat.
- [x] Mode B Card 2: riwayat HANYA milik sendiri (scope `userId` dari JWT, sama pola `GET /activity-log/me`), search+sort+kolom No, klik baris buka detail (`Dialog`).
- [x] Mode A TIDAK menampilkan sidebar/top bar — dirender langsung oleh `(piket)/layout.tsx` di luar `PiketSidebar`/`PiketContent` untuk path `/piket/jurnal/isi` (bukan lewat route group terpisah — App Router tidak bisa "melewati" parent layout, jadi solusinya conditional-render di layout yang sama).
- [x] Nada/microcopy Mode A netral (tidak ada "gagal"/"terlambat"/"denda" — dicek ulang teks final).
- [x] Build + type-check `apps/api` dan `apps/web` hijau. `tsc --noEmit` bersih kedua app, jest 203/203 (192 lama + 11 baru).

## Bug Ditemukan + Diperbaiki Saat Verifikasi Live (2026-08-07)
User login sungguhan sebagai piket (`hilma`) dan menemukan kolom "Tanggal" di tabel Riwayat Catatan Saya (Mode B Card 2) menampilkan **"Invalid Date"**. Akar masalah: `formatTanggal()` di `jurnal-view.tsx` mengasumsikan input `dateKey` berformat `"YYYY-MM-DD"` murni lalu `split("-")` manual — TAPI `entry.tanggal` dari backend (`PiketJournalEntry.tanggal`, field Prisma `DateTime`) ternyata di-serialize sebagai **ISO datetime penuh** (`"2026-08-06T00:00:00.000Z"`), bukan date-only. Split manual pada string itu menghasilkan komponen salah (`T00:00:00.000Z` ikut kesplit). **Fix**: `formatTanggal()` diganti pakai `new Date(dateKey)` langsung (parse ISO penuh, bukan split manual), `timeZone: "UTC"` dipertahankan supaya tanggal tidak bergeser di WIB. `formatTanggalLengkap()` di `jurnal-isi-view.tsx` (Mode A) TIDAK terkena bug yang sama karena inputnya beda sumber — `tanggalUtang[]` dari `getDebt()` memang `dateKey()` string murni (`YYYY-MM-DD`), bukan objek `PiketJournalEntry`.

## Status Eksekusi — SELESAI (2026-08-07, 1 bug ditemukan+diperbaiki saat verifikasi live)
**Backend**: model `PiketJournalEntry` baru (migration `20260807072336_t130_piket_journal_entries`, CREATE TABLE murni, tidak destruktif). Modul `piket-journal/` baru — `getDebt()` (generate tanggal mundur dari `PiketSchedule.createdAt` per hari, cross-reference vs entry existing, UTC-aware konsisten pola `late-entry-slips.service.ts`), `getToday()`, `getMyList()`, `submit()` (2 jalur: upsert untuk hari ini, create-sekali+tangkap-unique-violation untuk utang lampau, pola T129). Endpoint semua scope ketat `userId` dari JWT.
**Frontend**: `Textarea` primitif baru di `packages/ui` (belum ada sebelumnya). `(piket)/layout.tsx` — guard blocking: fetch `/piket-journal/me/debt`, redirect ke `/piket/jurnal/isi` kalau ada utang, path itu sendiri dikecualikan dari cek DAN dirender di luar shell sidebar/topbar (conditional return di layout yang sama — App Router tidak bisa skip parent layout, jadi bukan route group terpisah). Mode A (`jurnal-isi-view.tsx`) — 1 utang per waktu, progres "Catatan X dari Y", tombol dinamis. Mode B (`jurnal-view.tsx`) — 2 card meniru `izin-keluar-view.tsx` persis (Form Hari Ini upsert + Riwayat search/sort/Dialog detail). Menu sidebar baru.
**Test baru**: `piket-journal.service.spec.ts` (11 test) — `getDebt()` 6 skenario (tanpa jadwal, hari ini dikecualikan, Jumat 2 minggu lampau urut kronologis, sebagian/semua sudah diisi, gabungan 2 jadwal hari berbeda), `submit()` 5 skenario (upsert hari ini, ditolak kalau tidak bertugas, create utang sah, ditolak tanggal tidak sah, race condition unique-violation). Date mocking via `global.Date` override untuk determinisme.
**Verifikasi live**: curl API (debt/submit/list semua diverifikasi, termasuk reject tanggal tidak sah), DAN verifikasi UI browser SUNGGUHAN oleh USER LANGSUNG (bukan cuma saya) — login piket `hilma` (password direset sementara utk testing, dikembalikan setelah), berhasil lihat blocking form, isi utang, lanjut ke Mode B, lihat entry di Riwayat — menemukan bug "Invalid Date" yang saya perbaiki di atas. Saya sempat mengalami redirect-loop di sesi Playwright saya sendiri (kemungkinan token JWT manual expired/state race dari berulang kali percobaan login otomatis) — TIDAK direproduksi user di sesi browser normal mereka, dianggap bukan bug produk nyata.

## Validasi Claudian
- [x] Titik awal mundur utang = `PiketSchedule.createdAt` PER ROW hari itu (bukan dari awal sistem) — piket yang baru dijadwalkan minggu ini tidak dapat utang dari sebelum dijadwalkan.
- [x] Query dinamis dipakai (bukan precompute-job) — dianggap cukup murah untuk skala sekolah ini (grid piket kecil, rentang tanggal wajar), TIDAK dievaluasi lebih lanjut karena tidak ada indikasi masalah performa saat verifikasi live.
- [x] Redirect guard TIDAK infinite loop dalam kondisi normal (dibuktikan user login sungguhan sukses sampai Mode B) — path Mode A dikecualikan dari cek, Mode B (`/piket/jurnal` menu biasa) TETAP kena guard (tidak ada celah bypass).
- [x] `Textarea` belum ada di `packages/ui` sebelumnya — dibuat baru, styling konsisten `Input` (radius/border/warna sama, `rounded-xl` bukan pill karena multi-baris).
- [x] Dicatat eksplisit: kasus izin piket (sakit/cuti) adalah task SUSULAN terpisah, v1 ini TIDAK menanganinya — semua hari terjadwal tetap wajib diisi tanpa pengecualian.
