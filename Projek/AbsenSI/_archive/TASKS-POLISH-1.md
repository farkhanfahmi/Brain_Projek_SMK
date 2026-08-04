---
tags: [absensi, tasks, polish, bugfix]
updated: 2026-07-16
---

# Task Perbaikan Pasca-Fase-1 — Batch 1

← [[Projek/AbsenSI/00-INDEX AbsenSI|Index]]

> Temuan dari uji coba langsung oleh user (Fahmi) tanggal 2026-07-16, setelah seluruh Fase 1 (termasuk Fase 1b Dashboard Piket) selesai development. Dikerjakan bertahap, satu task per giliran, mengikuti pola yang sama dengan [[Projek/AbsenSI/TASKS-FASE-1|TASKS-FASE-1]] (baca sebelum coding, verifikasi Playwright, update catatan implementasi, cleanup data test).

---

## 📊 Progress

| Kategori | Task | Selesai |
|---|---|---|
| Bug kritis | P001 | 1/1 |
| Fitur — CRUD | P002–P004 | 3/3 |
| Fitur — Tambah manual | P005 | 1/1 |
| Desain UI/UX | P006–P008 | 3/3 |
| **Total** | | **8/8** |

---

## ✅ P001 — Fix Bug Cookies "Cookies can only be modified in a Server Action or Route Handler"

**Prioritas: Kritis** — menyebabkan navigasi admin gagal total secara intermiten. **Selesai 2026-07-16.**

- [x] Root cause: `apiFetch()` (`apps/web/src/lib/api.ts`) melakukan `cookieStore.set()` untuk auto-refresh token saat access token 401 — valid dipanggil dari Route Handler, TAPI fungsi ini juga dipakai langsung di banyak Server Component (`page.tsx`), yang tidak boleh menulis cookie di tengah render. Muncul intermiten karena hanya terjadi setelah access token 15 menit kedaluwarsa.
- [x] Dipakai di 14 file `page.tsx` (admin, piket, guru) — semua tetap jalan lewat `apiFetch()` yang sudah disederhanakan, tidak perlu diubah satu-satu.
- [x] Perbaikan: dipilih **opsi (b)** — pindahkan seluruh logic refresh ke `middleware.ts` (satu-satunya tempat legal menulis cookie sebelum render Server Component dimulai). `apiFetch()` disederhanakan jadi murni baca cookie + fetch, tanpa upaya refresh/write sama sekali.

**Catatan implementasi:**
- `apps/web/src/lib/jwt-decode-edge.ts` (baru) — decoder payload JWT pakai `atob()` (bukan `Buffer`, tidak tersedia di Edge Runtime tempat middleware jalan). Dipakai untuk baca klaim `exp` saja (tanpa verifikasi signature — otorisasi asli tetap di backend), demi keputusan proactive-refresh.
- `apps/web/src/middleware.ts` — sekarang cek `isJwtExpiringSoon(accessToken, 60)` di setiap request ke path terproteksi; kalau access token sudah/hampir expired (buffer 60 detik), panggil `POST /auth/refresh` ke backend, lalu set cookie baru dengan pola yang benar: `request.cookies.set()` dulu (supaya Server Component di request yang sama langsung lihat token baru) → baru `NextResponse.next({ request })` → baru `response.cookies.set()` (supaya browser dapat `Set-Cookie` header). Kalau refresh gagal (refresh token juga sudah dicabut/expired) → redirect ke `/login`.
- `apps/web/src/lib/api.ts` — `apiFetch()` disederhanakan total, tidak ada lagi `cookieStore.set()` di dalamnya. Kalau tetap dapat 401 (kasus langka: refresh token dicabut manual di tengah request), lempar `ApiError` yang ditangkap `error.tsx`.
- `apps/web/src/components/shell/route-error.tsx` (baru) + `error.tsx` di `(admin)`, `(piket)`, `(guru)` — jaring pengaman kedua: kalau ada 401 yang somehow tetap lolos sampai render, user diarahkan ke `/login` dengan pesan jelas, bukan layar "Unhandled Runtime Error" mentah.
- **Verifikasi end-to-end (Playwright):** login normal → ambil access token asli → re-sign dengan `exp` sudah lewat (pakai `JWT_SECRET` asli dari `.env`, tanpa perlu nunggu 15 menit sungguhan) → suntik ke cookie browser → navigasi ke `/kampus`. Hasil: halaman render normal penuh (sidebar, tabel data), **tidak ada** "Unhandled Runtime Error", dan cookie access token berubah nilai setelah navigasi (bukti middleware berhasil refresh diam-diam sebelum render).
- **Insiden sampingan yang ditemukan & diperbaiki selama verifikasi:** dev server `apps/web` (port 3000) sempat rusak (semua asset CSS/JS 404, halaman tampil tanpa styling) karena `.next` folder tertimpa `BUILD_ID` production dari build verifikasi sebelumnya (proses `next dev` dan `pnpm build` berbagi folder `.next` yang sama). Diperbaiki dengan restart dev server + hapus `.next` lama. **Pelajaran:** jangan pernah jalankan `pnpm build` di package yang dev server-nya sedang aktif tanpa restart dev server setelahnya.
- `pnpm turbo run build` (full monorepo, 6 task) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** ditemukan user 2026-07-16, root cause dianalisis & diperbaiki Claude Code sesi yang sama.

