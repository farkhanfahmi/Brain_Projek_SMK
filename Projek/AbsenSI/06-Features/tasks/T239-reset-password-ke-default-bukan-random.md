# T239 — API+Web: Reset Password Admin — Kembalikan ke Password Default, Bukan Random

## Depends on
Tidak ada dependency teknis. Independen, murni modul `users`.

## Konteks — Kondisi Kode Saat Ini (dikonfirmasi via riset 2026-08-21)

**Tidak ada mekanisme "token" reset password (link/kode terpisah) di sistem ini** — dikonfirmasi grep menyeluruh (`resetToken`, `forgot-password`, dsb — 0 hasil, skema `User` tidak punya kolom token apa pun). Yang user maksud "token" adalah **password ACAK 10 karakter** yang dihasilkan `resetPassword()` dan ditampilkan admin di dialog — bentuknya kode acak (`generatePassword()`, `customAlphabet(...)`) sehingga terlihat seperti "token", padahal itu password sungguhan yang harus disampaikan manual ke pemilik akun.

`UsersService.resetPassword()` (`apps/api/src/users/users.service.ts:148-159`) — KONDISI SEKARANG:
```ts
async resetPassword(id: number): Promise<{ username: string; newPassword: string }> {
  const user = await this.ensureExists(id);
  const newPassword = generatePassword(); // RANDOM 10 karakter
  const passwordHash = await bcrypt.hash(newPassword, SALT_ROUNDS);
  await this.prisma.user.update({ where: { id }, data: { passwordHash, mustChangePassword: true } });
  return { username: user.username, newPassword };
}
```
`generatePassword` (baris 16): `customAlphabet("abcdefghjkmnpqrstuvwxyzACDEFGHJKMNPQRSTUVWXYZ23456789", 10)` dari `nanoid`.

**Password default `"12345678"` SUDAH dipakai** di `generateAkunGuruMassal()` (T232, `users.service.ts:235-269`) — TAPI sebagai **string literal langsung** (BUKAN konstanta bernama/exported), muncul 2x di file yang sama, tidak reusable dari method lain saat ini.

**`setPassword()` (`users.service.ts:161-171`) — method TERPISAH, TIDAK terdampak task ini** — untuk kasus admin set password MANUAL (bukan default/random), tetap seperti sekarang.

**Frontend** (`apps/web/src/app/(admin)/akun/akun-view.tsx:146-158`) — `handleResetPassword()` panggil endpoint, tampilkan `resetResult.newPassword` di dialog ("Password Baru", baris 280-306) dengan pesan "Sampaikan password ini secara manual ke pemilik akun... tidak akan ditampilkan lagi setelah dialog ditutup".

## Keputusan Diminta User (2026-08-21)

Ganti `resetPassword()` — **JANGAN generate password acak lagi**, LANGSUNG set ke **password default** (`"12345678"`, KONSISTEN nilai yang sama dipakai T232) — konsisten juga `mustChangePassword: true` (perilaku ini TIDAK berubah, tetap wajib ganti password di login berikutnya).

## Spec Detail

### 1. Backend — konstanta password default (reusable, extract dari T232)

- TAMBAH konstanta bernama, REKOMENDASI `apps/api/src/users/users.service.ts` (atau shared util kalau ada tempat lebih tepat) — `const DEFAULT_PASSWORD = "12345678";` — REFACTOR `generateAkunGuruMassal()` (T232) SEKALIAN untuk pakai konstanta ini (BUKAN literal terpisah lagi di 2 tempat, KONSISTEN prinsip "1 nilai 1 tempat").

### 2. Backend — `resetPassword()` — ganti random jadi default

