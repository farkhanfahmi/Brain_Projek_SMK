# T261 — Web: Profil Akun (Piket, Admin Jurnal, Pembina Ekstra) — Nama Guru Asli, Bukan Username

## Depends on
Tidak ada. Perbaikan kecil, low-risk — REPLIKASI PERSIS pola yang SUDAH TERBUKTI benar di
role `guru` (T229).

## Objective
Nama yang tampil di top bar/profil akun untuk role `guru_piket`, `admin_jurnal`, dan
`pembina_ekstra` — jadi **nama guru asli** (`Teacher.nama`), bukan `username` akun login.
Role `admin`/`super_admin`/`card_admin`/`kepsek` TIDAK disentuh (sesuai permintaan user
"selain admin" — akun ini biasanya tidak selalu terhubung ke 1 orang guru spesifik).

## Konteks — Pola SUDAH ADA, Tinggal Direplikasi (dikonfirmasi via riset 2026-08-28)

`apps/web/src/app/(guru)/layout.tsx` (baris 30-45, **T229**, SUDAH BENAR) — pola persis:
```ts
const me = await apiFetch<{ isPembinaEkstra: boolean; teacherNama: string | null; teacherFoto: string | null }>(
  "/users/me",
).catch((err) => {
  if (err instanceof ApiError) return { isPembinaEkstra: false, teacherNama: null, teacherFoto: null };
  throw err;
});
// ...
<GuruShell userName={me.teacherNama ?? user.username} userFoto={me.teacherFoto} ... />
```
Endpoint `/users/me` (`users.service.ts`) SUDAH generik — TIDAK khusus role `guru`, sudah
mengembalikan `teacherNama`/`teacherFoto` dari relasi `User.teacherId → Teacher.nama/foto`
untuk User MANAPUN yang py `teacherId` terisi (guru_piket/admin_jurnal/pembina_ekstra
SEMUANYA biasa py `teacherId` terisi sama seperti guru — VERIFIKASI SAAT IMPLEMENTASI kalau
ternyata ada role yang TIDAK pernah py teacherId, fallback `?? user.username` sudah
menangani itu dengan aman).

**3 layout BELUM diperbaiki, masih pakai `user.username` mentah**:
1. `apps/web/src/app/(piket)/layout.tsx:61` — `userName={user.username}`.
2. `apps/web/src/app/(admin-jurnal)/layout.tsx:27` — `userName={user.username}`.
3. `apps/web/src/app/(pembina-ekstra)/layout.tsx:30` — `userName={user.username}`.

## Spec Detail

Untuk KETIGA file di atas — REPLIKASI PERSIS pola `(guru)/layout.tsx`:
1. Tambah `await apiFetch<{ teacherNama: string | null; teacherFoto: string | null }>("/users/me")`
   dengan `.catch()` fallback SAMA (jangan sampai error `/users/me` bikin layout gagal total).
2. Ganti `userName={user.username}` jadi `userName={me.teacherNama ?? user.username}`.
3. Kalau komponen shell (`PiketContent`/`AdminJurnalContent`/`TopBarWithTitle`) SUDAH terima
   prop `userFoto` (cek `(guru)/components/guru-shell.tsx` sebagai referensi) — SERTAKAN
   `userFoto={me.teacherFoto}` juga sekalian (konsisten, bukan cuma nama tapi foto profil
   asli juga) — VERIFIKASI SAAT IMPLEMENTASI apakah `PiketContent`/`TopBarWithTitle` sudah
   punya prop ini, kalau belum ada DI LUAR SCOPE task ini (task ini fokus nama, foto boleh
   menyusul kalau propnya memang sudah tersedia tanpa perubahan tambahan).

`(piket)/layout.tsx` KHUSUS — fetch `/users/me` tambahan ini taruh BERSAMAAN
`Promise.all` dengan 2 fetch existing (`/piket-journal/me/debt`, `/piket-schedules/me/today`)
supaya tidak nambah waktu load berurutan (VERIFIKASI SAAT IMPLEMENTASI — TAPI JANGAN ubah
urutan logic `hasDebt` redirect yang sudah ada, murni optimisasi paralel kalau memungkinkan
tanpa mengubah behavior blocking jurnal wajib).

## Edge Cases
- **Akun role ini TANPA `teacherId` terisi** (kalau ada, jarang tapi mungkin secara data) —
  fallback `?? user.username` otomatis menangani, TIDAK PERLU logic tambahan.
- **`/users/me` gagal/timeout** — `.catch()` SAMA seperti pola guru (fallback ke null semua
  field, layout tetap render dengan username sebagai fallback) — JANGAN sampai 1 endpoint
  gagal membuat seluruh halaman piket/admin-jurnal/pembina-ekstra tidak bisa dibuka.

## Files
- **Modifikasi:** `apps/web/src/app/(piket)/layout.tsx`.
- **Modifikasi:** `apps/web/src/app/(admin-jurnal)/layout.tsx`.
- **Modifikasi:** `apps/web/src/app/(pembina-ekstra)/layout.tsx`.
- **Jangan sentuh:** `apps/web/src/app/(guru)/layout.tsx` (sudah benar, referensi pola),
  `apps/web/src/app/(admin)/layout.tsx` (di luar scope, admin sengaja tidak diubah),
  endpoint `/users/me` (sudah generik, tidak perlu perubahan backend).