---

## ✅ P002 — CRUD Kampus Lengkap (Update + Delete)

**Selesai 2026-07-16.**

- [x] Backend: `PATCH /kampus/:id`, `DELETE /kampus/:id` (super_admin) — cek `kelas` dan `users` terkait sebelum delete (tolak dengan `BadRequestException` + pesan jelas kalau masih ada yang terikat, sesuai pola `JurusanService`).
- [x] UI: tombol edit (dialog mode create/edit, pola sama seperti `KelasForm`) dan delete (dialog konfirmasi terpisah, tombol merah `danger-text`) di halaman `/kampus`.

**Catatan implementasi:**
- `apps/api/src/core/kampus/dto/update-kampus.dto.ts` (baru), `kampus.service.ts` — tambah `update()` dan `remove()` (cek `kelas.count` + `user.count` by `kampusId`, keduanya harus 0 sebelum delete boleh jalan — `User.kampusId` nullable, jadi ikut dicek selain `Kelas.kampusId`).
- `apps/web/src/app/(admin)/kampus/kampus-view.tsx` — ditulis ulang total: state `editing`/`deleting` terpisah, `KampusForm` (create+edit sekaligus) dan `DeleteKampusForm` sebagai dialog kedua. Tombol hapus reuse komponen `Dialog` yang sudah ada (belum ada `AlertDialog` di `packages/ui`, tidak perlu tambah dependency baru untuk satu use-case ini).
- **Bug ditemukan & diperbaiki saat verifikasi:** `apps/web/src/lib/api.ts` — `apiFetch()` cuma skip `.json()` parse untuk status `204`, padahal NestJS default mengembalikan `200` dengan body kosong (`Content-Length: 0`) untuk handler yang return `undefined` (bukan `204` kecuali eksplisit `@HttpCode(204)`). Delete jadi gagal dengan pesan generik "Terjadi kesalahan" karena `.json()` gagal parse body kosong. Diperbaiki dengan baca `response.text()` dulu, kalau string kosong return `undefined`, baru `JSON.parse()` kalau ada isi — lebih robust untuk semua kasus body kosong, bukan cuma 204.
- **Insiden sampingan yang ditemukan & diperbaiki saat verifikasi:** proses `apps/api` yang aktif di port 3001 ternyata bukan `nest start --watch` yang live, melainkan proses `dist/main` lama yang orphan (zombie sejak sesi sebelumnya, serve kode lama tanpa endpoint PATCH/DELETE kampus). Diperbaiki: kill proses zombie, jalankan ulang `pnpm --filter @absensi/api dev` yang benar. **Pelajaran tambahan:** jangan asumsikan proses yang listen di port tertentu adalah proses dev yang diharapkan — cek `ps -fp $(lsof -i :PORT -t)` untuk pastikan command-nya benar (`nest start --watch`, bukan `node dist/main`).
- Verifikasi Playwright: create → muncul di tabel; edit → nama berubah; delete Kampus yang masih punya kelas → diblokir dengan pesan "Kampus tidak bisa dihapus — masih ada 2 kelas yang terikat ke kampus ini"; delete kampus tanpa kelas terkait → berhasil, hilang dari tabel. Semua lewat UI asli (dialog, bukan curl).
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] — kelas.kampus_id FK ke kampus.

