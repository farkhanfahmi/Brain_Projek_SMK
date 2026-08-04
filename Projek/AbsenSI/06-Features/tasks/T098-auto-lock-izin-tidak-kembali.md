# T098 — Schema+API: Auto-Lock Siswa Izin Keluar Tidak Kembali (Amandemen ADR-017) + Hapus "Perlu Ditinjau"

## Depends on
Tidak ada — menyentuh `PermitsService`/`StudentsService`/job scheduler baru, tidak bentrok dengan T090-T097.

## Objective
Menutup celah: siswa yang izin keluar dan tidak kembali (piket lupa/lengah menandai) saat ini TETAP BISA tap masuk normal keesokan harinya karena `lockedAt` masih null. Setelah task ini, sistem otomatis mengunci siswa tersebut lewat tengah malam kalau piket belum menyelesaikan permit-nya (Sudah Kembali / Dianggap Pulang / Tidak Kembali manual) — override eksplisit ADR-017 untuk kasus ini, konsisten dengan preseden ADR-025.

## Context
- **App:** `apps/api`
- **Diskusi lengkap 2026-07-30** — proses klarifikasi panjang, ringkasan keputusan di bawah adalah hasil akhir SETELAH kontradiksi awal diluruskan (draft pertama user minta "tutup celah" TAPI JUGA "tidak kembali = aksi manual piket" — dua hal itu kontradiktif kalau lock cuma terjadi saat piket klik tombol. Resolusi: ada 2 mekanisme lock terpisah, satu manual satu otomatis-terjadwal, dijelaskan di bawah).
- **ADR terkait**: 11-Decisions (11-Decisions.md) ADR-017 (lock harus manual, alasan: risiko human-error salah kunci lebih besar dari manfaat), ADR-025 (preseden override: 2x terlambat auto-lock karena itu fakta objektif dari data tap, beda dari "tidak kembali" yang piket perlu menilai situasi lapangan). **T098 menambah pengecualian KEDUA** pada prinsip ADR-017, harus ditulis sebagai revisi ADR baru (lihat bagian "Update Dokumentasi" di bawah), BUKAN diam-diam menyimpang dari dokumentasi.
- **Job scheduler existing**: `apps/api/src/attendance/end-of-day.service.ts` + `end-of-day.scheduler.ts` (BullMQ, cron `0 18 * * *` = jam 18:00 WIB tiap hari) — HANYA untuk deteksi "belum tap pulang" (T016), TIDAK menulis apapun ke DB, murni logging. T098 butuh job BARU dengan jadwal beda (tengah malam / dini hari, SEBELUM jam masuk sekolah keesokan harinya) karena keputusan "final belum kembali" baru valid setelah hari itu benar-benar berakhir.

## Keputusan Final (dikonfirmasi user 2026-07-30, setelah diskusi panjang)

Model baru untuk resolusi permit `jenis: keluar` (menggantikan pemahaman sebelumnya soal "Belum Kembali"):

1. **3 kondisi resolusi manual oleh piket** (existing 2 + 1 baru):
   - **Sudah Kembali** (`confirmKembali`, sudah ada) — tidak berubah.
   - **Dianggap Pulang** (`setPulang`, sudah ada) — jam pulang = jam keluar, tidak berubah.
   - **Tidak Kembali** (BARU, aksi manual piket — piket klik tombol ini setelah menilai situasi) — begitu diklik, sistem LANGSUNG mengunci siswa (`lockedAt`, `lockedReason` khusus, pola sama seperti lock manual biasa `StudentsService.lock()` TAPI dipicu dari konteks permit, bukan dialog lock generik terpisah). `statusKembali` di permit di-set value baru atau reuse existing (cek enum `StatusKembali` — mungkin perlu tambah value baru `tidak_kembali`, BEDA dari `pulang` yang artinya "dianggap pulang normal").

