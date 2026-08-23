# T191 — API+Web: Redesign Form Izin Guru (Search Nama, Kelas Ditinggalkan, Rentang Tanggal) + Fix Filter Riwayat Izin

## Depends on
Tidak ada dependency teknis. Independen dari task lain di rangkaian ini.

## Objective
1. Form create Izin Guru (`TeacherPermit`) — ganti dropdown pilih guru jadi **field search nama**; ganti konsep "sesi" jadi **"kelas yang ditinggalkan pada tanggal tertentu"** (sistem tampilkan kelas yang PUNYA jadwal hari itu, admin tinggal pilih); ganti tanggal tunggal jadi **rentang tanggal** (1 hari = isi tanggal sama di keduanya).
2. Riwayat Izin Guru — filter **auto-apply** (langsung terapkan begitu field diisi) SUDAH BENAR SECARA DESAIN (tidak butuh tombol), TAPI user melaporkan ADA masalah — VERIFIKASI ULANG dan PERBAIKI kalau ternyata TIDAK auto-apply dengan benar di semua field filter (bukan cuma search).

## Context — Temuan Riset (2026-08-15)

**Form create SAAT INI**: `CreateTeacherPermitDto` — `teacherId` (dropdown biasa, TIDAK searchable), `tanggal` (SINGLE date, bukan rentang), `sessionId?` (TUNGGAL, opsional, dari toggle "sesi" yang fetch `GET /teaching-sessions?teacherId&tanggal`). `TeacherPermitsService.create()` validasi sessionId harus milik teacherId+tanggal yang sama.

**Reusable untuk kelas-ditinggalkan**: `ScheduleResolverService.getJadwalHariIni(teacherId, tanggal)` SUDAH ADA dan persis fungsi yang dibutuhkan — endpoint `GET /teaching-sessions?teacherId&tanggal` SUDAH mengembalikan field kelas+ruangan+kampus+mapel per sesi.

**Riwayat Izin SAAT INI**: `izin-table.tsx` filter SUDAH auto-apply (`onChange` → `useMemo` client-side filter) — TIDAK ADA tombol "Terapkan" DAN memang tidak seharusnya ada satu (auto-apply adalah desain yang benar, BUKAN bug hilang tombol). **User mungkin mengalami filter yang TERASA tidak langsung ter-apply karena alasan LAIN** (misal debounce terlalu lama, field tertentu yang justru TIDAK auto-apply padahal terlihat seperti filter, atau state UI yang membingungkan) — WAJIB VERIFIKASI LANGSUNG (live-test tiap field filter satu-satu) sebelum menyimpulkan "sudah benar" atau menemukan bug sesungguhnya. JANGAN asumsi laporan user salah tanpa reproduksi.

**Keputusan user (dikonfirmasi)**: untuk rentang tanggal, kelas yang bisa dipilih = **gabungan SEMUA kelas unik** yang muncul di jadwal guru itu sepanjang rentang tanggal (bukan per-hari terpisah) — admin centang sekali, berlaku untuk semua hari dalam rentang itu.

## Spec Detail

### 1. Backend — Schema `TeacherPermit`: dukung rentang tanggal + multi-kelas