```ts
async resetPassword(id: number): Promise<{ username: string; newPassword: string }> {
  const user = await this.ensureExists(id);
  const passwordHash = await bcrypt.hash(DEFAULT_PASSWORD, SALT_ROUNDS);
  await this.prisma.user.update({ where: { id }, data: { passwordHash, mustChangePassword: true } });
  return { username: user.username, newPassword: DEFAULT_PASSWORD };
}
```
- **Response shape TIDAK BERUBAH** (`{ username, newPassword }`) — FE existing TIDAK PERLU diubah sama sekali (dialog "Password Baru" tetap tampilkan nilainya apa adanya, cuma sekarang isinya SELALU `"12345678"` bukan random — VERIFIKASI SAAT IMPLEMENTASI apakah pesan di dialog FE perlu disesuaikan teksnya, misal dari "Password baru untuk akun ini" jadi lebih eksplisit "Password direset ke default: 12345678" — REKOMENDASI: update teks pesan supaya admin paham ini nilai TETAP/dikenal, bukan random yang harus dicatat buru-buru).
- `generatePassword()`/import `customAlphabet` dari `nanoid` — CEK apakah masih dipakai di tempat LAIN di file ini setelah perubahan ini (kalau tidak ada pemakaian lain, hapus import yang jadi tidak terpakai, JANGAN biarkan dead code).

### 3. Frontend — update pesan dialog (opsional tapi direkomendasikan)

`apps/web/src/app/(admin)/akun/akun-view.tsx` — dialog "Password Baru" (baris 280-306) — PERTIMBANGKAN update teks supaya jelas ini password DEFAULT yang dikenal (bukan rahasia acak sekali-lihat) — REKOMENDASI teks: "Password direset ke default (12345678). [Nama guru] wajib menggantinya saat login berikutnya." — TIDAK WAJIB mengubah struktur dialog, cukup teks penjelasan.

## Edge Cases

- **Admin reset password akun yang SAMA berkali-kali** — SELALU hasilnya `"12345678"` lagi (idempotent, TIDAK ADA masalah — behavior yang diinginkan).
- **`setPassword()` (set manual admin) — TIDAK TERDAMPAK task ini sama sekali** — tetap terima input bebas dari admin, TIDAK diubah jadi default paksa.
- **Keamanan**: password default yang SAMA untuk semua reset (dan sama dengan T232 generate massal) — INI KEPUTUSAN SADAR USER (persis sama seperti keputusan T232 sebelumnya), TIDAK PERLU validasi kekuatan password tambahan, DI LUAR SCOPE task ini untuk dipermasalahkan ulang.

## Files
- **Modifikasi:** `apps/api/src/users/users.service.ts` (`resetPassword()`, konstanta `DEFAULT_PASSWORD` shared dengan `generateAkunGuruMassal()`), `apps/web/src/app/(admin)/akun/akun-view.tsx` (opsional, teks dialog).
- **Jangan sentuh:** `setPassword()` (method terpisah, tetap manual), `ForcePasswordChangeConfig`/`mustChangePassword` logic (TIDAK berubah, tetap `true` setelah reset).

## Acceptance Criteria
- [ ] `POST /users/:id/reset-password` — password akun jadi `"12345678"`, BUKAN random lagi.
- [ ] `mustChangePassword` tetap `true` setelah reset (perilaku existing TIDAK berubah).
- [ ] `generateAkunGuruMassal()` (T232) dan `resetPassword()` — SAMA-SAMA pakai 1 konstanta `DEFAULT_PASSWORD`, tidak ada 2 nilai literal terpisah lagi.
- [ ] `setPassword()` (set manual) — TIDAK terdampak, tetap terima input bebas.
- [ ] Login dengan password default setelah reset — berhasil, langsung diarahkan wajib ganti password (`mustChangePassword` redirect existing tetap jalan).
- [ ] Build + type-check hijau, jest baru: reset password hasilnya selalu "12345678", `mustChangePassword` tetap true.

## Validasi Claudian
- [ ] Konfirmasi konstanta `DEFAULT_PASSWORD` SATU sumber dipakai KEDUA method (`resetPassword()` dan `generateAkunGuruMassal()`), bukan 2 literal terpisah yang bisa drift kalau nanti nilai default berubah.
- [ ] Konfirmasi `generatePassword()`/import `nanoid` yang jadi tidak terpakai (kalau memang tidak ada pemakaian lain) DIHAPUS, bukan dibiarkan jadi dead code.
- [ ] Konfirmasi `setPassword()` method TERPISAH sama sekali tidak tersentuh oleh perubahan ini.
