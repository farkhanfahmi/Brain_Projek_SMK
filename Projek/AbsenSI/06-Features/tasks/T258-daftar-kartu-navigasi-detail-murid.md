# T258 — Web: Daftar Kartu — Klik Nama Navigasi ke Detail Murid, Kelola Kartu Pindah ke Sana

## Depends on
Tidak ada. Independen — T259/T260 (task lain hasil diskusi sama) TIDAK bergantung pada task
ini, boleh dikerjakan dalam urutan apa pun.

## Objective
Klik nama siswa di halaman **Daftar Kartu** — SEKARANG buka Dialog "Riwayat Kartu" sempit
(cuma kartu, tanpa biodata). GANTI jadi navigasi ke halaman **Detail Murid**
(`/siswa/{id}`, sama seperti Direktori Siswa piket) — section "Riwayat Kartu" di halaman
itu (sekarang read-only) DIUPGRADE supaya bisa Tambah/Aktifkan/Ganti Kartu, menggantikan
fungsi Dialog lama sepenuhnya.

## Konteks — Dikonfirmasi via Riset 2026-08-28

**Pola navigasi SUDAH ADA dan established** — Direktori Siswa piket
(`apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx:207`) SUDAH
`router.push(\`/siswa/${student.id}\`)` — mengarah ke `apps/web/src/app/(admin)/siswa/[id]/`
(`page.tsx` + `siswa-detail-view.tsx`, komponen `SiswaDetailView` di-REUSE lintas role lewat
props `readOnly`/`hideRiwayatKartu`/`canAbsenManual`). **Kartu Daftar Kartu HARUS ikut pola
yang SAMA PERSIS**, BUKAN pola baru.

**Gap yang ditemukan**: `SiswaDetailView` sudah punya section "Riwayat Kartu"
(baris ~350-395) TAPI **READ-ONLY** — cuma tabel (UID/Status/Diterbitkan/Dicabut), TIDAK ADA
kolom Aksi, TIDAK ADA tombol Tambah Kartu. Sementara Dialog "Riwayat Kartu" di
`apps/web/src/app/(admin)/kartu/kartu-view.tsx` (baris ~690-730+) yang SELAMA INI dibuka
dari Daftar Kartu punya SEMUA aksi: tombol "Tambah Kartu" (kondisional `bisaTambahKartu`,
baris 687-703), kolom Aksi per baris (Aktifkan/Ganti Kartu, method `handleActivate()`
baris 651-662, `handleReplaced()`/`ReplaceCardForm`, `AddCardForm`).

## Spec Detail

### 1. `kartu-view.tsx` — ganti pembuka Dialog jadi navigasi
Cari handler `onClick` yang membuka Dialog "Riwayat Kartu — {target.nama}" (klik nama siswa
di tabel Daftar Kartu) — GANTI jadi `router.push(\`/siswa/${student.id}\`)`, KONSISTEN
persis pola Direktori Siswa piket. Dialog+komponen terkait (`AddCardForm`/`ReplaceCardForm`/
`handleActivate`) TIDAK dihapus dulu di langkah ini — akan DIPINDAHKAN (bukan diduplikasi)
ke `SiswaDetailView` di poin 2, baru file lama dibersihkan setelah migrasi selesai.

**PENTING — cek dulu apakah "Daftar Kartu" juga dipakai untuk GURU/KARYAWAN** (bukan cuma
siswa) — kalau baris tabel Daftar Kartu mencakup guru juga (T119 multi-kartu guru/karyawan),
navigasi utk guru HARUS ke halaman detail GURU yang sesuai (kalau ada), BUKAN `/siswa/{id}`
— VERIFIKASI SAAT IMPLEMENTASI struktur `target.type` (`"student"` vs `"teacher"`, sudah
terlihat dipakai di `bisaTambahKartu` baris 687-688) sebelum asumsi SEMUA baris adalah siswa.

### 2. `SiswaDetailView` — upgrade section Riwayat Kartu jadi bisa dikelola
Section "Riwayat Kartu" (baris ~350-395) — tambah:
- Kolom **Aksi** per baris kartu — "Aktifkan" (kartu nonaktif, REUSE `PATCH /cards/:id/activate`
  T165) dan "Ganti Kartu" (buka `ReplaceCardForm`, REUSE komponen dari `kartu-view.tsx` APA
  ADANYA — extract ke lokasi shared kalau dipakai 2 tempat, atau import langsung kalau
  strukturnya mengizinkan, VERIFIKASI SAAT IMPLEMENTASI).
- Tombol **"Tambah Kartu"** di header section — muncul kondisional SAMA PERSIS logic
  `bisaTambahKartu` existing (siswa: cuma kalau 0 kartu aktif; guru/karyawan: selalu boleh,
  T119) — REUSE `AddCardForm` APA ADANYA.