---

## ✅ P003 — CRUD Jurusan Lengkap (Update UI + Delete)

**Selesai 2026-07-16.**

- [x] Backend: `PATCH /jurusan/:id` sudah ada — tambah `DELETE /jurusan/:id` (cek `kelas` terkait, pola sama dengan `KampusService.remove()` di P002).
- [x] UI: badge jurusan statis di `/kelas` sekarang punya ikon pensil (edit, buka dialog) dan ikon sampah (hapus, dialog konfirmasi terpisah) langsung di dalam badge — bukan diklik langsung di teksnya, supaya tetap jelas mana yang bisa ditekan.

**Catatan implementasi:**
- `apps/api/src/core/jurusan/jurusan.service.ts` — tambah `remove()`: cek `kelas.count({ where: { jurusanId } })`, tolak dengan `BadRequestException` kalau > 0.
- `apps/api/src/core/jurusan/jurusan.controller.ts` — tambah `@Delete(":id")`.
- `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` — `JurusanCard` ditulis ulang: state `editing`/`deleting`, komponen `JurusanForm` (create+edit) dan `DeleteJurusanForm` sebagai dialog terpisah, persis pola `KampusForm`/`DeleteKampusForm` dari P002. Badge sekarang `<span>` dengan 2 tombol ikon kecil (`Pencil`, `Trash2`) menempel di kanan nama.
- Verifikasi Playwright: create jurusan baru → muncul sebagai badge; edit nama → berubah; delete jurusan "Rekayasa Perangkat Lunak" (masih punya 2 kelas terkait, termasuk kelas baru "X RPL 2" yang sudah ada di data) → diblokir dengan pesan "Jurusan tidak bisa dihapus — masih ada 2 kelas yang terikat ke jurusan ini"; delete jurusan test tanpa kelas → berhasil, hilang dari daftar.
- **Insiden kecil selama verifikasi (self-caused, sudah dibereskan):** sanity-check curl PATCH sempat tidak sengaja menimpa nama jurusan seed asli ("Rekayasa Perangkat Lunak" → "test-check") karena lupa pakai id yang aman untuk uji coba manual. Langsung disadari dan dikembalikan ke nama asli sebelum lanjut ke pengujian Playwright yang sesungguhnya — dicek ulang di akhir task, data jurusan & kampus kembali sama seperti sebelum P002/P003 dikerjakan (2 kampus, 2 jurusan sesuai seed).
- **Pengulangan insiden dev-server dari P001/P002:** web dev server sempat berhenti compile chunk (`app-pages-internals.js` dll. tidak pernah ter-generate) setelah proses lama dibiarkan idle terlalu panjang lintas beberapa task berturut-turut — diperbaiki dengan restart bersih + hapus `.next`. Ini bukan disebabkan build production kali ini (tidak ada `BUILD_ID` production yang ketinggalan), kemungkinan proses dev yang idle lama kehilangan state webpack compiler-nya. **Pelajaran:** kalau curl ke halaman dev server balik 200 tapi asset JS-nya 404 padahal tidak habis `pnpm build`, jangan asumsikan itu race condition compile biasa — cek langsung isi folder `.next/static/chunks` dan restart proses kalau chunk penting memang tidak ada di disk.
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** sama dengan P002, halaman `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx`.

---

## ✅ P004 — Delete Kelas

**Selesai 2026-07-16.**

- [x] Backend: `DELETE /kelas/:id` (cek `students` **dan** `schedules` terkait sebelum delete — awalnya cuma diminta cek `students`, tapi `Schedule.kelasId` juga referensi ke kelas meski nullable secara skema, jadi ikut dicek supaya tidak ada jadwal yang diam-diam jadi orphan).
- [x] UI: tombol delete (ikon `Trash2`, merah) ditambahkan di samping tombol edit yang sudah ada di tabel Kelas (`/kelas`), dengan dialog konfirmasi terpisah — pola sama persis dengan P002/P003.

