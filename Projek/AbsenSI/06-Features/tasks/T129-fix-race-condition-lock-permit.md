# T129 — API: Fix Race Condition — Lock/Unlock Siswa & Resolusi Izin Bisa Tumpang Tindih

## Depends on
Tidak ada dependency teknis. Fix pola query di 2 service (`students.service.ts`, `permits.service.ts`), tidak ada perubahan skema.

## Objective
Aksi lock/unlock siswa dan resolusi permit (`confirmKembali`/`setPulang`) dilindungi dari race condition — kalau 2 request datang nyaris bersamaan (2 piket beda tab, atau double-klik/double-submit tidak sengaja), tidak boleh keduanya lolos validasi dan saling menimpa tanpa error.

## Context
- **App:** `apps/api` — perbaikan pola query, tidak ada perubahan skema/model.
- **Celah dikonfirmasi 2026-08-06** lewat audit keamanan dashboard piket (Explore agent, baca kode langsung): SEMUA method mutasi state-transition di 2 service ini pakai pola yang SAMA dan RENTAN race:
  1. `findUnique` — baca state saat ini.
  2. Percabangan `if` di level JavaScript — cek apakah state sudah sesuai (misal `lockedAt === null` sebelum lock, atau `statusKembali === "belum"` sebelum resolve).
  3. `prisma.update({ where: { id } })` — TANPA predikat state di `WHERE` clause, tanpa `SELECT ... FOR UPDATE`, tanpa kolom versi (`optimistic locking`).
  - Kalau 2 request datang di antara langkah 1 dan 3 (window race yang nyata, meski singkat), KEDUANYA bisa lolos pengecekan JS sebelum salah satu commit — yang terakhir menulis MENANG diam-diam, tanpa error, berpotensi:
    - **`students.service.ts` `lock()`/`unlock()`** (baris ±441-458, ±473-494): 2 lock bersamaan bisa saling timpa `lockedById`/`lockedReason` (siapa yang benar-benar mengunci jadi ambigu). Race lock+unlock bisa meninggalkan `lockedAt`/`unlockedAt` yang tidak konsisten, dan **2 event `broadcastStudentLock` berbeda ikut terkirim** (relevan untuk T108, notifikasi piket bisa menampilkan sinyal ganda/salah).
    - **`permits.service.ts` `confirmKembali()`/`setPulang()`** (baris ±272-295, ±298-347): 2 piket resolve permit yang sama nyaris bersamaan lewat jalur BERBEDA (satu klik "Sudah Kembali", satu klik "Dianggap Pulang") bisa keduanya lolos cek `statusKembali === "belum"`, menghasilkan state akhir yang field-nya campur dari 2 operasi berbeda (misal `waktuPulang` ke-set dari `setPulang` tapi `statusKembali` akhirnya value dari `confirmKembali` yang menang terakhir) — data tidak konsisten secara logis.
  - `setPulang`/`tandaiIzinTidakKembali` SUDAH pakai `$transaction` (baris ±302, ±381) — TAPI ini cuma membungkus beberapa write JADI SATU (atomicity), TIDAK mencegah request KEDUA melakukan read-then-write yang sama secara paralel (MySQL default REPEATABLE READ tetap mengizinkan write-skew tanpa row lock eksplisit).

## Spec Detail

### Pola Fix yang Direkomendasikan
Ganti pola `findUnique` → `if` di JS → `update({ where: { id } })` MENJADI salah satu:

**Opsi A (direkomendasikan, paling sederhana): `updateMany` dengan predikat state di WHERE**
```ts
const result = await this.prisma.student.updateMany({
  where: { id: studentId, lockedAt: null }, // predikat state MASUK ke WHERE
  data: { lockedAt: new Date(), lockedById, lockedReason },
});
if (result.count === 0) {
  // Tidak ada baris ter-update — berarti sudah locked oleh request lain (race kalah),
  // atau siswa tidak ditemukan. Lempar ConflictException/NotFoundException sesuai kondisi.
  throw new ConflictException("Siswa sudah terkunci (kemungkinan diproses piket lain)");
}
```
`updateMany` dengan `WHERE` yang menyertakan predikat state itu ATOMIK di level database — kalau 2 request race, HANYA SATU yang akan menemukan baris yang match (`count: 1`), yang lain `count: 0` dan harus ditangani sebagai "gagal karena sudah diproses lebih dulu", BUKAN silent overwrite.