- **SEMUA aksi ini gated `!readOnly`** (prop existing `SiswaDetailView`) — piket
  (`hideRiwayatKartu=true`) tetap TIDAK LIHAT section ini sama sekali (tidak berubah); role
  lain yang `readOnly=true` (kalau ada) lihat section tapi TANPA tombol aksi (konsisten pola
  gating existing untuk edit kelas/foto di file yang sama).

### 3. Bersihkan Dialog lama SETELAH section baru berfungsi
Setelah section Riwayat Kartu di `SiswaDetailView` terverifikasi punya fungsi SETARA penuh
— hapus Dialog "Riwayat Kartu" lama + trigger-nya dari `kartu-view.tsx` (kode mati, sudah
tidak pernah dibuka lagi sejak poin 1). JANGAN hapus lebih dulu dari fungsi penggantinya
benar-benar siap (urutan penting: bangun baru → verifikasi → baru bongkar lama).

## Edge Cases
- **Card_admin role** (`hideRiwayatKartu=false` per komentar existing "card_admin TETAP
  lihat Riwayat Kartu") — HARUS tetap bisa kelola kartu penuh dari halaman Detail Murid
  (bukan cuma lihat) — VERIFIKASI role ini py `readOnly=false` atau minimal exception
  khusus utk section Riwayat Kartu (mengingat comment baris 71-74 bilang card_admin
  "cuma boleh CRUD kartu" — mungkin `readOnly=true` untuk field LAIN tapi Riwayat Kartu
  tetap harus aktif utk role ini, VERIFIKASI SAAT IMPLEMENTASI kombinasi prop yang benar).
- **Siswa dengan banyak riwayat kartu** (replace berkali-kali) — tabel Riwayat Kartu tetap
  tampil semua baris histori (TIDAK PERLU pagination, skala kecil per siswa, TIDAK sama
  dengan tabel Riwayat Presensi di T259 yang memang perlu pagination).

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/kartu/kartu-view.tsx` (ganti trigger Dialog jadi
  navigasi; hapus Dialog+form terkait SETELAH poin 2 selesai).
- **Modifikasi:** `apps/web/src/app/(admin)/siswa/[id]/siswa-detail-view.tsx` (section
  Riwayat Kartu tambah Aksi+Tambah Kartu).
- **Kemungkinan extract:** `AddCardForm`/`ReplaceCardForm` ke lokasi shared kalau dipakai
  2 tempat berbeda (VERIFIKASI struktur saat implementasi).

## Acceptance Criteria
- [x] Klik nama siswa di Daftar Kartu → navigasi ke `/siswa/{id}` (bukan Dialog lagi) —
      diganti `<Link>`, KONSISTEN pola Direktori Siswa piket.
- [x] Halaman Detail Murid — section Riwayat Kartu punya tombol Tambah Kartu (kondisional
      `!student.cards?.some(c => c.status === "active")`, aturan 1-kartu-aktif T119) +
      kolom Aksi (Aktifkan/Ganti/Nonaktifkan) per baris via `CardActions` shared.
- [x] Piket (Direktori Siswa) TIDAK melihat perubahan apa pun — `page.tsx`/`direktori-siswa-view.tsx`
      SAMA SEKALI tidak disentuh task ini, `hideRiwayatKartu={isPiket}` tetap identik —
      regresi nol dijamin struktural (bukan cuma asumsi).
- [x] Guru/karyawan di Daftar Kartu — **KEPUTUSAN**: TIDAK ADA halaman detail guru di
      codebase ini (dikonfirmasi grep, 0 hasil) — Dialog "Riwayat Kartu" LAMA TETAP
      DIPERTAHANKAN khusus utk 2 tab ini (BUKAN dihapus total, spec sendiri bilang
      "kalau ada" utk halaman detail guru — tidak ada, jadi tidak ada yang perlu dipindah
      untuk audiens ini). Nama guru/karyawan TETAP membuka Dialog SAMA seperti sebelumnya.
- [x] Dialog "Riwayat Kartu" — bagian yang benar-benar jadi kode mati (klik nama MURID)
      sudah dihapus; Dialog itu sendiri TIDAK dihapus total karena masih dipakai
      guru/karyawan (lihat poin di atas) — `RiwayatKartuDialog` disederhanakan jadi
      KHUSUS `type: "teacher"` (cabang student dihapus, tidak pernah lagi dipanggil).
- [x] Build + type-check hijau (`tsc --noEmit` bersih, `next build` sukses).

## Validasi Claudian
- [x] Konfirmasi tidak ada duplikasi logic — `CardActions`/`ReplaceCardForm`/`AddCardForm`
      DIEKSTRAK ke `components/card-management.tsx` (lokasi shared BARU), dipakai KEDUA
      tempat (`kartu-view.tsx` dialog guru/karyawan + `siswa-detail-view.tsx` section
      murid) via import, 0 baris logic diduplikasi.
- [x] Konfirmasi card_admin tetap bisa kelola kartu penuh — dikonfirmasi via ANALISIS KODE
      (bukan test login manual — lihat Implementasi): `readOnly = isPiket || isCardAdmin`
      TAPI `hideRiwayatKartu = isPiket` SAJA (BUKAN termasuk card_admin) — section baru
      SENGAJA TIDAK digating `!readOnly` tambahan, cuma `!hideRiwayatKartu` (existing,
      tidak diubah) — begitu section terlihat, pemiliknya PASTI super_admin ATAU
      card_admin, keduanya otomatis dapat akses penuh tanpa perlu prop baru.
- [x] Konfirmasi urutan pengerjaan — section baru DIBANGUN+DIVERIFIKASI (tsc+build hijau)
      SEBELUM kode Dialog lama utk MURID dihapus, dalam sesi eksekusi yang sama (bukan 2
      sesi terpisah, tapi urutan edit tetap: card-management.tsx dulu → kartu-view.tsx
      REUSE-kan import dulu → BARU hapus definisi lokal duplikat).

## Implementasi (2026-08-28)

**Ekstraksi shared**: `apps/web/src/components/card-management.tsx` BARU — `CardActions`
(Aktifkan/Ganti/Nonaktifkan per status), `ReplaceCardForm`, `AddCardForm` (REUSE endpoint
`POST /cards/tap-assign` sama persis), plus 2 helper API kecil `activateCard()`/
`revokeCard()` (`PATCH /cards/:id/activate|revoke`) — SEMUA disalin APA ADANYA dari
`kartu-view.tsx` lama, TIDAK ADA perubahan behavior.

**`kartu-view.tsx`**: baris Nama+ikon "Lihat Riwayat" MURID diganti `<Link href="/siswa/{id}">`
(dari `<button onClick={setRiwayatTarget}>`). Baris Nama guru/karyawan TIDAK diubah (masih
`setRiwayatTarget`, buka Dialog). `riwayatTarget` state disempitkan tipenya jadi HANYA
`{type: "teacher", ...}` (union `"student"|"teacher"` lama dihapus — TypeScript sendiri
yang memastikan tidak ada pemanggilan tersisa dgn type "student"). `RiwayatKartuDialog`
disederhanakan: `load()` selalu `/teachers/:id` (cabang `/students/:id` dihapus, tidak
pernah dipanggil lagi), `bisaTambahKartu` dihapus (guru SELALU boleh tambah, tidak perlu
cek lagi). Definisi lokal `CardActions`/`ReplaceCardForm`/`AddCardForm`/`StatusPill` DIHAPUS
(≈210 baris kode mati), diganti import dari shared + `StatusBadge` (`@absensi/ui`, sudah
dipakai file lain, KONSISTEN — `StatusPill` custom sebelumnya redundan).

**`siswa-detail-view.tsx`**: section Riwayat Kartu tambah kolom Aksi (`CardActions`) +
tombol "Tambah Kartu" di header (kondisional sama `bisaTambahKartu` lama). State baru
`replaceCardTarget`/`addingCard`/`cardError` + handler `refreshStudent()` (re-`GET
/students/:id` setelah mutasi apa pun, REPLIKASI pola `handlePindahKelas` yang sudah ada
di file yang sama — BUKAN re-fetch cards terpisah, cukup refresh `student` penuh karena
`Student.cards` sudah termasuk di response itu). 2 `Dialog` baru (Ganti Kartu, Tambah
Kartu) ditambah di akhir komponen, pola SAMA persis dialog lain di file ini.

**Verifikasi**: `tsc --noEmit` bersih, `next build` web sukses (`/siswa/[id]` naik
~1.3 kB mencerminkan kode baru, wajar). Regresi piket dijamin STRUKTURAL —
`(piket)/piket/siswa/direktori-siswa-view.tsx` dan `(admin)/siswa/[id]/page.tsx` (yang
menentukan `hideRiwayatKartu`) SAMA SEKALI tidak disentuh task ini. **Keterbatasan**:
test manual login sebagai card_admin sungguhan (klik Tambah/Ganti/Aktifkan dari halaman
Detail Murid) BELUM dilakukan — diverifikasi lewat pembacaan kode+alur prop, bukan
observasi browser. Test end-to-end kamera/tap kartu fisik jelas di luar scope verifikasi
sesi ini (sama seperti task-task UI murni sebelumnya).