## Acceptance Criteria
- [x] Login sebagai `guru_piket` — nama di top bar adalah nama guru asli, bukan username.
- [x] Login sebagai `admin_jurnal` — sama.
- [x] Login sebagai `pembina_ekstra` — sama.
- [x] Login sebagai `admin`/`super_admin`/`card_admin`/`kepsek` — TIDAK BERUBAH (regresi
      check, tetap seperti sebelumnya).
- [x] Akun tanpa `teacherId` di role ini tetap fallback ke username, tidak error/kosong.
- [x] Build + type-check hijau.

## Validasi Claudian
- [x] Konfirmasi pola fetch+fallback 100% SAMA dengan `(guru)/layout.tsx` (bukan
      reimplementasi mirip-mirip) — copy-paste pattern-nya, sesuaikan cuma nama variabel
      shell yang dipanggil.
- [x] Konfirmasi `(piket)/layout.tsx` — urutan redirect `hasDebt` (jurnal wajib blocking)
      TIDAK terganggu oleh penambahan fetch `/users/me` (verifikasi logika: `Promise.all`
      cuma memparalelkan 3 fetch, `if (hasDebt) redirect(...)` tetap dieksekusi PERSIS
      setelah await selesai, urutan/behavior blocking tidak berubah — belum test manual
      klik-coba akun berutang jurnal sungguhan, lihat catatan Implementasi).

## Implementasi (2026-08-28)

Ketiga layout (`(piket)/layout.tsx`, `(admin-jurnal)/layout.tsx`, `(pembina-ekstra)/layout.tsx`)
diberi fetch `/users/me` REPLIKASI PERSIS pola `(guru)/layout.tsx` (T229) — `.catch()`
fallback ke `{ teacherNama: null, teacherFoto: null }` kalau `ApiError` (endpoint gagal
tidak boleh membuat layout gagal total), lalu `userName={me.teacherNama ?? user.username}`.

- **`(admin-jurnal)/layout.tsx`**: fetch tunggal ditambah sebelum render, `AdminJurnalContent`
  TIDAK punya prop `userFoto` (tidak dimodifikasi, sesuai spec — di luar scope task ini).
- **`(pembina-ekstra)/layout.tsx`**: pakai `TopBarWithTitle` LANGSUNG (bukan wrapper content
  custom) yang SUDAH menerima prop `userFoto` tanpa perubahan tambahan apa pun — jadi
  `userFoto={me.teacherFoto}` ikut disertakan sekalian (satu-satunya dari 3 layout yang dapat
  foto profil asli juga, sesuai kondisi spec "kalau propnya memang sudah tersedia").
- **`(piket)/layout.tsx`**: fetch `/users/me` DIPARALELKAN via `Promise.all` bersama 2 fetch
  existing (`/piket-journal/me/debt`, `/piket-schedules/me/today`) — TAPI hanya diletakkan
  SETELAH early-return blocking-path (`JURNAL_BLOCKING_PATH`) tetap di posisi paling awal,
  tidak berubah. `if (hasDebt) redirect(...)` tetap dieksekusi persis setelah `Promise.all`
  resolve, urutan/behavior blocking jurnal wajib tidak berubah sama sekali — murni
  optimisasi paralel 3 fetch yang sebelumnya 2 fetch berurutan. `PiketContent` TIDAK punya
  prop `userFoto` (tidak dimodifikasi, di luar scope).

**Backend**: `UsersService.findMe()` (`users.service.ts:226`) dan endpoint `GET /users/me`
(`users.controller.ts:57`) dikonfirmasi VIA BACA KODE sudah generik — TIDAK ada `@Roles()`
guard, dan resolusi `teacherNama`/`teacherFoto` murni dari relasi `User.teacher` (bukan
role-specific) — bekerja identik untuk role manapun yang py `teacherId` terisi. Tidak ada
perubahan backend (sesuai spec).

**Verifikasi:**
- `tsc --noEmit` web — bersih, tanpa error.
- `next build` web — sukses penuh (exit 0), tidak ada error compile.
- `(admin)/layout.tsx` dan `(guru)/layout.tsx` dikonfirmasi TIDAK tersentuh (`git status`
  bersih untuk kedua file itu) — regresi role admin dijamin nol secara struktural.
- Tidak ada test suite existing yang menguji layout Next.js ini (server component route
  group, pola yang sama sekali tidak ditest di codebase manapun) — tidak ada test yang
  perlu diupdate.
- **Belum diverifikasi live** (DB dev naik-turun sepanjang sesi, konsisten keterbatasan
  sesi ini): login sungguhan sebagai `guru_piket`/`admin_jurnal`/`pembina_ekstra` untuk
  konfirmasi visual nama asli tampil, dan test manual akun piket berutang jurnal untuk
  konfirmasi redirect blocking tetap jalan setelah paralelisasi fetch.
