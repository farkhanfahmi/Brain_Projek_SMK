# T087 — API+UI: TTL Refresh Token Per-Role (Guru/Kepsek Panjang, Guru Piket Expire Jam 18:00)

## Depends on
Tidak ada — extend mekanisme JWT refresh yang sudah ada (`auth.service.ts`, `middleware.ts`), infrastruktur auto-refresh silent SUDAH LENGKAP (middleware sudah proaktif refresh access token tiap request, tidak perlu dibangun dari nol).

## Context
- **App:** `apps/api` + `apps/web`
- **File:** `apps/api/src/auth/auth.service.ts`, `apps/api/src/auth/auth.types.ts`, `apps/web/src/lib/session.ts`
- **Ref:** Diminta user 2026-07-26 — guru & kepsek ingin TIDAK perlu login ulang setiap buka aplikasi lagi (device yang sama). Guru piket sebaliknya: komputer piket dipakai bergantian tiap hari (shift), WAJIB auto-logout jam 18:00 supaya device "bersih" untuk shift piket besok, tidak ada sesi lama yang nyangkut.

## Spec Detail

### Keputusan Final (dikonfirmasi user, 2026-07-26)
- **"Tidak ada expiry" untuk guru & kepsek** diimplementasikan sebagai **refresh token berumur SANGAT PANJANG (1 tahun)** + auto-refresh silent yang SUDAH ADA di `middleware.ts` — BUKAN literal JWT tanpa field `exp` sama sekali (praktik tidak aman: token/device dicuri tidak akan pernah kadaluarsa sendiri). Tetap bisa di-revoke manual oleh admin kapan saja (mekanisme blacklist Redis yang sudah ada, tidak berubah).
- **Guru piket: refresh token expire TEPAT jam 18:00 di HARI YANG SAMA dia login** — berapapun jam dia login (pagi/siang), token mati jam 18:00 hari itu juga. Bukan durasi tetap (mis. "8 jam dari login"), tapi jam mutlak.

### Backend — `auth.types.ts` & `auth.service.ts`
1. Tambah konstanta baru di `auth.types.ts`:
   ```typescript
   export const REFRESH_TOKEN_TTL_LONG_SECONDS = 365 * 24 * 60 * 60; // 1 tahun — guru, kepsek, admin, admin_jurnal
   ```
   (`REFRESH_TOKEN_TTL_SECONDS` (7 hari) yang lama TETAP ADA — kemungkinan masih relevan untuk role lain yang tidak disebut user, mis. `super_admin`/`card_admin`, KONFIRMASI dulu ke user role mana saja yang termasuk "guru & kepsek" secara harfiah vs yang dimaksud lebih luas "semua akun non-piket". Default aman: HANYA ubah `guru` dan `kepsek` dulu sesuai permintaan literal, JANGAN diam-diam ubah role lain tanpa scope eksplisit.)
2. **Hitung TTL guru_piket secara dinamis** (bukan konstanta tetap, karena jam 18:00 relatif terhadap waktu SEKARANG saat login):
   ```typescript
   function computePiketRefreshTtl(): number {
     const now = new Date();
     const today18 = new Date(now.getFullYear(), now.getMonth(), now.getDate(), 18, 0, 0);
     if (now >= today18) {
       // Login SETELAH jam 18:00 -- KEPUTUSAN FINAL (dikonfirmasi user 2026-07-26):
       // expire SEGERA/sangat singkat di hari yang SAMA, BUKAN digeser ke besok jam 18:00.
       // Piket yang login telat sore/malam sengaja dapat sesi nyaris tidak berguna --
       // mendorong dia lapor/tunggu shift berikutnya login ulang normal, bukan "menembus"
       // aturan 18:00 dengan cara login telat.
       return 60; // grace period singkat (1 menit) supaya request yang sedang jalan tidak gagal di tengah, TAPI sesi berikutnya WAJIB login baru
     }
     return Math.floor((today18.getTime() - now.getTime()) / 1000);
   }
   ```
3. `issueTokenPair()` — ganti logic pemilihan `refreshTtl`:
   ```typescript
   const refreshTtl =
     role === UserRole.guru_piket ? computePiketRefreshTtl()
     : role === UserRole.kepsek || role === UserRole.guru ? REFRESH_TOKEN_TTL_LONG_SECONDS
     : REFRESH_TOKEN_TTL_SECONDS;
   ```