**Catatan implementasi:**
- `apps/api/src/core/kelas/kelas.service.ts` — tambah `remove()`: cek `student.count({ where: { kelasId } })` lalu `schedule.count({ where: { kelasId } })`, tolak dengan `BadRequestException` + pesan spesifik kalau salah satu > 0.
- `apps/api/src/core/kelas/kelas.controller.ts` — tambah `@Delete(":id")`.
- `apps/web/src/app/(admin)/kelas/kelas-jurusan-view.tsx` — `KelasCard` dapat state `deleting` baru + komponen `DeleteKelasForm` (pola identik `DeleteKampusForm`/`DeleteJurusanForm`). Kolom aksi tabel diperlebar (`w-16` → `w-24`) untuk menampung 2 ikon.
- Verifikasi Playwright: create kelas baru → muncul di tabel; delete "X RPL 1" (masih ada 2 siswa terkait, data seed) → diblokir dengan pesan "Kelas tidak bisa dihapus — masih ada 2 siswa yang terikat ke kelas ini"; delete kelas test tanpa siswa/jadwal → berhasil, hilang dari tabel. Data akhir dicek bersih — cuma 4 kelas asli (X RPL 1, X RPL 2, X TKJ 1, XI TKJ 1) yang tersisa.
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** sama dengan P002/P003.

---

## ✅ P005 — Tambah Manual Siswa & Guru

**Selesai 2026-07-16.**

- [x] Backend: `POST /students` (NISN, nama, kelas_id, tanggal_lahir opsional), `POST /teachers` (NIP, nama) — akses `super_admin`/`card_admin`, sama dengan role yang sudah pegang endpoint GET-nya (class-level `@Roles` di controller, tidak perlu decorator tambahan).
- [x] UI: dialog "Tambah Siswa" di `/siswa`, "Tambah Guru" di `/guru` — pola sama dengan dialog Kampus/Kelas (`Dialog` + form + `apiClientFetch` POST).
- [x] Validasi NISN/NIP unik — `ConflictException` (409) dengan pesan jelas kalau duplikat, konsisten dengan constraint `@unique` yang sudah ada di schema dan pola yang sama dipakai `ImportService`.

**Catatan implementasi:**
- `apps/api/src/core/students/dto/create-student.dto.ts` (baru) — `nisn`, `nama` wajib, `kelasId` (`@Type(() => Number)` karena form kirim string), `tanggalLahir` opsional (`@IsDateString`).
- `apps/api/src/core/students/students.service.ts` — `create()`: cek `kelas` ada dulu (`BadRequestException` kalau tidak), cek NISN belum dipakai (`ConflictException`), baru `prisma.student.create()`.
- `apps/api/src/core/teachers/dto/create-teacher.dto.ts` (baru), `teachers.service.ts` — `create()`: cek NIP belum dipakai, lebih sederhana dari student karena tidak ada relasi wajib.
- `apps/api/src/core/students/students.controller.ts`, `teachers.controller.ts` — tambah `@Post()` masing-masing, memanggil service yang baru.
- `apps/web/src/app/(admin)/guru/guru-view.tsx` — ditulis ulang: state `teachers` jadi lokal (bukan langsung dari props) supaya bisa di-`setState` setelah create, dialog `GuruForm` (NIP + Nama) mengikuti pola `KampusForm`.
- `apps/web/src/app/(admin)/siswa/siswa-view.tsx` — state `students` jadi lokal, dialog `SiswaForm` (NISN, Nama, `Select` Kelas dari `kelasList` yang sudah di-fetch page.tsx, `Input type="date"` opsional untuk tanggal lahir — pola date-input sama dengan yang dipakai di `holiday-form-dialog.tsx`/`academic-years-section.tsx`).
- Verifikasi Playwright: create guru baru → muncul di tabel; create guru dengan NIP duplikat → ditolak dengan pesan "NIP ... sudah terdaftar"; create siswa baru (dengan tanggal lahir) → muncul di tabel; create siswa dengan NISN duplikat → ditolak dengan pesan "NISN ... sudah terdaftar".
- **Catatan data test:** belum ada endpoint `DELETE` untuk `students`/`teachers` (di luar scope P005), jadi 2 baris data uji ("Guru PW Verify", "Siswa PW Verify") sengaja **dibiarkan** di database dev, tidak bisa dibersihkan lewat API seperti task-task sebelumnya. Tidak mengganggu fungsionalitas — cuma entri tambahan di tabel Siswa/Guru.
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** [[Projek/AbsenSI/06-Features/import-data-master|import-data-master.md]] (jalur import CSV yang sudah ada, tambah jalur manual sebagai pelengkap bukan pengganti).