2. **Auto-lock terjadwal (BARU, job baru, INI yang menutup celah)** — job baru jalan lewat tengah malam (usulkan jam 00:30 atau 05:00 WIB, sebelum jam masuk sekolah pagi — putuskan waktu pasti saat implementasi, pertimbangkan beban server & konsistensi dengan `EndOfDayScheduler` yang sudah ada jam 18:00): query semua `Permit` `jenis: keluar` dengan `tanggal` = KEMARIN (hari yang baru saja berakhir) dan `statusKembali: belum` (piket belum sempat resolve dengan cara manapun) → untuk TIAP siswa itu, **sistem sendiri yang mengunci** (`lockedAt = now()`, `lockedById: null` — pola sama seperti lock otomatis 2x-terlambat ADR-025, BUKAN piket manapun yang mengunci), `lockedReason` khusus dan BERBEDA dari 2x-terlambat, misal: `"Tidak ada konfirmasi kembali dari izin keluar [tanggal] — dikunci otomatis sistem"`.
   - **Guard idempotency**: kalau siswa SUDAH terkunci (`lockedAt !== null`, dari sumber manapun — manual piket, 2x-terlambat, atau auto-lock T098 hari sebelumnya), JANGAN overwrite lock yang sudah ada — skip siswa itu di job ini (`lockedAt !== null` = "sudah tertangani", tidak perlu tumpuk lock, sama pola dengan `applyLateStrikeLock` existing yang cek `student.lockedAt !== null` dulu sebelum lock).

3. **Auto-unlock saat resolve (BARU, WAJIB — supaya piket tidak perlu 2 langkah)** — kalau siswa TERLANJUR ter-auto-lock oleh job T098 (bisa dibedakan dari `lockedById === null` DAN `lockedReason` mengandung penanda khusus T098, BEDA dari lock 2x-terlambat yang juga `lockedById === null` tapi `lockedReason` berbeda — pastikan string/enum pembeda jelas, jangan cuma andalkan text matching yang rapuh, pertimbangkan kolom baru atau prefix terstruktur), lalu keesokan harinya piket akhirnya menemukan siswa itu sebenarnya SUDAH kembali/pulang (misal orang tua konfirmasi terlambat) dan piket klik "Sudah Kembali"/"Dianggap Pulang" pada permit LAMA yang sama — sistem HARUS otomatis `unlock()` siswa itu di aksi yang SAMA (bukan piket harus buka dialog Unlock terpisah). Kalau permit itu TIDAK terkait lock (siswa entah kenapa belum sempat ter-auto-lock, atau sudah di-unlock manual sebelumnya oleh piket lain) — resolve permit berjalan normal tanpa efek samping ke lock.

4. **Section "Perlu Ditinjau" DIHAPUS TOTAL** — dengan mekanisme auto-lock di atas, begitu tengah malam lewat, siswa yang belum diresolve OTOMATIS masuk "Siswa Terkunci" (section existing) — tidak perlu section perantara "Perlu Ditinjau" lagi. User eksplisit menyatakan tidak melihat urgensi section ini setelah mekanisme baru berjalan.

## Spec Detail

### Schema
- `enum StatusKembali` — cek nilai existing (`belum`, `sudah`, `pulang`), tambah `tidak_kembali` untuk kasus piket eksplisit klik "Tidak Kembali" (beda dari auto-lock sistem — permit BISA tetap `statusKembali: belum` untuk kasus auto-lock, karena piket sendiri belum benar-benar memutuskan resolusinya, cuma sistem yang proteksi sementara; ATAU putuskan auto-lock JUGA mengubah `statusKembali` — **perlu keputusan implementasi**: apakah auto-lock mengubah status permit atau cuma mengunci siswa tanpa menyentuh field permit. Rekomendasi: JANGAN ubah `statusKembali` saat auto-lock — biarkan tetap `belum` supaya piket besok masih bisa pilih salah satu dari 3 resolusi manual di atas terhadap permit yang sama, lock cuma efek SAMPING sementara sampai resolusi sebenarnya terjadi).
- Pertimbangkan kolom penanda sumber lock yang lebih terstruktur dari sekadar `lockedReason` text — misal enum baru `LockSource { manual, auto_2x_terlambat, auto_tidak_kembali_izin }` sebagai kolom baru di `Student`, ATAU pola string prefix konsisten yang mudah di-parse balik (`"[AUTO_TIDAK_KEMBALI]"` di awal `lockedReason`). **Diskusikan/putuskan pola ini saat implementasi** — jangan asal pilih tanpa pertimbangan bagaimana `confirmKembali`/`setPulang` nanti perlu MENDETEKSI apakah lock siswa ini berasal dari permit yang sedang diresolve.