4. **Efek berantai ke `refresh()`**: setiap kali refresh dipanggil (termasuk auto-refresh silent middleware), `issueTokenPair()` dipanggil ULANG dengan role yang sama — untuk guru_piket ini PENTING: kalau piket refresh token di jam 14:00, TTL baru dihitung ulang "sisa waktu sampai 18:00 hari ini" (BUKAN direset ke penuh) — perilaku ini OTOMATIS BENAR dari `computePiketRefreshTtl()` yang selalu hitung dari `now`, TIDAK perlu logic tambahan, tapi WAJIB diverifikasi lewat test bahwa refresh di siang hari TIDAK memperpanjang sesi piket melewati jam 18:00.

### Frontend — `apps/web/src/lib/session.ts`
- `REFRESH_TOKEN_MAX_AGE` (cookie maxAge) di-hardcode 30 hari untuk SEMUA role — ini HARUS jadi dinamis juga, atau cookie browser akan menyimpan refresh token guru_piket lebih lama dari yang backend anggap valid (cookie tidak terhapus otomatis meski JWT sudah expired di server — tidak berbahaya secara keamanan karena backend tetap menolak token expired, tapi janggal/membingungkan kalau di-debug nanti).
- Opsi termudah: `login/route.ts` (route handler yang set cookie) sudah TAHU `role` dari response backend — set `maxAge` cookie mengikuti breakdown yang sama seperti backend (1 tahun untuk guru/kepsek, sisa-waktu-ke-18:00 untuk guru_piket, 7 hari untuk lainnya). Cek apakah response login API sudah expose durasi TTL sesungguhnya, atau perlu backend kirim `expiresIn` di response body supaya frontend tidak duplikasi logic perhitungan jam 18:00.

## JANGAN
- ❌ JANGAN implementasikan sebagai JWT literal tanpa `exp` claim — tetap harus ada masa berlaku (1 tahun), sekalipun praktis terasa "tidak pernah expired" untuk penggunaan normal
- ❌ JANGAN ubah TTL role lain (`super_admin`, `card_admin`, `admin_jurnal`) tanpa konfirmasi eksplisit — scope literal permintaan user HANYA "guru" dan "kepsek" untuk umur panjang, "guru_piket" untuk expire jam 18:00
- ❌ JANGAN hilangkan mekanisme revoke/blacklist manual admin yang sudah ada — refresh token umur panjang TETAP harus bisa dicabut paksa kapan saja (logout dari device manapun, atau reset password)
- ❌ JANGAN improvisasi sendiri soal edge case "piket login setelah jam 18:00" — WAJIB tanya user dulu sebelum implementasi (lihat catatan di atas)

## Files
- **Modifikasi:** `apps/api/src/auth/auth.types.ts` — konstanta TTL baru
- **Modifikasi:** `apps/api/src/auth/auth.service.ts` — `issueTokenPair()` pilih TTL berdasarkan role, fungsi baru `computePiketRefreshTtl()`
- **Modifikasi:** `apps/web/src/lib/session.ts` — `REFRESH_TOKEN_MAX_AGE` jadi fungsi/dinamis per-role
- **Modifikasi:** `apps/web/src/app/api/auth/login/route.ts` — set cookie maxAge sesuai role (cek dulu apakah backend perlu expose `expiresIn` di response)

## Acceptance Criteria
- [ ] Login sebagai guru/kepsek → refresh token JWT `exp` claim menunjukkan ~1 tahun ke depan
- [ ] Login sebagai guru_piket jam 09:00 → refresh token `exp` claim menunjukkan hari yang sama jam 18:00 (bukan 24 jam dari login)
- [ ] Guru_piket yang refresh token-nya di jam 14:00 (sesi sudah jalan) → token baru TETAP expire jam 18:00 hari itu (tidak diperpanjang ke jam 18:00 besok)
- [ ] Guru/kepsek yang tidak buka aplikasi selama beberapa minggu → saat buka lagi tetap otomatis masuk tanpa login ulang (asalkan belum lewat 1 tahun & tidak di-revoke admin)
- [ ] Admin tetap bisa paksa logout akun manapun (guru/kepsek/piket) kapan saja lewat mekanisme existing
- [ ] `jest` unit test baru untuk `computePiketRefreshTtl()` mencakup: login pagi, login sore mendekati 18:00, login setelah 18:00 (sesuai keputusan edge case yang dikonfirmasi user)
