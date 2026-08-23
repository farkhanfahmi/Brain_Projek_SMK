# T118 — API: Fix Bug — Registrasi Kartu dengan UID Bekas Nonaktif Selalu Ditolak

## Depends on
Tidak ada dependency teknis. Fix bug di 1 method service, dipakai 2 endpoint (create manual + tap-assign).

## Objective
Kartu yang sudah dinonaktifkan (misal karena hilang) bisa DIAKTIFKAN KEMBALI kalau ditemukan lagi, dengan cara didaftarkan ulang memakai UID yang sama — SELAMA pemiliknya (siswa/guru) SAMA seperti sebelumnya. Kalau pemiliknya beda, tetap ditolak, tapi pesan error harus menyebutkan nama pemilik asli kartu itu.

## Context
- **App:** `apps/api` (fix logic murni, tidak ada perubahan skema).
- **Bug dikonfirmasi 2026-08-06 (Explore agent, baca kode langsung)**: `CardsService.create()` (`apps/api/src/cards/cards.service.ts:140-164`) memanggil `ensureUidAvailable(uid)` (baris 248-255) SEBELUM insert. Method itu:
  ```ts
  private async ensureUidAvailable(uid: string) {
    const existing = await this.prisma.card.findUnique({ where: { uid } });
    if (existing) {
      throw new ConflictException(
        `UID ${uid} sudah pernah didaftarkan sebelumnya — UID tidak bisa dipakai ulang meski kartu lama sudah nonaktif`,
      );
    }
  }
  ```
  **Ini BUKAN bug tak disengaja** — pesan errornya eksplisit menyatakan kebijakan "tidak bisa dipakai ulang meski nonaktif". Kebijakan itu yang sekarang diubah oleh task ini.
  - `Card.uid` adalah `String @unique` di schema (`schema.prisma:238`) — constraint DB memang mencegah 2 baris `Card` dengan UID sama, ini TIDAK berubah (tetap 1 baris per UID, direaktivasi bukan dibuat baris baru).
  - **Nonaktifkan kartu = soft-delete**, bukan hapus baris — `revoke(id)` (`cards.service.ts:166-173`) cuma `UPDATE status=inactive, revokedAt=now()`. Baris `Card` lama TETAP ADA di DB, itu sebabnya `ensureUidAvailable` masih menemukannya dan menolak.
  - Endpoint yang terpengaruh: `create()` (registrasi manual UID) DAN `tapAssign()` (`cards.service.ts:132-138`, tap fisik) — **keduanya memanggil `create()` yang sama**, jadi fix di 1 tempat otomatis berlaku untuk keduanya (form Registrasi Kartu hasil T106 pakai kedua jalur ini, tap maupun input manual).

## Keputusan Final (dikonfirmasi user 2026-08-06)

1. **Pemilik SAMA** (UID kartu nonaktif didaftarkan lagi untuk `studentId`/`teacherId` yang PERSIS SAMA seperti pemilik kartu nonaktif itu) → **REAKTIVASI** baris `Card` yang sama: `status: active`, `revokedAt: null`, timestamp lain di-update sesuai kebutuhan (cek field lain yang relevan di model, misal `issuedAt` — putuskan apakah di-reset ke waktu reaktivasi atau dipertahankan nilai asli, kemungkinan besar masuk akal di-update ke waktu reaktivasi karena ini "pendaftaran ulang").
2. **Pemilik BEDA** (UID sama tapi `studentId`/`teacherId` baru berbeda dari pemilik kartu nonaktif itu) → **TETAP DITOLAK** (`ConflictException`), TAPI pesan error harus menyebutkan **nama pemilik asli** kartu itu, contoh: `"UID {uid} sudah terdaftar untuk {nama pemilik asli} (nonaktif) — tidak bisa didaftarkan ke pemilik berbeda. Kalau kartu ini memang milik {nama pemilik asli}, daftarkan ulang untuk siswa/guru yang sama untuk mengaktifkan kembali."`
3. **Kasus nyata yang melatarbelakangi**: kartu hilang → admin nonaktifkan → kartu ketemu lagi → admin ingin aktifkan kembali untuk siswa/guru yang sama. Ini alur UTAMA yang harus lancar tanpa halangan.

## Spec Detail

