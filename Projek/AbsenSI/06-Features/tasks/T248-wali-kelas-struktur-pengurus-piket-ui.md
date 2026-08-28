# T248 — Web: Wali Kelas — UI Struktur Pengurus Kelas + Jadwal Piket Kelas

## Depends on
**T247** (schema `KelasPengurus`/`KelasPiketJadwal`/`User.studentId`) — WAJIB selesai dulu.

## Objective
1 halaman baru di sub-menu Wali Kelas: input struktur pengurus kelas (Ketua/Sekretaris/
Bendahara + 3 wakil opsional) di bagian atas, jadwal piket kebersihan kelas per hari di
bagian bawah. Assign siswa ke Ketua/Wakil Ketua OTOMATIS provision akun login siswa.

## Spec Detail — UI/UX (sudah disepakati dengan user 2026-08-25)

### 1. Sub-menu baru
Sidebar Wali Kelas (`apps/web/src/app/(guru)/guru/wali-kelas/`, pola sub-menu SAMA seperti
Biodata Murid/Rekap yang sudah ada, REPLIKASI struktur routing existing) — tambah item
**"Struktur & Piket Kelas"** (nama final VERIFIKASI ke user saat implementasi kalau ada
preferensi lain), route baru misal `/guru/wali-kelas/struktur-piket`.

### 2. Section atas — Struktur Pengurus
Card, 6 baris tetap (urutan: Ketua, Wakil Ketua, Sekretaris, Wakil Sekretaris, Bendahara,
Wakil Bendahara) — label jabatan kiri (badge "Opsional" kecil untuk 3 jabatan wakil), kanan
dropdown pencarian siswa (isi dari roster kelas wali ini, REUSE endpoint
`/journal/kelas-wali-siswa` yang sudah ada — sama sumber data dengan Biodata Murid).

**Auto-save per baris** (konsisten aturan project "auto-apply, no tombol Simpan terpisah",
T199) — pilih siswa → langsung PATCH ke backend, toast konfirmasi kecil. Slot kosong
(terutama yang wajib: Ketua/Sekretaris/Bendahara) tampil highlight/warning halus kalau
kosong (BUKAN blocking — wali kelas boleh isi bertahap).

**Provisioning akun otomatis** — assign siswa ke **Ketua** atau **Wakil Ketua** → backend
OTOMATIS provision akun `User` (role `ketua_kelas`, username=NISN, password default,
`mustChangePassword: true`) KALAU siswa itu belum punya akun. Assign ke Sekretaris/
Bendahara/wakil lainnya TIDAK provision akun apa pun (scope akun login CUMA Ketua+Wakil
Ketua, sesuai desain awal).

Tampilkan kredensial default (username+password) SEKALI setelah provisioning berhasil
(dialog konfirmasi, pola SAMA seperti `generateAkunGuruMassal()` T232 menampilkan hasil) —
wali kelas WAJIB sampaikan ke siswa bersangkutan.

**Ganti Ketua/Wakil Ketua ke siswa lain** — akun `User` siswa LAMA di-nonaktifkan
(`status: nonaktif`), akun siswa BARU di-provision (atau diaktifkan lagi kalau sudah pernah
punya akun dari histori tahun sebelumnya — VERIFIKASI SAAT IMPLEMENTASI, lihat Edge Cases
T247).

### 3. Section bawah — Jadwal Piket Kelas
Baris chip hari (Senin s/d Sabtu) — HANYA hari yang sudah pernah ditambah yang tampil
sebagai chip, + tombol "+ Tambah Hari" (dropdown pilih dari hari yang BELUM ada). Klik 1
chip → expand area di bawahnya: list siswa piket hari itu (badge nama + tombol hapus "x"
per siswa) + 1 dropdown "+ Tambah siswa piket" dari roster kelas. **REPLIKASI pola
klik-untuk-expand** yang SUDAH ADA di `hari-ini-tab.tsx` (SummaryCard+area expand di bawah)
— style visual KONSISTEN, bukan komponen baru dari nol.

Hapus chip hari (ikon "x" kecil di chip) → hapus SEMUA baris `KelasPiketJadwal` untuk hari
itu (konfirmasi ringan/toast-undo, BUKAN dialog konfirmasi berat — ini cuma jadwal
kebersihan, bukan data kritikal).

### 4. Mobile-responsif
Kedua section HARUS tetap usable di layar sempit (aturan wajib project, mobile-first) —
dropdown pencarian siswa full-width di mobile, chip hari wrap ke baris berikutnya kalau
tidak muat, area expand piket per-hari full-width. TIDAK PERLU scroll horizontal (beda dari
tabel data besar) — konten section ini secara natural vertikal/ringkas.