**Opsi B (kalau Opsi A tidak cukup untuk kasus tertentu — misal butuh data lain dari row sebelum tahu pesan error yang tepat): `SELECT ... FOR UPDATE` dalam transaction**
- Pakai `$transaction` dengan raw query atau `prisma.$queryRaw` untuk row-level lock eksplisit — lebih kompleks, pertimbangkan HANYA kalau Opsi A tidak cukup ekspresif untuk kasus tertentu.

### Terapkan ke Semua Method Terdampak
- `apps/api/src/core/students/students.service.ts` — `lock()` (predikat `lockedAt: null`), `unlock()` (predikat `lockedAt: { not: null } }` atau field lock-source yang relevan).
- `apps/api/src/permits/permits.service.ts` — `confirmKembali()` dan `setPulang()` (predikat `statusKembali: "belum"` di kedua method, supaya siapa pun yang menang race, yang kalah dapat error jelas alih-alih silent overwrite). Cek juga `tandaiIzinTidakKembali()` kalau pola yang sama berlaku di situ.

### Frontend — Tangani Error Baru dengan Baik
- Kalau backend sekarang bisa melempar `ConflictException` baru untuk kasus race (bukan cuma "sudah dalam state itu" seperti sebelumnya, tapi juga "keduluan piket lain barusan") — pastikan UI piket menampilkan pesan yang masuk akal (misal "Data sudah diproses piket lain, silakan refresh halaman"), BUKAN error generik yang membingungkan. Cek `piket-board-view.tsx` (lock/unlock form) dan halaman terkait permit untuk penanganan error existing, sesuaikan pesan kalau perlu.

## Edge Cases
- Race yang menang vs kalah — pastikan urutan TIDAK penting untuk KEBENARAN hasil akhir (siapa pun yang menang, hasilnya tetap valid, cuma yang kalah dapat error informatif) — bukan soal "siapa yang benar", tapi soal "jangan ada 2 operasi konflik yang diam-diam sukses berbarengan".
- Retry otomatis dari frontend (kalau ada) setelah dapat `ConflictException` — pastikan TIDAK retry buta tanpa re-fetch state terbaru dulu (kalau retry, harus re-cek state sekarang, bukan asumsi state lama masih berlaku).