### Backend
- `apps/api/src/cards/cards.service.ts` — modifikasi `create(dto)` (baris 140-164):
  - Ganti pemanggilan `ensureUidAvailable(dto.uid)` dengan logic baru yang MENGEMBALIKAN info kartu existing (bukan cuma validasi boolean/throw) — perlu tahu `existing.status`, `existing.studentId`, `existing.teacherId`, dan nama pemilik asli (join ke `Student`/`Teacher` untuk resolve nama).
  - Alur baru:
    1. `findUnique({ where: { uid: dto.uid }, include: { student: true, teacher: true } })`.
    2. Kalau TIDAK ADA existing → lanjut alur `create()` normal seperti sekarang (tidak berubah).
    3. Kalau ADA existing DAN `status === "active"` → tetap tolak seperti sekarang (kartu aktif tidak boleh direbut sama sekali, ini beda kasus dari kartu nonaktif — pastikan behavior ini TIDAK berubah, cuma kasus `inactive` yang berubah).
    4. Kalau ADA existing DAN `status === "inactive"`:
       - Kalau `existing.studentId === dto.studentId` (untuk siswa) ATAU `existing.teacherId === dto.teacherId` (untuk guru), DAN jenis pemilik yang sama (bukan kartu siswa lalu didaftarkan ke guru meski ID kebetulan sama) → **REAKTIVASI**: `prisma.card.update({ where: { id: existing.id }, data: { status: "active", revokedAt: null, ... } })`, return kartu yang direaktivasi (bukan insert baru).
       - Kalau BEDA pemilik → `ConflictException` dengan pesan menyebutkan nama pemilik asli (resolve dari `existing.student.nama` atau `existing.teacher.nama`, sesuaikan field nama yang benar di model — cek dulu).
  - **Hati-hati double-check `ensureOwnerExistsAndHasNoActiveCard`** (dipanggil sebelum `ensureUidAvailable` di alur lama, baris ±142) — pastikan fungsi ini TIDAK bentrok dengan alur reaktivasi baru (misal kalau siswa yang SAMA itu ternyata sedang punya kartu AKTIF LAIN dengan UID berbeda — apakah reaktivasi kartu lama tetap diizinkan sambil siswa itu juga punya kartu aktif lain? Ini edge case yang perlu diputuskan: kemungkinan besar TIDAK masuk akal 1 siswa punya 2 kartu aktif sekaligus, jadi validasi existing "no active card" ini harus TETAP jalan sebelum reaktivasi diizinkan — cek urutan pengecekan supaya konsisten).

### Tidak Perlu Diubah
- `tapAssign()` — otomatis ikut fix karena memanggil `create()` yang sama, tidak perlu modifikasi terpisah.
- `revoke()` — tidak berubah, tetap soft-delete seperti sekarang.
- Skema Prisma — tidak ada migration baru, fix ini murni logic service.

## Edge Cases
- Kartu nonaktif untuk SISWA, lalu didaftarkan ulang dengan `dto` yang isinya `teacherId` (bukan `studentId`) meski entah bagaimana ID-nya kebetulan sama → HARUS ditolak (jenis pemilik beda, bukan cuma ID), jangan sampai reaktivasi salah taut ke tipe pemilik yang salah.
- Kartu nonaktif, siswa pemiliknya SUDAH tidak aktif lagi (`Student.status: nonaktif`, misal sudah lulus) → tentukan apakah reaktivasi tetap diizinkan (secara teknis bisa, tapi apakah masuk akal mengaktifkan kartu untuk siswa yang sudah lulus?) — **klarifikasi ke user saat implementasi** kalau kasus ini dirasa perlu ditolak juga, atau dibiarkan (mengaktifkan kartu tidak otomatis mengaktifkan siswa, jadi mungkin tidak masalah — tapi worth dicek).
- Reaktivasi kartu yang PEMILIKNYA sekarang punya kartu aktif LAIN (lihat catatan `ensureOwnerExistsAndHasNoActiveCard` di atas) → harus tetap ditolak konsisten dengan aturan "1 orang 1 kartu aktif" yang sudah ada.

