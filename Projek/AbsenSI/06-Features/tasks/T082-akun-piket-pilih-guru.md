# T082 — API+UI: Field Pilih Guru saat Membuat Akun Piket

## Depends on
Tidak ada — pola UI dan validasi backend sudah ada persis untuk `role: "guru"` di file yang sama, tinggal direplikasi untuk `role: "guru_piket"`.

## Context
- **App:** `apps/api` + `apps/web`
- **File:** `apps/web/src/app/(admin)/akun/akun-view.tsx`, backend user/auth module terkait
- **Ref:** Diminta user 2026-07-24 — saat membuat akun `guru_piket`, form sekarang cuma minta pilih Kampus (`akun-view.tsx:340-356`), TIDAK ada field pilih guru dari data Guru yang sudah ada. User ingin field ini WAJIB, sama seperti pola `role: "guru"` yang sudah punya dropdown Teacher (`akun-view.tsx:322-338`).

## Spec Detail

### Keputusan Final (dikonfirmasi user)
- Field pilih Guru untuk akun `guru_piket` **WAJIB diisi**, sama seperti akun `guru` biasa — bukan opsional.

### Backend
- Cek skema `User` — field `teacherId` KEMUNGKINAN BESAR sudah ada (dipakai `role: "guru"`), tinggal pastikan validasi DTO create/update user MEWAJIBKAN `teacherId` juga ketika `role === "guru_piket"` (sekarang kemungkinan cuma wajib untuk `role === "guru"` — cek DTO `create-user.dto.ts`/`update-user.dto.ts`, tambahkan `@ValidateIf` untuk `guru_piket` juga)
- **Pertimbangkan**: 1 guru bisa jadi piket DAN py guru biasa sekaligus (2 akun berbeda, `teacherId` sama) — TIDAK perlu unique constraint `teacherId` per user, itu sudah benar kalau memang belum ada constraint semacam itu (cek dulu, jangan tambahkan pembatasan baru yang tidak diminta)

### UI
- `akun-view.tsx` — ubah kondisi render (baris ~340) dari HANYA `role === "guru_piket"` menampilkan dropdown Kampus, jadi tampilkan **DUA field**: dropdown Guru (sama persis komponen di baris 322-338, cukup ubah kondisi `role === "guru" || role === "guru_piket"`) DAN dropdown Kampus yang sudah ada (piket tetap butuh kampus assignment, itu tidak berubah)
- Validasi client-side: submit button disabled kalau `role === "guru_piket"` tapi `teacherId` kosong (pola sama seperti guru biasa)

## JANGAN
- ❌ JANGAN hapus field Kampus untuk `guru_piket` — piket tetap butuh kampus assignment (fitur existing, tidak diubah), field Guru ini TAMBAHAN bukan pengganti
- ❌ JANGAN buat constraint 1-guru-1-akun — satu Teacher boleh punya akun guru DAN akun guru_piket terpisah kalau memang begitu keadaannya di sekolah

## Files
- **Modifikasi:** `apps/web/src/app/(admin)/akun/akun-view.tsx` — extend kondisi render dropdown Guru untuk `guru_piket`
- **Modifikasi (cek dulu, mungkin sudah cukup):** `apps/api/src/core/users/dto/create-user.dto.ts`, `update-user.dto.ts` (atau lokasi setara) — validasi `teacherId` wajib untuk `guru_piket`

## Acceptance Criteria
- [ ] Pilih role "Guru Piket" saat buat akun baru → muncul 2 dropdown: Guru dan Kampus, keduanya wajib
- [ ] Submit tanpa pilih Guru → ditolak (client-side disabled dan/atau backend 400)
- [ ] Akun guru_piket yang sudah ada bisa di-edit untuk isi/ubah field Guru-nya (mode edit form sama)
- [ ] Backend menolak `POST /users` dengan `role: guru_piket` tanpa `teacherId` (400)