## Files
- **Modifikasi:** `apps/api/src/core/students/students.service.ts` (`lock()`, `unlock()`), `apps/api/src/permits/permits.service.ts` (`confirmKembali()`, `setPulang()`, cek `tandaiIzinTidakKembali()`), kemungkinan `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (penanganan pesan error baru).
- **Jangan sentuh:** skema Prisma (tidak perlu kolom versi/optimistic-lock baru kalau Opsi A `updateMany`-predikat sudah cukup).

## Acceptance Criteria
- [x] 2 permintaan lock BERSAMAAN ke siswa yang sama — HANYA SATU yang berhasil, yang lain dapat error jelas. **Diverifikasi live**: 10 request `POST /students/300/lock` sungguhan bersamaan (paralel shell background job, bukan sequential) → 1×201, 9×409 dengan pesan "Siswa ini sudah terkunci (kemungkinan diproses piket lain)". State DB akhir dicek, bersih (1 lock valid, tidak ada anomali).
- [x] 2 permintaan resolve permit BERSAMAAN (confirmKembali vs setPulang) ke permit yang sama — HANYA SATU yang berhasil, state akhir konsisten. **Diverifikasi live — kasus PALING kritis**: 5×`PATCH /permits/2/confirm-kembali` + 5×`PATCH /permits/2/set-pulang` (2 jalur BERBEDA) ditembak bersamaan ke permit yang sama → 1×200 (confirm-kembali menang), 9×409. State akhir `statusKembali: sudah` bersih (BUKAN campuran field dari setPulang), dan `attendanceRecord` TIDAK tersentuh sama sekali (setPulang yang kalah dibatalkan SEBELUM sempat menulis).
- [x] Kasus NORMAL (1 request, tidak ada race) tetap berfungsi identik. Dibuktikan implisit dari test race di atas (1 dari N selalu berhasil dengan behavior sama seperti kasus tunggal) + 192/192 jest existing tetap lulus.
- [x] Frontend menampilkan pesan error yang masuk akal untuk kasus "kalah race". Dicek: `piket-board-view.tsx` SUDAH konsisten pakai pola `err.message` (dari `ConflictException` backend) di semua titik lock/unlock/permit — pesan baru ("kemungkinan diproses piket lain") otomatis tampil tanpa perlu ubah kode frontend sama sekali.
- [x] Build + type-check `apps/api` hijau (web tidak disentuh, tidak perlu diubah). `tsc --noEmit` bersih, jest 192/192.
- [x] Verifikasi eksplisit dengan concurrent request NYATA (curl paralel shell background job + `wait`, bukan sequential loop) — dilakukan untuk `lock()`, `unlock()`, dan race lintas jalur `confirmKembali()`/`setPulang()`.

## Status Eksekusi — SELESAI (2026-08-07)
**Pola fix**: Opsi A dari spec (predikat state masuk `WHERE` di `updateMany`, atomik di level DB) diterapkan ke SEMUA method terdampak:
- `students.service.ts` `lock()` — predikat `lockedAt: null`, `unlock()` — predikat `lockedAt: { not: null } }`. Keduanya: `update()` diganti `updateMany()`, `count === 0` → `ConflictException`, lalu `findUniqueOrThrow()` terpisah untuk fetch data lengkap (include relasi) buat broadcast+response.
- `permits.service.ts` `confirmKembali()` — predikat `statusKembali: belum`, pola sama. `setPulang()` — predikat SAMA tapi `updateMany` dijalankan DI DALAM `$transaction` yang sudah ada, count dicek SEBELUM lanjut ke write `attendanceRecord` (kalau kalah race, transaction dibatalkan sebelum menyentuh attendance record sama sekali). `tandaiIzinTidakKembali()` — race window BEDA dari 2 method lain (bukan `Permit.statusKembali`, tapi `AttendanceRecord.waktuPulang`, dan operasinya CREATE Permit baru bukan update existing) — predikat `waktuPulang: null` dipindah ke `updateMany` pada `attendanceRecord`, DI DALAM transaction, DICEK SEBELUM `permit.create()` (kalau kalah race, tidak ada Permit duplikat yang ter-create).

**Frontend**: TIDAK ADA perubahan kode — `piket-board-view.tsx` sudah konsisten menampilkan `err instanceof Error ? err.message : "..."` di semua titik terkait (lock, unlock, confirm-kembali/set-pulang), jadi pesan `ConflictException` baru otomatis muncul jelas ke piket tanpa perlu sentuh frontend sama sekali.

**Verifikasi live concurrent request NYATA** (bukan asumsi dari baca kode — WAJIB sesuai spec): dev API, JWT manual guru_piket (perlu insert 1 baris `piket_schedules` sementara supaya lolos `PiketOnDutyGuard`, dihapus lagi setelah test), curl paralel shell background job (`&` + `wait`, bukan loop sequential — supaya request BENAR-BENAR bersamaan di level OS/network, bukan cuma berurutan cepat) ke endpoint sungguhan:
- 10× lock bersamaan → 1×201, 9×409.
- 10× unlock bersamaan → 1×201, 9×409.
- 5× confirm-kembali + 5× set-pulang bersamaan ke permit SAMA → 1×200, 9×409, state akhir bersih (tidak campuran).
Semua data test (jadwal piket sementara, state lock/permit) dibersihkan/dikembalikan setelah verifikasi selesai.

## Validasi Claudian
- [x] Verifikasi concurrent request NYATA dilakukan — bukan cuma review kode. Race condition terbukti benar-benar teratasi lewat observasi status code + state DB, bukan asumsi.
- [x] SEMUA method di Spec Detail diperbaiki dengan pola konsisten: `lock()`, `unlock()`, `confirmKembali()`, `setPulang()`, DAN `tandaiIzinTidakKembali()` (yang terakhir punya race window berbeda struktur tapi prinsip sama — predikat state di `updateMany` WHERE, dicek sebelum operasi lanjutan) — tidak ada method yang terlewat.