### Backend
- `apps/api/src/permits/dto/` — DTO baru untuk aksi "Tidak Kembali" manual (`tandaiTidakKembali` atau serupa, BEDA dari `tandaiIzinTidakKembali` yang sudah ada — cek dulu apakah nama itu bentrok maknanya, method existing itu untuk kasus KLARIFIKASI "tidak tap pulang" yang beda konteks dari "izin keluar tidak kembali", JANGAN keliru menimpa method yang salah).
- `apps/api/src/permits/permits.service.ts` — method baru untuk resolusi manual "Tidak Kembali" (lock langsung), dan modifikasi `confirmKembali`/`setPulang` untuk cek+auto-unlock kalau relevan (poin 3 di atas).
- `apps/api/src/permits/permits.service.ts` — **HAPUS** `findPerluDitinjau()`, `countPerluDitinjau()` (dan endpoint terkait di controller) — pastikan tidak ada pemanggil lain sebelum hapus (grep dulu).
- **Job baru**: `apps/api/src/attendance/` — service+scheduler baru (pola sama `EndOfDayService`/`EndOfDayScheduler`, BullMQ, cron waktu berbeda) untuk auto-lock. Nama disarankan: `auto-lock-izin-tidak-kembali.service.ts` + `.scheduler.ts`, ATAU tambahkan sebagai job KEDUA di `EndOfDayScheduler` yang sudah ada dengan jadwal cron berbeda dalam queue yang sama (`upsertJobScheduler` bisa daftar >1 job) — **putuskan mana yang lebih rapi saat implementasi**, keduanya valid secara teknis.
- `apps/api/src/attendance/attendance.service.ts` — TIDAK perlu diubah (`tap()` sudah cek `lockedAt` generik, otomatis ikut menolak siswa yang di-lock oleh mekanisme baru ini tanpa perubahan kode tap itu sendiri).

### Frontend
- `apps/web/src/app/(piket)/piket/piket-board-view.tsx` — `BelumKembaliSection`: tambah tombol ke-3 "Tidak Kembali" (styling danger/merah, beda dari "Sudah Kembali"/hijau dan "Dianggap Pulang"/existing), dengan dialog konfirmasi (mirip `LockForm` yang sudah ada, minta alasan singkat opsional atau langsung submit).
- **HAPUS** section "Perlu Ditinjau" total: `PerluDitinjauSection` komponen, `SummaryCard` untuk ini, state `perluDitinjau`/`initialPerluDitinjau`, prop terkait di `page.tsx`.
- `apps/web/src/lib/core-types.ts` — cek apakah ada tipe yang perlu diubah (`StatusKembali` union type kalau ditambah value baru).

## Update Dokumentasi (WAJIB, bagian dari task ini — bukan opsional)
- 11-Decisions.md (11-Decisions.md) — tambah **ADR-026** (amandemen ADR-017 kedua setelah ADR-025): judul kira-kira "Auto-Lock Siswa Izin Keluar Tidak Kembali — Amandemen ADR-017 (Kedua)". Isi: konteks (celah yang ditemukan — siswa bisa tap normal besok pagi walau tidak pernah kembali dari izin kemarin karena lock murni manual), keputusan (job terjadwal lewat tengah malam auto-lock permit `keluar` yang masih `belum` dari hari sebelumnya, dengan auto-unlock saat piket akhirnya resolve permit itu), alasan (celah keamanan/pengawasan lebih berbahaya daripada risiko salah-kunci yang jadi alasan ADR-017 asli — DAN auto-unlock-saat-resolve mengurangi risiko itu dibanding kalau tidak ada auto-unlock sama sekali), konsekuensi (section Perlu Ditinjau dihapus, dst).
- 06-Features/dashboard-piket.md (06-Features/dashboard-piket.md) — update bagian yang menjelaskan "Perlu Ditinjau"/"Belum Kembali" sesuai model baru.