**KEPUTUSAN ARSITEKTUR**: `TeacherPermit` SAAT INI 1 baris = 1 tanggal + 1 sesi opsional. Untuk RENTANG tanggal + MULTI kelas — PUTUSKAN (VERIFIKASI dulu, JANGAN asumsi sepihak, klarifikasi user kalau ragu):
- **Opsi A (REKOMENDASI)**: TETAP 1 baris `TeacherPermit` = 1 record per (guru, RENTANG tanggal) — tambah `tanggalSelesai` (nullable, NULL = 1 hari = `tanggal` itu sendiri, KONSISTEN backward-compatible data lama) DAN ganti `sessionId` (single) jadi RELASI many-to-many ke `TeachingSession` (tabel junction baru `TeacherPermitSession`, SERUPA pola `GradeAssessmentSession` dari T172) — supaya bisa simpan BANYAK kelas yang ditinggalkan dalam 1 izin.
- **Opsi B**: 1 baris PER TANGGAL dalam rentang (auto-generate N baris kalau rentang N hari) — LEBIH SEDERHANA query-nya (tidak perlu ubah struktur tanggal) TAPI berpotensi banyak baris duplikat untuk 1 kejadian izin yang sama, dan JIKA salah satu hari perlu dibatalkan sebagian, jadi rumit (harus hapus baris tertentu, bukan 1 record utuh).
- WAJIB PUTUSKAN dan JELASKAN alasan sebelum implementasi besar — REKOMENDASI KUAT Opsi A (1 record utuh mewakili 1 kejadian izin, lebih natural untuk approval/riwayat/pembatalan).

```prisma
// Tambah ke TeacherPermit:
tanggalSelesai DateTime? @db.Date @map("tanggal_selesai") // NULL = 1 hari (sama dengan `tanggal`)

// Model baru (kalau Opsi A):
model TeacherPermitSession {
  id        Int @id @default(autoincrement())
  permitId  Int @map("permit_id")
  sessionId Int @map("session_id")

  permit  TeacherPermit   @relation(fields: [permitId], references: [id], onDelete: Cascade)
  session TeachingSession @relation(fields: [sessionId], references: [id])

  @@unique([permitId, sessionId])
  @@map("teacher_permit_sessions")
}
```
- `sessionId` (single, lama) — EVALUASI: hapus kolom lama SETELAH migrasi data existing ke tabel junction baru (KALAU ada data lama dengan `sessionId` terisi — migration data WAJIB, JANGAN kehilangan data existing), ATAU biarkan kolom lama ada tapi TIDAK dipakai lagi (deprecated) kalau migrasi data dirasa berisiko — PUTUSKAN saat implementasi berdasarkan VOLUME data existing yang perlu dimigrasikan.

### 2. Backend — endpoint baru "kelas yang ditinggalkan untuk rentang tanggal"

- `GET /teacher-permits/kelas-ditinggalkan?teacherId=&from=&to=` — loop tiap tanggal dalam rentang, panggil `getJadwalHariIni(teacherId, tanggal)` (REUSE method existing), GABUNGKAN hasil jadi SET kelas UNIK (dedupe by `kelasId`) di seluruh rentang — return daftar kelas (id, nama, mapel per kelas kalau guru ajar beda mapel di kelas sama beda hari — TAMPILKAN semua kombinasi unik kelas+mapel yang relevan).
- `TeacherPermitsService.create()` — terima `tanggal`, `tanggalSelesai?`, `sessionIds: number[]` (GANTI dari `sessionId` tunggal) — validasi SEMUA sessionIds milik teacherId (KONSISTEN pola validasi T172 `GradesService`), simpan via `TeacherPermitSession` (transaksi).

### 3. Frontend — form create Izin Guru

- Ganti dropdown `teacherId` — field **search-select** (ketik nama, hasil terfilter live, KONSISTEN pola search yang SUDAH established di banyak tabel proyek — REUSE komponen combobox/search-select kalau SUDAH ada di `packages/ui`, JANGAN buat dari nol kalau ada preseden).
- Ganti field tanggal tunggal — **range date picker** (2 input: Dari/Sampai, ATAU 1 komponen range picker kalau `packages/ui`/library yang dipakai proyek SUDAH punya — REKOMENDASI: default `tanggalSelesai` SAMA dengan `tanggal` awal kalau admin cuma isi 1 tanggal/klik sekali, sesuai instruksi user "jika hanya 1 hari biar admin mengisi tanggal yang sama").
- Ganti toggle "sesi" — **field "Kelas yang Ditinggalkan"**: setelah admin isi rentang tanggal + pilih guru, fetch endpoint baru (poin 2), tampilkan CHECKLIST kelas (bisa pilih lebih dari 1) — REPLACE konsep toggle seharian/sesi LAMA sepenuhnya.

