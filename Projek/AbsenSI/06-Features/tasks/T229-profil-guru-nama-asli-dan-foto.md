# T229 — API+Web: Profil Guru Tampilkan Nama Asli + Foto (Bukan Username/Inisial)

## Depends on
Tidak ada dependency teknis. Independen, murni modul `users`/`auth` (backend) + shell guru (frontend).

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-20)

Seluruh UI akun guru (top-bar mobile, top-bar desktop, drawer profil) menampilkan **`user.username`** (dari JWT) sebagai nama tampilan — BUKAN nama asli guru (`Teacher.nama`). Avatar SELALU inisial teks atau ikon generik, TIDAK PERNAH foto asli meski `Teacher.foto` sudah ada di database dan terisi (dipakai di modul lain: kartu presensi realtime, `photos.service.ts`).

**Komentar eksplisit sudah ada di kode** (`profile-drawer.tsx`, T168) mengakui keterbatasan ini:
```
T168 — drawer profil, dibuka lewat avatar di top bar. Tidak ada field foto profil
yang diekspos GET /users/me saat ini (Teacher.foto ada di DB tapi belum
dikembalikan endpoint itu) — di luar scope task ini untuk menambahkannya, jadi
avatar SELALU inisial nama, bukan foto.
```

**Temuan penting**: `GET /users/me` (`apps/api/src/users/users.service.ts:199-211`) **SUDAH** mengembalikan `teacherNama` (dari `Teacher.nama`) — field ini SUDAH dipakai dengan benar di route group `(piket)` (`apps/web/src/app/(piket)/piket/page.tsx:30`, `me.teacherNama ?? user!.username`). **TAPI** di `(guru)/layout.tsx:30`, response `/users/me` di-tipe-kan HANYA `{ isPembinaEkstra: boolean }` — field `teacherNama` yang sebenarnya ADA di response tetap diabaikan, dan `userName={user.username}` (baris 41) tetap dipakai dari JWT, BUKAN dari `teacherNama`.

## Spec Detail

### 1. Backend — tambah `foto` ke `GET /users/me`

`apps/api/src/users/users.service.ts:199-211`, method `findMe()` — TAMBAH `foto` ke `select`:
```ts
async findMe(id: number): Promise<{ isPembinaEkstra: boolean; ekstrakurikulerDibinaId: number | null; teacherNama: string | null; teacherFoto: string | null }> {
  const [ekstra, user] = await Promise.all([
    this.prisma.ekstrakurikuler.findUnique({ where: { pembinaId: id }, select: { id: true } }),
    this.prisma.user.findUnique({ where: { id }, select: { teacher: { select: { nama: true, foto: true } } } }),
  ]);
  return {
    isPembinaEkstra: !!ekstra,
    ekstrakurikulerDibinaId: ekstra?.id ?? null,
    teacherNama: user?.teacher?.nama ?? null,
    teacherFoto: user?.teacher?.foto ?? null,
  };
}
```
- **VERIFIKASI SAAT IMPLEMENTASI**: cek bagaimana `foto` disimpan untuk `Student` (pola existing T028, kemungkinan path relatif ke folder upload) — REPLIKASI cara resolve URL foto lengkap yang sudah dipakai di tempat lain (`photos.service.ts`), JANGAN buat pola URL baru berbeda.
- User TANPA `teacherId` (misal `super_admin`/`card_admin` yang bukan guru) — `teacherNama`/`teacherFoto` tetap `null`, endpoint TIDAK error.

### 2. Frontend — `(guru)/layout.tsx` pakai `teacherNama`, bukan `username`

`apps/web/src/app/(guru)/layout.tsx` — UBAH tipe fetch `/users/me` dari `{ isPembinaEkstra: boolean }` jadi include `teacherNama`+`teacherFoto` (sesuai response backend poin 1) — PAKAI `me.teacherNama ?? user.username` sebagai `userName` yang diteruskan ke `GuruShell` (KONSISTEN pola fallback yang SUDAH dipakai `(piket)/piket/page.tsx:30` — REPLIKASI, bukan pola baru).
- **Fallback ke `username` WAJIB dipertahankan** untuk kasus `teacherNama` null (data lama/tidak lengkap) — JANGAN tampilkan string kosong/undefined kalau `teacherNama` null.

### 3. Frontend — teruskan `userFoto` ke semua komponen avatar