## Files
- **Modifikasi:** `apps/api/src/cards/cards.service.ts` (`create()`, `ensureUidAvailable()` — kemungkinan diganti total jadi method baru yang mengembalikan data lebih kaya, bukan cuma void/throw).
- **Jangan sentuh:** `revoke()`, `replace()`, schema Prisma, `tapAssign()` (otomatis ikut fix tanpa perlu disentuh langsung), `apps/web` (tidak perlu perubahan frontend — pesan error `ConflictException` yang sudah diperbaiki otomatis tampil ke admin lewat mekanisme error handling yang sudah ada, KECUALI kalau dicek dan ternyata frontend butuh penyesuaian tampilan pesan error yang lebih panjang, verifikasi saat implementasi).

## Acceptance Criteria
- [x] Kartu nonaktif didaftarkan ulang dengan UID sama + pemilik SAMA → berhasil, kartu jadi aktif kembali (bukan error). **Diverifikasi live 2026-08-06** terhadap data dev asli (card id 967, UID 2796876377, ADESTYA NURHANA PERTIWI): `status: inactive→active`, `revokedAt: null`, `issuedAt` diperbarui ke waktu reaktivasi.
- [x] Kartu nonaktif didaftarkan ulang dengan UID sama + pemilik BEDA → tetap ditolak, pesan error menyebutkan nama pemilik asli. **Diverifikasi**: `"UID 2796876377 sudah terdaftar untuk ADESTYA NURHANA PERTIWI (nonaktif) — tidak bisa didaftarkan ke pemilik berbeda. Kalau kartu ini memang milik ADESTYA NURHANA PERTIWI, daftarkan ulang untuk siswa/guru yang sama untuk mengaktifkan kembali."`
- [x] Kartu AKTIF tidak bisa direbut UID-nya oleh siapa pun (behavior lama tidak berubah). **Diverifikasi**: pesan baru juga menyebut nama pemilik asli (peningkatan dari pesan generik lama), status "(aktif)" dibedakan dari "(nonaktif)".
- [x] UID benar-benar baru (belum pernah terdaftar sama sekali) tetap bisa didaftarkan normal seperti sekarang (regresi nol). **Diverifikasi**: create baru sukses normal.
- [x] Aturan "1 pemilik 1 kartu aktif" tetap dijaga saat reaktivasi — `ensureOwnerExistsAndHasNoActiveCard` tetap dipanggil SEBELUM logic reaktivasi baru, urutan tidak diubah.
- [x] Build + type-check `apps/api` hijau (`tsc --noEmit`, `nest build` bersih), jest 183/183 tetap lulus (tidak ada spec file khusus `CardsService` sebelumnya, tidak ditambah — cukup diverifikasi live).

## Validasi Claudian
- [x] Verifikasi field nama: `Student.nama` dan `Teacher.nama` dikonfirmasi benar via schema + hasil query live.
- [x] Urutan `ensureOwnerExistsAndHasNoActiveCard` (baris 142, TIDAK dipindah) tetap sebelum resolusi `existing` UID — reaktivasi tidak bisa lolos kalau pemilik baru sudah punya kartu aktif lain, karena exception dilempar duluan di situ sebelum logic UID sempat jalan.
- [ ] Kasus siswa/guru pemilik berstatus nonaktif (misal sudah lulus) saat reaktivasi diminta — **TIDAK diberi validasi tambahan**, dibiarkan tetap bisa direaktivasi (mengaktifkan kartu tidak otomatis mengaktifkan siswa/guru itu sendiri, jadi tidak masalah secara data). Tidak ditemukan alasan kuat untuk menolak saat implementasi, tidak dieskalasi ke user karena bukan bagian dari 3 keputusan final yang sudah dikonfirmasi di awal task.

## Status Eksekusi — SELESAI (2026-08-06)
`apps/api/src/cards/cards.service.ts` — `create()` diubah total: `ensureUidAvailable()` (void/throw) diganti alur baru yang `findUnique` existing card dulu (include nama student/teacher), lalu branch reaktivasi vs tolak vs create normal. `ensureUidAvailable()` privat lama TETAP ADA (tidak dihapus) karena masih dipakai `replace()` sesuai spec ("Jangan sentuh replace()"). Tidak ada perubahan skema, tidak ada perubahan frontend (pesan error lebih panjang otomatis tampil lewat error handling existing). Diverifikasi live terhadap data dev asli (bukan cuma unit test), lalu data dikembalikan ke state semula setelah test.