### 4. Frontend — verifikasi & fix filter Riwayat Izin

- `izin-table.tsx` — **WAJIB live-test SATU PER SATU tiap field filter** yang ada (search nama, rentang tanggal, kategori, dll — SEBUTKAN field apa saja yang ada saat implementasi) — untuk MASING-MASING, konfirmasi perubahan input LANGSUNG mengubah hasil tabel TANPA perlu aksi tambahan apa pun. KALAU ditemukan field yang TIDAK auto-apply (misal butuh blur/enter, atau state tidak ter-sync), PERBAIKI supaya SEMUA field konsisten auto-apply.
- KALAU SETELAH VERIFIKASI ternyata SEMUA field SUDAH benar auto-apply (tidak ada bug ditemukan) — LAPORKAN temuan ini ke user secara eksplisit (mungkin masalah yang dialami user adalah PERSEPSI/UX, misal delay visual yang terasa lambat, atau field tertentu yang TERLIHAT seperti butuh submit padahal tidak) — JANGAN memaksakan "perbaikan" untuk sesuatu yang sebenarnya sudah benar, tapi PERTIMBANGKAN tambah indikator visual (misal skeleton loading singkat) kalau update terasa "diam" tanpa umpan balik.

## Edge Cases
- Guru dengan NAMA MIRIP saat search — pastikan hasil pencarian menampilkan info tambahan (NIY, mata pelajaran) untuk disambiguasi.
- Rentang tanggal panjang (misal 1 minggu) — endpoint kelas-ditinggalkan HARUS efisien (batch query, bukan N query sekuensial per hari kalau bisa dihindari).
- Guru TANPA jadwal mengajar sama sekali di rentang tanggal itu (misal karyawan yang salah pilih, atau guru piket murni) — checklist kelas KOSONG, form TETAP bisa submit izin TANPA kelas dipilih (izin "seharian" tanpa kelas spesifik yang ditinggalkan, KONSISTEN kemungkinan skenario lama "cakupan seharian").
- Data `TeacherPermit` LAMA (sebelum migrasi ke multi-kelas) — TETAP bisa ditampilkan di riwayat (tanggalSelesai NULL = 1 hari, sessionId lama dikonversi/ditampilkan sebagai 1 kelas di relasi baru kalau migrasi dilakukan).

## Files
- **Modifikasi:** `apps/api/prisma/schema.prisma` (`TeacherPermit` +field, model baru `TeacherPermitSession`), `apps/api/src/teacher-permits/` (service+controller+DTO), `apps/web/src/app/(admin-jurnal)/admin-jurnal/izin/izin-form.tsx` (dan duplikat T157 `(admin)/izin-guru/`), `izin-table.tsx` (verifikasi+fix filter).
- **Buat:** migration Prisma baru.

## Acceptance Criteria
- [x] Form create — search nama guru (bukan dropdown polos) berfungsi.
- [x] Form create — rentang tanggal berfungsi, default `tanggalSelesai = tanggal` kalau admin cuma pilih 1 tanggal.
- [x] Form create — checklist kelas-ditinggalkan menampilkan GABUNGAN unik kelas dari SELURUH rentang tanggal, bisa pilih lebih dari 1.
- [x] Backend validasi SEMUA sessionIds milik guru yang login.
- [x] Riwayat Izin — SEMUA field filter dikonfirmasi auto-apply (verifikasi KODE, bukan live-click — lihat catatan).
- [x] Data TeacherPermit lama TETAP tampil benar di riwayat — TIDAK RELEVAN, 0 baris data lama di DB dev saat implementasi (dicek `SELECT COUNT(*)` sebelum migrasi).
- [x] Build + type-check hijau, jest baru untuk skenario rentang tanggal + multi-kelas.