- `apps/web/src/app/(guru)/components/guru-shell.tsx` — terima prop baru `userFoto?: string | null`, teruskan ke `TopBar` (mobile), `TopBarWithTitle` (desktop), dan `ProfileDrawer`.
- `apps/web/src/app/(guru)/components/top-bar.tsx` — avatar bulat SAAT INI selalu render `initials(userName)` — UBAH jadi: kalau `userFoto` ada, render `<Image>` foto; kalau tidak, FALLBACK ke inisial seperti sekarang (JANGAN hilangkan fallback inisial, itu tetap dibutuhkan untuk guru tanpa foto).
- `apps/web/src/app/(guru)/components/profile-drawer.tsx` — SAMA POLA, avatar besar di drawer render foto kalau ada, fallback inisial kalau tidak. Hapus/update komentar T168 yang menyatakan ini "di luar scope" (sudah dikerjakan task ini).
- `apps/web/src/components/shell/top-bar.tsx` (desktop shared, dipakai lintas route group termasuk guru) — SAAT INI avatar selalu ikon generik `<UserIcon>` — VERIFIKASI SAAT IMPLEMENTASI apakah komponen shared ini juga perlu terima prop foto opsional (dipakai role lain seperti admin yang mungkin tidak punya foto sama sekali, jadi TETAP fallback ke ikon generik untuk role non-guru) — REKOMENDASI: tambah prop `userFoto?: string | null` opsional, dipakai HANYA kalau ada, tidak mengubah behavior existing untuk role yang tidak mengirim prop ini.

## Edge Cases

- **Guru tanpa foto sama sekali** (`Teacher.foto` null) — SEMUA titik fallback ke inisial nama (`teacherNama` kalau ada, `username` kalau tidak) — KONSISTEN behavior yang sudah ada sekarang, HANYA tambahan foto sebagai prioritas pertama kalau tersedia.
- **File foto ada di DB tapi file fisik hilang/rusak** (path tidak valid) — `<Image>` gagal load — PASTIKAN ada `onError` fallback ke inisial (BUKAN broken-image icon browser default), KONSISTEN pola penanganan foto siswa kalau sudah ada polanya di tempat lain.
- **User bukan guru** (super_admin dsb mengakses shell lain, di luar scope `(guru)/`) — TIDAK terdampak task ini sama sekali (scope murni shell guru).

## Files
- **Modifikasi:** `apps/api/src/users/users.service.ts` (`findMe()` tambah `foto`), `apps/web/src/app/(guru)/layout.tsx` (pakai `teacherNama`), `apps/web/src/app/(guru)/components/guru-shell.tsx`+`top-bar.tsx`+`profile-drawer.tsx` (terima+render `userFoto`), `apps/web/src/components/shell/top-bar.tsx` (opsional, prop foto).
- **Jangan sentuh:** `(piket)/piket/page.tsx` (SUDAH benar, pola referensi — tidak perlu diubah), JWT payload backend (`AccessTokenPayload`) — TIDAK PERLU tambah field ke situ, cukup lewat `/users/me` yang sudah ada (JWT tetap ringkas, tidak perlu bengkak dengan data yang bisa berubah seperti foto/nama).

## Acceptance Criteria
- [x] Top-bar mobile guru — tampilkan nama asli guru (`teacherNama`), fallback `username` kalau null.
- [x] Top-bar desktop guru — SAMA, nama asli bukan username.
- [x] Drawer profil — nama asli + foto (kalau ada) di avatar besar.
- [x] Avatar (mobile+desktop+drawer) — tampilkan foto asli guru kalau `Teacher.foto` terisi, fallback inisial kalau tidak.
- [x] Guru tanpa foto — tetap tampil inisial seperti sekarang, TIDAK error/broken image (`onError` handler di semua 3 titik).
- [x] Role lain (piket, admin, dst di luar shell guru) — TIDAK terdampak, behavior tetap sama seperti sebelumnya (prop `userFoto` opsional, tidak dikirim = fallback ikon generik seperti semula).
- [x] Build + type-check hijau (tsc api+web bersih).

## Validasi Claudian
- [x] Konfirmasi fallback `teacherNama ?? username` REPLIKASI pola yang SUDAH ada di `(piket)/piket/page.tsx`, bukan pola baru yang berbeda — dikonfirmasi sama persis di `(guru)/layout.tsx`.
- [x] Konfirmasi resolve URL foto REUSE pola existing dari modul lain yang sudah pakai `Teacher.foto` — grep menemukan pola `/api/photo-proxy/students/${foto.split("/").pop()}` dipakai 6 tempat di modul siswa, direplikasi persis jadi `/api/photo-proxy/teachers/${userFoto.split("/").pop()}`.
- [x] Konfirmasi SEMUA titik avatar (mobile top-bar, desktop top-bar, drawer) diperbaiki bersamaan — diverifikasi live via Playwright screenshot ketiganya sekaligus dalam 1 sesi login, bukan 1 titik saja.

## Verifikasi Live (2026-08-21)
Foto test sementara di-assign ke `teacher id=142` (akun `ujicoba_guru`, dev DB) untuk verifikasi
end-to-end via Playwright — login, screenshot top-bar mobile (390x844), drawer mobile, top-bar
desktop (1440x900). Semua 3 titik konfirmasi tampil "Guru Uji Coba Jurnal" (bukan
"ujicoba_guru") + foto asli di avatar. Data test (foto file + kolom `teachers.foto`)
dibersihkan kembali ke `NULL` setelah verifikasi selesai — tidak ada sisa data uji coba
tertinggal di dev DB.
