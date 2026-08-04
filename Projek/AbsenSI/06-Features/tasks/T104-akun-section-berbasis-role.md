# T104 — UI: Manajemen Akun Berbasis Section (Admin/Piket/Guru/Pembina Ekstra)

## Depends on
Tidak ada — murni restrukturisasi UI `AkunView`, TIDAK mengubah skema/endpoint backend (`User.role` tetap 1 kolom seperti sekarang, section adalah lapisan presentasi yang memetakan ke role yang sudah ada).

## Objective
Ganti halaman Manajemen Akun (`apps/web/src/app/(admin)/akun/akun-view.tsx`) dari 1 tabel+form flat (dengan dropdown Role manual) menjadi 4 SECTION terpisah: **Admin**, **Piket**, **Guru**, **Pembina Ekstra**. Saat membuat akun baru, admin pilih section dulu (bukan role) — role otomatis terkunci sesuai section (kecuali section Admin, lihat di bawah), field form yang relevan otomatis mengikuti. Daftar akun juga ditampilkan terpisah per section, bukan 1 tabel gabungan semua role.

## Context
- **App:** `apps/web` (murni frontend — backend `UsersController`/`UsersService` TIDAK berubah sama sekali)
- Diskusi 2026-07-31 — user merasa halaman Manajemen Akun sekarang membingungkan karena harus paham istilah role teknis (`guru_piket`, dst) dan field form yang muncul-hilang tergantung role yang dipilih. Usulan: balik alurnya — section dulu (bahasa awam: "saya mau buat akun piket"), role & field ikut otomatis.
- **File existing yang jadi baseline**: `apps/web/src/app/(admin)/akun/akun-view.tsx` (594 baris) — `ROLE_OPTIONS` (baris 31) SUDAH USANG, cuma berisi 5 role (`super_admin, card_admin, guru, kepsek, guru_piket`), TIDAK termasuk `admin_jurnal` dan `pembina_ekstra` yang sudah ada di enum `UserRole` (`apps/web/src/lib/core-types.ts`) — **bug yang harus ikut diperbaiki di task ini**, admin_jurnal dan pembina_ekstra saat ini TIDAK BISA dibuat lewat UI sama sekali walau backend mendukungnya.

## Keputusan Final (dikonfirmasi user 2026-07-31)

1. **4 section**: Admin, Piket, Guru, Pembina Ekstra.
2. **Mapping section → role**:
   - **Piket** → `guru_piket` (1:1)
   - **Guru** → `guru` (1:1)
   - **Pembina Ekstra** → `pembina_ekstra` (1:1)
   - **Admin** → `super_admin`, `card_admin`, `admin_jurnal`, `kepsek` (4 role sekaligus — PENGECUALIAN, lihat poin 3)
3. **Section Admin TETAP punya dropdown Role di dalamnya** (dikonfirmasi eksplisit user, BEDA dari 3 section lain) — karena "Admin" bukan 1 kewenangan tunggal, melainkan payung buat 4 jenis admin dengan kewenangan sangat berbeda (super_admin = akses penuh, card_admin = cuma kartu, admin_jurnal = cuma jurnal, kepsek = read-only dashboard TV). Form section Admin: pilih section "Admin" → form MASIH menampilkan dropdown Role berisi 4 pilihan itu.
4. **3 section lain (Piket/Guru/Pembina Ekstra) TIDAK ADA dropdown Role sama sekali** — begitu section dipilih, `role` di-set otomatis di background (tidak terlihat/tidak bisa diubah user), field form yang muncul otomatis relevan:
   - Section **Piket** → field `teacherId` (pilih guru existing) + `kampusId` (pilih kampus) tetap muncul, SAMA seperti kondisional existing untuk role `guru_piket` sekarang.
   - Section **Guru** → field `teacherId` muncul (SAMA seperti kondisional existing untuk role `guru`).
   - Section **Pembina Ekstra** → TIDAK perlu `teacherId`/`kampusId` (akun berdiri sendiri, `teacherId` selalu null untuk role ini per desain T096) — cek apakah ada field lain yang relevan (misal assign ke ekstrakurikuler mana — TAPI itu kemungkinan dilakukan dari halaman `ekstra-kurikuler` (assign pembina ke ekstra), BUKAN dari form buat akun ini, JANGAN gabungkan 2 concern berbeda kecuali user eksplisit minta).
5. **Daftar akun ditampilkan terpisah per section** — bukan 1 tabel gabungan semua role seperti sekarang. Kemungkinan bentuk: 4 tab (mirip pola `Tabs` yang sudah dipakai di halaman Upload Foto, lihat T103 (06-Features/tasks/T103-sidebar-admin-berkelompok.md)) ATAU 4 halaman terpisah — **putuskan/konfirmasi ke user bentuk visual navigasinya sebelum implementasi** (tab dalam 1 halaman vs halaman terpisah dengan sub-nav) — belum eksplisit dibahas di diskusi ini.

## Spec Detail