## Edge Cases
- **Kelas belum punya siswa sama sekali** — dropdown pencarian siswa kosong, tampilkan
  pesan jelas ("Belum ada siswa di kelas ini") BUKAN dropdown kosong tanpa penjelasan.
- **Siswa yang sama dipilih untuk 2 jabatan berbeda** (misal jadi Ketua SEKALIGUS piket
  hari Senin) — INI VALID, tidak ada exclusivity, satu siswa boleh rangkap peran struktur+
  piket. Yang TIDAK VALID: siswa sama jadi Ketua DAN Wakil Ketua sekaligus (constraint
  `@@unique([kelasId, jabatan, academicYearId])` di T247 sudah cegah 1 siswa dobel di
  JABATAN yang SAMA, tapi tidak cegah 1 siswa pegang 2 JABATAN BEDA — VERIFIKASI ke user
  apakah ini perlu dicegah atau dibiarkan boleh, REKOMENDASI: biarkan boleh, sekolah kecil
  wajar 1 siswa pegang lebih dari 1 peran).
- **Provisioning gagal** (misal NISN sudah kepakai sebagai username entitas lain — harusnya
  tidak mungkin karena NISN unik per Student, tapi VERIFIKASI tidak ada tabrakan username
  lintas Teacher/Student) — pesan error jelas, BUKAN 500 generik (aturan wajib project).

## Files
- **Buat:** halaman baru `apps/web/src/app/(guru)/guru/wali-kelas/struktur-piket/` (page.tsx
  + view component, pola SAMA struktur route existing sub-menu wali kelas).
- **Buat:** endpoint backend baru di `apps/api/src/journal/` (atau modul baru khusus kalau
  scope-nya dianggap cukup besar — VERIFIKASI SAAT IMPLEMENTASI) — CRUD `KelasPengurus` +
  `KelasPiketJadwal`, scoped `kelasIdWali` dari JWT (pola SAMA `JournalKelasWaliController`
  existing — `kelasId` TIDAK PERNAH dari query param).
- **Modifikasi:** `nav-groups.ts` (atau file sidebar wali kelas yang relevan) tambah menu.

## Acceptance Criteria
- [x] Wali kelas bisa assign siswa ke 6 jabatan pengurus, auto-save per baris (Select
      langsung PATCH, tanpa tombol Simpan terpisah).
- [x] Assign Ketua/Wakil Ketua otomatis provision akun, kredensial ditampilkan sekali
      (dialog, tidak bisa dilihat lagi setelah ditutup) — diverifikasi live via curl.
- [x] Ganti Ketua/Wakil Ketua menonaktifkan akun lama, provision/aktifkan akun baru —
      diverifikasi live (akun lama `status: nonaktif`, akun baru ter-provision aktif).
- [x] Wali kelas bisa tambah/hapus hari piket + assign/hapus siswa per hari — diverifikasi
      live (tambah hari, hapus siswa dari hari, hapus chip hari total).
- [x] Semua endpoint scoped ke `kelasIdWali` JWT wali kelas yang login — `requireKelasIdWali()`
      REPLIKASI PERSIS pola `JournalKelasWaliController`, kelasId TIDAK PERNAH dari body/query.
- [x] Responsif di layar mobile sempit (Select full-width mobile, chip hari wrap, area
      expand piket full-width — base Tailwind mobile, `sm:` untuk desktop).
- [x] `@LogActivity`-EQUIVALENT — dicatat MANUAL via `ActivityLogService.record()` di
      service (bukan decorator, karena `assignPengurus()`/`setPiketHari()` masing-masing
      bisa mutasi >1 tabel dalam 1 transaction, snapshot granular butuh kontrol manual).
- [x] Build + type-check hijau (`tsc --noEmit` api+web bersih).

## Validasi Claudian
- [x] Konfirmasi scoping kelasId SELALU dari JWT — SEMUA 4 endpoint (`GET /kelas-pengurus`,
      `POST /kelas-pengurus/:jabatan`, `GET/POST /kelas-pengurus/piket`) pakai
      `requireKelasIdWali(user)`, tidak ada satu pun yang terima kelasId dari client.
- [x] Konfirmasi provisioning akun idempotent — diverifikasi live: assign ulang siswa yang
      SAMA ke jabatan yang sama menghasilkan `provisioned: null` (tidak provision ulang,
      tidak error).