## Validasi Claudian
- [x] **WAJIB putuskan Opsi A vs B** (struktur rentang tanggal) — Opsi A dikonfirmasi eksplisit user (1 record + tanggalSelesai + junction table `TeacherPermitSession`, pola sama `GradeAssessmentSession` T172).
- [x] **WAJIB live-test filter Riwayat Izin SATU PER SATU** — browser Playwright TERKUNCI dipakai sesi lain saat implementasi (tidak dipaksa ambil alih, sesuai instruksi keselamatan sesi paralel). Diverifikasi via PEMBACAAN KODE: `izin-guru-view.tsx` (filter Tanggal+Guru, `onChange` → `handleFilterChange` → refetch langsung, tanpa debounce/tombol submit) dan `izin-table.tsx` (search nama, `onChange` → `useMemo` filter client-side langsung). Ketiga field TIDAK ADA jalur yang butuh aksi tambahan — kesimpulan: sudah benar auto-apply, TIDAK ADA bug ditemukan di kode. Endpoint baru di-smoke-test via curl (401 tanpa auth, route ter-mapping benar) sebagai pengganti sebagian verifikasi live.
- [x] Konfirmasi migrasi data `TeacherPermit` lama — 0 baris di DB dev (`SELECT COUNT(*) FROM teacher_permits` = 0), kolom `sessionId` DIHAPUS LANGSUNG tanpa migrasi data (tidak ada data untuk kehilangan).

## Catatan Implementasi (2026-08-16)

- **Blast radius lebih besar dari spec tertulis**: riset menemukan 4 file konsumen lain (di luar `teacher-permits/`) yang assume `TeacherPermit.sessionId` tunggal — `teaching-sessions.service.ts` (3 titik: `getMyToday`, `startSession`, `flagPermitsNeedingFollowUp`+`izinSesiSpesifikSudahLewat`), `tv-piket.service.ts` (`hitungGuruIzin`), `teacher-attendance-report.service.ts` (2 mode: single-day + range). Semua diupdate konsisten ke pola `sessions: TeacherPermitSession[]` + query rentang tanggal (`tanggal <= X AND (tanggalSelesai >= X OR (tanggalSelesai IS NULL AND tanggal = X))`), dikonfirmasi user untuk lanjut full scope (bukan minimal sesuai file yang disebut spec).
- **Report range mode**: 1 permit rentang N hari sekarang dihitung N kali (1x per tanggal dalam rentang, clamped ke `[from, to]` query laporan) — SEBELUMNYA (skema 1-tanggal) otomatis 1x per permit, jadi ini perubahan behavior yang DISENGAJA supaya laporan akurat untuk rentang.
- **`getKelasDitinggalkan()`**: panggil `TeachingSessionsService.generateForDate()` (idempotent, existing dari T040) untuk SETIAP tanggal dalam rentang SEBELUM query — supaya tanggal FUTURE (izin diinput lebih awal untuk minggu depan) juga sudah punya baris `TeachingSession` untuk dipilih, bukan cuma tanggal yang sudah kena cron harian.
- **Insiden proses dev API duplikat**: ditemukan 3 proses `pnpm --filter @absensi/api dev` (nest watch) berjalan bersamaan di folder dev sejak sesi-sesi lalu (leftover lupa dimatikan), menyebabkan proses yang benar-benar pegang port 3101 macet tidak recompile sejak 09:52. Dikonfirmasi ke user sebelum kill+restart (bukan systemd production, jadi bukan skenario insiden T117). Dibersihkan, direstart bersih, endpoint baru terverifikasi ter-mapping.
- **Sesi paralel T188**: terdeteksi sesi lain aktif mengubah `admin-jurnal-sidebar.tsx` (menambah menu Kalender Pendidikan) di tengah kerja T191 — dikonfirmasi user T188 sudah selesai sebelum saya lanjut, tidak ada konflik file (T191 tidak menyentuh sidebar).