## Edge Cases
- Siswa auto-lock oleh job T098, TAPI SEBELUM piket sempat resolve, siswa itu juga kena 2x-terlambat (kasus lain, ADR-025) — `lockedAt` sudah terisi dari T098, `applyLateStrikeLock` existing SUDAH cek `student.lockedAt !== null` dulu sebelum lock (lihat kode existing) → tidak akan overwrite, aman secara alami tanpa perubahan tambahan.
- Piket resolve permit LAMA (`confirmKembali`) tapi siswa itu SUDAH di-unlock manual sebelumnya oleh piket lain (kasus 2 piket beda shift) — auto-unlock di poin 3 harus cek `lockedAt !== null` dulu sebelum mencoba unlock (jangan error kalau memang sudah unlock).
- Piket resolve permit LAMA yang statusnya `belum`, TAPI lock siswa itu sebenarnya berasal dari SUMBER LAIN (2x-terlambat, bukan dari T098) — auto-unlock TIDAK BOLEH ikut membuka lock yang bukan urusannya (inilah pentingnya penanda sumber lock yang jelas, poin di atas).
- Job auto-lock berjalan tapi Redis/BullMQ down semalam (skip 1 hari) — job berikutnya (job biasa `upsertJobScheduler` BullMQ akan tetap jalan sesuai jadwal berikutnya) TIDAK perlu backfill hari yang terlewat secara otomatis — cukup log warning, biarkan piket cek manual kalau ada gap (di luar scope perbaikan reliability infra, itu masalah operasional terpisah).

## Files
- **Buat:** service+scheduler baru untuk auto-lock (nama sesuai keputusan implementasi), migration Prisma kalau menambah kolom/enum baru.
- **Modifikasi:** `apps/api/src/permits/permits.service.ts`, `permits.controller.ts`, DTO baru; `apps/api/prisma/schema.prisma` (kalau ada perubahan enum/kolom); `apps/web/src/app/(piket)/piket/piket-board-view.tsx` (hapus Perlu Ditinjau, tambah tombol Tidak Kembali); `apps/web/src/app/(piket)/piket/page.tsx` (hapus prop terkait Perlu Ditinjau).
- **Jangan sentuh:** `AttendanceService.tap()` (sudah otomatis kompatibel via cek `lockedAt` generik), `applyLateStrikeLock` (mekanisme 2x-terlambat, di luar scope, cukup pastikan tidak saling menimpa).

## Acceptance Criteria
- [ ] Piket bisa klik "Tidak Kembali" manual di section Belum Kembali → siswa langsung terkunci saat itu juga.
- [ ] Job baru berjalan terjadwal, mengunci semua siswa dengan permit `keluar` kemarin yang masih `belum` — diverifikasi lewat trigger manual job (bukan menunggu jadwal asli) saat testing.
- [ ] Siswa yang SUDAH terkunci (dari sumber manapun) TIDAK di-lock ulang oleh job ini.
- [ ] Piket resolve (Sudah Kembali/Dianggap Pulang) permit yang jadi sumber auto-lock → siswa otomatis ter-unlock DALAM aksi yang sama, tidak perlu langkah kedua.
- [ ] Lock dari sumber LAIN (manual piket biasa, 2x-terlambat) TIDAK ikut ter-unlock oleh resolve permit yang tidak terkait dengannya.
- [ ] Section "Perlu Ditinjau" hilang total dari dashboard (komponen, state, endpoint backend).
- [ ] ADR-026 ditulis lengkap di 11-Decisions.md, dashboard-piket.md diperbarui.
- [ ] Build + type-check + test existing (terutama `attendance.service.spec.ts` soal lock) tetap hijau.

## Validasi Claudian
- [ ] Ini override ADR-017 KEDUA KALI (setelah ADR-025) — WAJIB ditulis sebagai ADR baru terdokumentasi, bukan penyimpangan diam-diam.
- [ ] Pastikan job baru TIDAK bentrok/duplikat dengan `EndOfDayScheduler` existing (jam 18:00, tujuan beda — deteksi belum-tap-pulang, bukan lock).
- [ ] Penanda sumber lock (2x-terlambat vs T098 vs manual piket) harus bisa dibedakan program secara ANDAL, bukan asumsi text-matching rapuh — putuskan strukturnya secara eksplisit sebelum coding, bukan sambil jalan.
- [ ] Baca ulang `attendance.service.ts` (`applyLateStrikeLock`, `tap()`) dan `students.service.ts` (`lock`/`unlock`) SEBELUM implementasi — reuse pola yang ada, jangan bikin mekanisme lock paralel yang tidak konsisten.