- [x] Test end-to-end backend LENGKAP via curl (browser test dilewati karena keterbatasan
      RAM mesin dev — lihat implementasi di bawah): assign Ketua → dapat kredensial
      username=NISN+password default → ganti ke siswa lain → akun lama nonaktif, akun baru
      aktif → kosongkan slot → akun ikut nonaktif. Piket: tambah hari → assign siswa →
      hapus 1 siswa → hapus hari total (chip hilang). Login sebagai siswa/T250 BELUM
      dibangun jadi tidak bisa ditest end-to-end sampai portal siswa ada.

## Bug Ditemukan + Diperbaiki Saat Implementasi (2026-08-27)

1. **Route ordering bug** — `@Post(":jabatan")` didaftarkan SEBELUM `@Post("piket")` di
   controller, menyebabkan NestJS/Express mencocokkan `POST /kelas-pengurus/piket` ke route
   dinamis duluan (`jabatan="piket"`), body divalidasi terhadap `AssignPengurusDto` yang
   SALAH (bukan `SetPiketHariDto`) → selalu 400 "property hari/studentIds should not exist".
   Awalnya disangka bug SWC (compiler dev hemat-RAM sesi ini) karena kebetulan test pertama
   dilakukan di server SWC — setelah restart ke tsc penuh bug TETAP ADA, membuktikan ini
   murni route ordering, bukan compiler. **Pelajaran**: route statis HARUS didaftarkan
   sebelum route dinamis pada prefix yang sama (`piket` vs `:jabatan`), NestJS tidak
   auto-reorder berdasar spesifisitas seperti beberapa router lain.
2. **Konvensi `hari` ganda ditemukan sebelum jadi bug** — `HARI_LABEL` global di
   `core-types.ts` pakai basis DAYOFWEEK MySQL (1=Minggu), TAPI `KelasPiketJadwal.hari`
   (T247) sengaja pakai basis independen `PiketSchedule` existing (1=Senin, ADR-024,
   komentar schema eksplisit). Ditemukan SEBELUM ditulis salah — frontend
   `struktur-piket-tab.tsx` REPLIKASI array lokal `HARI_LABEL = ["Senin",...]`
   (index+1=hari) dari `jadwal-piket-view.tsx`, BUKAN reuse `HARI_LABEL` global yang
   konvensinya beda.

## Implementasi (2026-08-27)

**Backend** — modul baru `kelas-pengurus` (`apps/api/src/kelas-pengurus/`):
- `KelasPengurusService.getPengurus()` — proyeksi 6 jabatan TETAP tampil (slot kosong =
  null di response, BUKAN baris DB studentId null, sesuai T247 spec).
- `assignPengurus()` — transaction: upsert `KelasPengurus` (unique kelasId+jabatan+academicYearId
  dari T247) + provisioning/nonaktifkan akun HANYA untuk jabatan ketua/wakil_ketua
  (`JABATAN_DENGAN_AKUN`). `provisionAkunJikaBelumAda()` — cek `User.findUnique({studentId})`
  dulu (idempotent), kalau sudah ada tapi nonaktif → aktifkan lagi TANPA reset password
  (histori tahun lalu, T247 edge case), kalau belum ada → create baru dengan
  `mustChangePassword: true` HARDCODE (bukan lewat `ForcePasswordChangeConfigService`,
  keputusan eksplisit — role ini di luar cakupan config itu).
- `setPiketHari()` — delete-then-create dalam transaction (pola sama
  `SchedulesService.upsertJamMasukHariList()`), `studentIds: []` = hapus hari total.
- `DEFAULT_PASSWORD = "12345678"` — SENGAJA duplikat konstanta kecil dari `UsersService`
  (bukan cross-import, ADR-003 batas modul — provisioning ini konteks beda: dipicu wali
  kelas, bukan admin).
- ActivityLog dicatat manual (`kelas_pengurus.assign/ganti/kosongkan`,
  `kelas_piket_jadwal.set_hari`) — bukan `@LogActivity` decorator (mutasi multi-tabel).

**Frontend**:
- `nav-groups.ts` — item baru "Struktur & Piket Kelas" di `WALI_KELAS_GROUP`.
- Route baru `struktur-piket/page.tsx` (server, `ensureWaliKelas()` guard) +
  `struktur-piket-tab.tsx` (client) — REPLIKASI pola grid-hari-klik dari
  `jadwal-piket-view.tsx` (ADR-024) untuk piket, REPLIKASI `kelas-searchable-select.tsx`
  untuk popover cari-siswa, dialog kredensial mirip pola `generate-akun-guru-dialog.tsx`
  tapi MENAMPILKAN password (beda dari itu yang tidak pernah echo password).
- Types baru di `core-types.ts`: `JabatanPengurus`, `JABATAN_PENGURUS_LABEL`,
  `JABATAN_PENGURUS_OPSIONAL`, `PengurusRow`, `AssignPengurusResult`, `PiketHariRow`.