---

## ✅ P006 — Sidebar: Scrollable + Collapsible

**Selesai 2026-07-16.**

- [x] `apps/web/src/components/shell/sidebar.tsx` — `overflow-y-auto` pada `<nav>`, nav item tidak lagi terpotong di layar pendek.
- [x] Mekanisme collapse (tombol toggle di bawah nav, localStorage untuk ingat preferensi) — dipilih **icon-only collapsed state** (bukan overlay slide-out), konsisten dengan sidebar yang selalu persistent di design system ini.

**Catatan implementasi:**
- `apps/web/src/components/shell/sidebar-state-context.tsx` (baru) — context `SidebarStateProvider`/`useSidebarState`, pola sama persis dengan `page-title-context.tsx` yang sudah ada. State `collapsed` disimpan di localStorage key `absensi:sidebar-collapsed`, dibaca via `useEffect` setelah mount (hindari hydration mismatch — server selalu render versi expanded dulu, baru client baca localStorage).
- `apps/web/src/components/shell/sidebar.tsx` — ditulis ulang jadi client component pemakai `useSidebarState()`. Lebar `w-60` (expanded) ↔ `w-24` (collapsed), transisi `transition-[width] duration-200`. Saat collapsed: label nav disembunyikan, ikon di-center, `title` attribute jadi tooltip pengganti label. Tombol toggle di bagian bawah sidebar (`ChevronLeft`/`ChevronRight`).
- `apps/web/src/app/(admin)/admin-content.tsx` (baru, client component) — dipisah dari `layout.tsx` (Server Component, tidak bisa langsung pakai `useSidebarState()`) supaya margin konten (`ml-60` ↔ `ml-24`) ikut bereaksi ke state collapse tanpa lempar seluruh layout jadi client component.
- `apps/web/src/app/(admin)/layout.tsx` — `Sidebar` dan `AdminContent` sekarang dibungkus `SidebarStateProvider`, `TopBarWithTitle`+`main` dipindah ke dalam `AdminContent`.
- **Scope check:** `(guru)` dan `(piket)` route group sengaja **tidak** pakai `Sidebar` sama sekali (`(guru)/layout.tsx` komentar eksplisit: guru tidak boleh punya jalur navigasi ke data lain; `(piket)` pakai `PiketNav` tab bar horizontal terpisah) — jadi P006 cuma menyentuh `(admin)`.
- Verifikasi Playwright: viewport pendek (1280×500) → nav `overflow-y: auto`, item terakhir ("Manajemen Akun") bisa dicapai lewat scroll; toggle collapse → lebar sidebar 240px → 96px; localStorage tersimpan `"1"`; reload halaman → tetap collapsed (tidak flash balik expanded); toggle expand lagi → balik 240px.
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** `apps/web/src/components/shell/sidebar.tsx`, `apps/web/src/app/(admin)/layout.tsx`, `apps/web/src/app/(admin)/admin-content.tsx`.

---

## ✅ P007 — Legend Kalender: Kontras Warna

**Selesai 2026-07-16.**

- [x] `apps/web/src/app/(admin)/kalender/calendar-utils.ts` — `HOLIDAY_COLOR`: 3 dari 4 warna (`libur_semester`, `libur_sekolah`, `libur_mendadak`) semua varian oranye (primary-soft/primary-soft-2/primary), sulit dibedakan sekilas.
- [x] Ganti jadi 4 warna yang jelas berbeda, semuanya token yang **sudah ada** di design system (tidak perlu tambah token baru — palet yang ada sudah cukup variatif begitu dipakai semua, bukan cuma varian oranye).