### Perbaikan bug yang harus ikut (WAJIB, ditemukan saat riset task ini)
- `ROLE_OPTIONS` (baris 31, `akun-view.tsx`) HARUS mencakup SEMUA 7 role di enum `UserRole` (`super_admin, card_admin, guru, kepsek, guru_piket, admin_jurnal, pembina_ekstra`) — atau, dengan restrukturisasi T104 ini, `ROLE_OPTIONS` kemungkinan HANYA dipakai untuk dropdown Role di dalam section Admin (4 role: `super_admin, card_admin, admin_jurnal, kepsek`), sedangkan 3 role lain (`guru, guru_piket, pembina_ekstra`) tidak lagi butuh dropdown sama sekali (di-hardcode per section). Sesuaikan konstanta ini dengan struktur baru, JANGAN sekadar menambah 2 role yang hilang ke array lama tanpa mempertimbangkan restrukturisasi section.

### Form per section
- **Section Admin**: sama seperti form existing SEKARANG untuk role-role itu (dropdown Role tetap ada, field lain tidak berubah — `admin_jurnal`/`kepsek` kemungkinan tidak butuh `teacherId`/`kampusId` sama sekali, cek behavior existing untuk role-role ini kalau ada, atau ini kasus baru karena `admin_jurnal` belum pernah bisa dibuat lewat UI).
- **Section Piket**: `role` di-hardcode `guru_piket` di request body (TIDAK ada UI dropdown), field `teacherId` (wajib/opsional? cek existing) + `kampusId` (wajib, sesuai existing) tetap tampil.
- **Section Guru**: `role` di-hardcode `guru`, field `teacherId` tetap tampil.
- **Section Pembina Ekstra**: `role` di-hardcode `pembina_ekstra`, TIDAK ada field `teacherId`/`kampusId` (kecuali ditemukan kebutuhan lain saat implementasi — cek dulu ke user kalau ternyata pembina_ekstra butuh field tambahan yang belum kepikiran di sesi diskusi ini).

### Daftar akun
- Filter/tampilkan HANYA akun dengan role yang cocok section aktif — bisa lewat filter client-side (data akun semua di-fetch sekali, di-filter di frontend berdasar section aktif) ATAU query parameter baru ke backend (`GET /users?role=...`, cek dulu apakah endpoint ini sudah mendukung filter role, kemungkinan besar belum — **JANGAN ubah backend kalau filter client-side saja sudah cukup**, jumlah akun di sekolah ini kemungkinan tidak besar, filter di frontend lebih sederhana dan tidak perlu sentuh API).

## Business Rules
- **TIDAK ADA perubahan skema database maupun endpoint backend** — `POST /users`/`PATCH /users/:id` menerima payload YANG SAMA PERSIS seperti sekarang (`username, password, role, teacherId, kampusId`), section HANYA menentukan NILAI `role` mana yang dikirim dan field mana yang ditampilkan di form, murni logic frontend.
- Validasi backend existing (misal `guru_piket` wajib `kampusId`) TIDAK berubah — kalau ada, tetap berlaku, section di frontend cuma memandu UI supaya user tidak salah isi, bukan pengganti validasi backend.

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/akun/akun-view.tsx` — restrukturisasi total: state section aktif, form per section, daftar akun per section.
- **Jangan sentuh:** `apps/api/src/users/*` (controller, service, DTO) — tidak ada perubahan backend sama sekali untuk task ini.

## Acceptance Criteria
- [ ] 4 section tersedia: Admin, Piket, Guru, Pembina Ekstra.
- [ ] Section Admin: form punya dropdown Role (4 pilihan: super_admin/card_admin/admin_jurnal/kepsek).
- [ ] Section Piket/Guru/Pembina Ekstra: TIDAK ada dropdown Role, `role` terkirim otomatis sesuai section, field form yang tampil sesuai kebutuhan masing-masing (lihat spec di atas).
- [ ] `admin_jurnal` dan `pembina_ekstra` SEKARANG BISA dibuat lewat UI (bug lama diperbaiki).
- [ ] Daftar akun ter-filter per section yang sedang aktif, bukan 1 tabel gabungan semua role.
- [ ] Build + type-check `apps/web` hijau.
- [ ] Verifikasi manual: buat 1 akun contoh di tiap 4 section, pastikan role tersimpan benar di database sesuai section yang dipilih.

## Validasi Claudian
- [ ] Klarifikasi ke user SEBELUM implementasi: bentuk visual navigasi antar 4 section (tab vs halaman terpisah) — belum eksplisit diputuskan di diskusi ini.
- [ ] Klarifikasi ke user: apakah section Pembina Ekstra butuh field tambahan (misal assign langsung ke ekstrakurikuler tertentu) atau itu tetap dilakukan terpisah dari halaman `ekstra-kurikuler` seperti sekarang.
- [ ] Ini MURNI perubahan frontend — kalau saat eksekusi ternyata perlu endpoint baru (misal filter `GET /users?role=`), pertimbangkan dulu apakah filter client-side sudah cukup sebelum menyentuh backend.
- [ ] Baca SELURUH `akun-view.tsx` (594 baris, termasuk `SetPasswordForm`/`DeleteAccountForm` yang disebutkan di summary sebelumnya) sebelum restrukturisasi — pastikan fungsi reset password, set password manual, dan hapus/nonaktifkan akun (sudah ada dari sesi sebelumnya) tetap berfungsi utuh setelah dipecah per section.