**Catatan implementasi:**
- Kombinasi baru: `libur_nasional` = merah (`bg-danger-bg text-danger-text`, tidak berubah), `libur_semester` = oranye (`bg-primary-soft text-primary`, tidak berubah), `libur_sekolah` = **hijau** (`bg-success-bg text-success-text`, sebelumnya oranye), `libur_mendadak` = **hitam/ink pekat** (`bg-ink text-white`, sebelumnya oranye solid). 4 keluarga warna berbeda (merah/oranye/hijau/hitam), bukan cuma gradasi satu warna.
- `apps/web/src/app/(admin)/kalender/kalender-view.tsx` — swatch `Legend` di bawah grid kalender disamakan (`bg-success-bg`, `bg-ink`) supaya tidak ada drift antara warna badge di grid dan warna di legend.
- Verifikasi Playwright: buat 3 hari libur sementara (satu per jenis yang berubah warnanya) via API, screenshot grid kalender bulan Juli 2026 — keempat jenis libur (termasuk yang sudah ada dari seed, "Libur Semester") langsung terlihat berbeda warnanya tanpa perlu baca teks. 3 hari libur test dihapus lagi setelah verifikasi (endpoint `DELETE /school-holidays/:id` sudah ada dari sebelumnya, jadi bisa dibersihkan lewat API — data akhir kembali ke 1 entri seed asli).
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** [[Projek/AbsenSI/08-UI-UX-Guidelines|08-UI-UX-Guidelines]], design system brief di vault `06-Features/design-system/`.

---

## ✅ P008 — Tombol Keluar (Destructive) + Kontras Field Input

**Selesai 2026-07-16 — task terakhir di backlog polish batch 1.**

- [x] `apps/web/src/components/shell/top-bar.tsx` — tombol "Keluar" pakai `text-ink` + `hover:bg-surface-subtle` (netral), diganti jadi merah/destructive (`text-danger-text`, `hover:bg-danger-bg`, `font-medium` supaya lebih tegas).
- [x] `packages/ui/src/components/ui/input.tsx` — `Input` pakai `bg-background` (sama dengan warna halaman `#EEE6D9`), diganti ke `bg-surface-subtle` (`#F7F3EC`) — warna krem terang yang beda baik dari background halaman maupun dari permukaan dialog putih/beige, jadi field terlihat jelas di kedua konteks.
- [x] Komponen lain dengan masalah kontras serupa: `SelectTrigger` di `packages/ui/src/components/ui/select.tsx` juga pakai `bg-background`, ikut diganti ke `bg-surface-subtle`. Tidak ada komponen `Textarea` di codebase ini (dicek `packages/ui/src/components/ui/` — cuma ada badge/button/calendar/date-picker/dialog/form/input/label/popover/select/skeleton/table), jadi tidak ada yang perlu disentuh di sana.

**Catatan implementasi:**
- Dicek juga `dialog.tsx` (`bg-background` untuk permukaan dialog itu sendiri) dan `button.tsx` varian `outline` (`bg-background`) — keduanya bukan bug kontras, itu memang gaya visual yang diinginkan (dialog tetap warna beige page, bukan putih — konsisten dengan screenshot di task-task sebelumnya), jadi **tidak diubah**.
- Verifikasi Playwright: dropdown logout → warna teks tombol "Keluar" dibaca via `getComputedStyle` = `rgb(225, 59, 59)` (persis `#E13B3B`, token `danger-text`). Dialog Tambah Kampus → background `Input` (`rgb(247, 243, 236)` / `#F7F3EC`) dibandingkan background dialog (`rgb(238, 230, 217)` / `#EEE6D9`) — beda, terkonfirmasi lewat kode maupun screenshot visual. Dialog Tambah Kelas → kedua `SelectTrigger` ("Pilih kampus", "Pilih jurusan") juga terlihat kontras dari background dialog, konsisten dengan `Input`.
- `pnpm turbo run build` (full monorepo) dan `pnpm --filter @absensi/api test` (Jest, 9 test) — keduanya hijau setelah fix.

**Ref:** design system brief (warna field harus punya kontras dari background page).

---

## Catatan Umum

- Setiap task: baca ulang bagian terkait di kode dulu (jangan asumsi dari deskripsi di sini saja — mungkin ada detail lain saat eksekusi).
- Verifikasi via Playwright (browser asli) untuk setiap perubahan UI, bukan cuma cek build sukses.
- P001 (bug cookies) dikerjakan paling dulu — ini blocker pengalaman pengguna yang aktif mengganggu, bukan cuma polish kosmetik.
- Setelah tiap task selesai: build + jest hijau, cleanup data test, update checklist & catatan implementasi di file ini.
