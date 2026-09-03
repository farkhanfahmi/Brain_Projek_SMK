---
tags: [absensi, design, presensi, jurnal-guru, izin-keluar, redesign]
status: draft-menunggu-approval
updated: 2026-09-02
---

# Desain — Redesign Presensi Guru + Izin Keluar (Reverse-Engineer JurnalePro + Adaptasi)

← Index (00-INDEX AbsenSI.md)

> **Status: DRAFT, MENUNGGU PERSETUJUAN USER.** Hasil diskusi panjang 2026-09-02, dipicu permintaan reverse-engineer fitur Presensi JurnalePro. User punya logika bisnis sendiri yang JAUH lebih matang dari sekadar tiru JurnalePro literal — dokumen ini merangkum SEMUA keputusan sejauh ini + celah yang masih perlu ditutup. **BELUM ditulis jadi task-task individual** — itu langkah SETELAH user approve dokumen ini secara keseluruhan.

---

## 1. Konteks & Kenapa Bukan Sekadar Tiru JurnalePro

JurnalePro: guru tap manual H/I/S/A untuk SEMUA siswa dari nol tiap sesi (kosong total di awal).

AbsenSI: **sudah lebih efisien** — presensi default dari tap gerbang, guru cuma koreksi. Redesign ini **mempertahankan keunggulan itu** sambil mengadopsi hal-hal JurnalePro yang benar-benar bernilai (quick-action, semua-hadir, validasi kelengkapan, rekap) — DIPADUKAN dengan logika bisnis baru milik user sendiri yang mengikat presensi kelas ↔ tap gerbang ↔ perizinan piket jadi 1 sistem konsisten.

## 2. Prinsip Inti (Disepakati)

1. **Tap gerbang = sumber data utama.** Siswa yang TIDAK tap gerbang tetap TAMPIL di daftar presensi guru (bukan disembunyikan), dengan status default **Alfa**.
2. **Guru WAJIB checklist manual** (Hadir/Izin/Alfa per siswa, atau tombol massal "Hadir Semua") — **sengaja tidak otomatis "semua hadir kalau tidak diapa-apakan"**, supaya guru betul-betul memanggil nama siswa satu per satu (kontrol pedagogis, bukan sekadar kepatuhan absensi).
3. **Validasi silang saat Simpan**: dibandingkan data tap gerbang vs checklist guru.
   - Siswa tap gerbang TAPI guru belum checklist apa pun → **blokir simpan**, tampilkan pesan `"<nama siswa> belum terabsen"`.
   - Siswa tap gerbang TAPI guru tandai **Alfa** → tercatat ke riwayat catatan siswa (`"Tidak ada di mata pelajaran <mapel>"`) DAN memicu auto-lock kartu (lihat §4).
4. **Izin dari guru piket = otomatis override status presensi** guru kelas — guru kelas tidak perlu tandai manual kalau piket sudah proses izin resmi.

## 3. Perubahan Interaksi: dari "Instant Toggle" ke "Checklist + Simpan"

**Ini perubahan arsitektur, bukan sekadar tambahan.** Kondisi SEKARANG (`attendance-table.tsx`): tiap klik toggle langsung `PATCH` ke server, tidak ada tombol "Simpan" terpisah.

**Kondisi BARU (disepakati)**: guru checklist semua siswa dulu (state lokal, belum tersimpan) → klik "Simpan" di akhir → validasi kelengkapan (poin 2.3) dijalankan → baru PATCH ke server sekaligus (bulk, bukan per-klik).

- Popup wajib isi keterangan MUNCUL saat guru pilih **Alfa** (poin 2.1) — state lokal, disimpan bareng saat "Simpan" ditekan (tidak langsung PATCH per-klik seperti sekarang).
- Tombol massal "Hadir Semua" tersedia (adopsi dari JurnalePro), tetap bisa dikoreksi individual sebelum "Simpan".

## 4. Auto-Lock Kartu Siswa — Timing Final: Saat Sesi CLOSED

**Disepakati:** lock **TIDAK instant** saat guru klik Alfa — dieksekusi saat **sesi ditutup** (`TeachingSession.closedAt` terisi, baik manual atau via cron `autoCloseDueSessions`). Alasannya: sepanjang sesi masih berlangsung, guru masih bisa koreksi checklist bebas — window ini otomatis jadi kesempatan mencegah salah-klik jadi permanen, TANPA butuh mekanisme terpisah.

**Job baru dibutuhkan**: perluas `autoCloseDueSessions()` (atau job terpisah dipanggil di titik yang sama) — setelah sesi di-set closed, cek semua `ClassAttendanceMark.status = tidak_ada_di_kelas` (Alfa) untuk siswa yang **tap gerbang hari itu** (kontradiksi: tap masuk tapi ditandai tidak ada di kelas) → jalankan:
- Kunci kartu siswa (`Student.lockedAt/lockedReason`, REPLIKASI pola ADR-017/T237) dengan `lockedReason` **PERSIS** teks keterangan alfa yang guru isi di popup (poin 2.1) — supaya piket tahu alasan tanpa buka riwayat terpisah.
- Catat ke riwayat/catatan siswa: `"Tidak ada di mata pelajaran <mapel>"` + keterangan guru.

## 5. Fitur Edit Presensi Setelah Sesi Closed (BARU — Belum Ada)

**Temuan investigasi kode:** TIDAK ADA fitur edit presensi historis yang proper. Backend `updateAttendance()` sebenarnya tidak block berdasarkan waktu/status (cuma cek `startedAt !== null`), tapi frontend `/guru/presensi` (kalender riwayat) HARDCODE `readOnly` dengan komentar developer "sementara, perbaiki nanti" — momen itu adalah SEKARANG.

**Desain baru:**
- Titik akses: `/guru/presensi` → kalender → detail tanggal → tombol "Edit" (baru) membuka mode edit, REUSE `AttendanceTable` yang sama TANPA `readOnly`.
- **Syarat**: sesi harus sudah `closed`. Rentang waktu dibatasi (usulan: 1 minggu terakhir, KONSISTEN pola JurnalePro "presensi hanya untuk 1 minggu terakhir" — user perlu konfirmasi rentang final).
- **WAJIB tercatat ke `activity_log`** setiap perubahan (before/after per siswa) — presensi adalah data akademik/kedisiplinan, beda dari draft biasa.
- **Cascade WAJIB**: kalau guru ubah status dari Alfa (yang sudah memicu lock+catatan §4) jadi Hadir/Izin — lock kartu siswa itu HARUS otomatis dicabut (atau minimal di-flag untuk ditinjau ulang piket) DAN catatan riwayat dikoreksi/ditandai "dibatalkan", supaya tidak ada kartu terkunci karena alasan yang sudah tidak valid. **Ini bagian paling rawan bug kalau tidak dirancang eksplisit** — perlu keputusan detail: auto-unlock langsung, atau cuma di-flag untuk piket cek manual?

## 6. Perizinan Saat Jam Pelajaran (Guru Kelas → Piket, Bukan Piket Langsung)

**Alur baru** (berbeda dari "Izin Keluar Sementara — Sub-alur A" existing yang di-input LANGSUNG oleh piket):

1. Siswa minta izin ke guru kelas (di tengah pelajaran).
2. Guru kelas klik **"Izin"** di checklist presensi → isi alasan + **jam kembali diharapkan** (jam KELUAR otomatis terisi = waktu form dibuka, **editable** oleh guru — disepakati poin 6).
3. Ini membuat entry baru masuk ke tabel **"Permintaan Izin"** di dashboard piket (BARU — beda dari form "Izin Keluar Sementara" existing yang piket isi sendiri) — piket TINGGAL REVIEW, bukan input dari nol.
4. Piket punya 2 aksi: **Izinkan** (cetak surat, sama alur existing) atau **Tolak**.
5. **Kalau ditolak**: (a) notifikasi muncul di dashboard guru kelas (BARU — perlu mekanisme notifikasi, REUSE pola `WaliKelasNotificationsProvider`/Socket.IO room yang sudah ada sebagai referensi struktur, TAPI scope ke guru pembuat izin, bukan wali kelas), (b) status presensi siswa itu **kembali kosong** (guru harus checklist ulang).
6. **Kalau diizinkan**: status presensi siswa otomatis "Izin" (tidak perlu guru checklist manual lagi, KONSISTEN prinsip inti §2.4).

**Model data baru dibutuhkan**: `Permit` existing TIDAK punya field "diajukan oleh guru kelas dari sesi presensi" maupun status "pending/ditolak" (`TeacherPermitStatus`/`StudentPermit` yang ada sekarang cuma `diizinkan` — tidak ada state "menunggu" atau "ditolak" untuk izin SISWA). Perlu skema baru atau perluasan `Permit` (`StatusKembali` beda konteks, jangan reuse serampangan) — **BUTUH DESAIN SKEMA TERPISAH**, dibahas saat breakdown ke task teknis.

## 7. Perizinan di Luar Jam Pelajaran (Istirahat dkk) — Revisi Logika User

**Logika AWAL user** (saat istirahat, siswa minta izin ke piket dulu) sempat punya celah: kalau jam kembali melebihi 1 mapel setelah istirahat, harus konfirmasi guru kelas dulu — TAPI sebelum mapel itu MULAI, guru belum bisa input apa pun, jadi surat tidak bisa dicetak. Race condition.

**Logika REVISI (final, disepakati user):**
- Izin di LUAR jam pelajaran aktif (istirahat dkk) → **keputusan sepenuhnya di piket**, tidak menunggu guru kelas.
- Begitu piket mengizinkan (membuat Permit), **guru kelas WAJIB menerima** status itu apa adanya — TIDAK ADA lagi mekanisme "guru kelas harus konfirmasi dulu" untuk kasus ini.
- Yang tampil di halaman presensi guru: **status Izin dari piket + jam kembali yang piket catat** — otomatis, guru tidak perlu approve/tolak apa pun untuk kasus izin-luar-jam-pelajaran ini.
- Ini MENGHAPUS state-machine berbasis waktu rumit yang sempat dikhawatirkan Hermes (poin 7 audit sebelumnya) — logika jadi jauh lebih sederhana: **sumber keputusan ditentukan oleh KAPAN izin diajukan** (saat jam pelajaran aktif → guru kelas yang mulai alurnya ke piket §6; di luar jam pelajaran → piket langsung, guru kelas cuma menerima hasil).

## 8. Konfirmasi Izin Pulang (Sub-alur B) — Diaktifkan Lagi, Diperbaiki

Task **T264** (2026-08-31) meng-hide fitur ini karena menampilkan SEMUA siswa tap pulang hari itu (885 baris, tidak praktis) — investigasi kala itu: tidak ada bug data, murni terlalu tidak selektif secara desain.

**Keputusan baru (disepakati user) — AKTIFKAN LAGI dengan syarat:**
- Filter HANYA menampilkan siswa yang **statusnya "Izin" dari inputan GURU KELAS** (hasil alur §6/§7 baru) — BUKAN semua siswa tap pulang seperti sebelumnya. Ini menjawab akar masalah T264 (over-inklusif) sekaligus jadi titik integrasi natural dengan redesign ini.
- **Auto-cleanup**: entry di antrian ini otomatis hilang ketika:
  - Ganti hari (antrian di-reset harian — scope query "hari ini" seperti pola existing lain di proyek).
  - Guru MENGUBAH status siswa dari Izin kembali ke Hadir (mis. ternyata siswa batal izin) — entry di dashboard piket ikut hilang otomatis (BUKAN nyangkut, konsisten real-time sinkron dengan status presensi guru).

## 9. Ringkasan Perubahan Skema Data (Awal, Belum Final)

| Area | Perubahan |
|---|---|
| `ClassAttendanceMark` | Kemungkinan perlu field baru: `keterangan` (teks alasan Alfa, wajib diisi via popup §2.1) — SAAT INI tidak ada field alasan sama sekali. |
| Izin siswa dari guru kelas (§6) | **Model/skema baru** — belum ada representasi "permintaan izin dari sesi presensi, status pending/ditolak/diizinkan" di skema saat ini. Perlu didesain terpisah dari `Permit` existing (yang cuma untuk alur piket-langsung). |
| Auto-lock (§4) | REUSE `Student.lockedAt/lockedReason/lockedById` (ADR-017) yang sudah ada — TIDAK perlu skema baru, cuma trigger baru (dipanggil dari job close sesi, bukan endpoint manual). |
| Cascade unlock saat edit (§5) | Perlu logic baru menghubungkan `ClassAttendanceMark` ↔ `Student.lockedAt` — SAAT INI tidak ada relasi eksplisit antara 2 hal ini di kode. |
| Notifikasi guru saat ditolak piket (§6) | Perlu mekanisme baru — REUSE pola Socket.IO room (`WaliKelasNotificationsProvider` sebagai referensi struktur), scope ke guru individual pembuat izin. |

## 10. Keputusan Final (disepakati 2026-09-02)

1. **Rentang waktu edit presensi**: **1 minggu** dari tanggal sesi (KONSISTEN pola JurnalePro).
2. **Cascade unlock**: **auto-unlock LANGSUNG** saat guru ubah Alfa→Hadir (bukan flag manual piket) — begitu status dikoreksi, `Student.lockedAt` dicabut otomatis + `unlockedById` = system actor (pola sama `lockGuruTanpaTapPulang`).
3. **Skema data izin baru**: lihat §12 di bawah.
4. **Job auto-lock**: **diperluas dari `autoCloseDueSessions()` existing** — dipanggil di titik yang sama saat sesi di-set closed, bukan job cron terpisah.

## 11. Asumsi Tambahan (diputuskan Hermes, non-blocking — koreksi kalau perlu)

- **Notifikasi "ditolak piket" ke guru**: badge/toast di dashboard guru, REUSE pola Socket.IO `WaliKelasNotificationsProvider` (join room per-guru, bukan per-kampus) — bukan WhatsApp/push eksternal.
- **Rentang 1 minggu edit presensi**: dihitung dari **tanggal sesi**, bukan tanggal klik edit.

## 12. Skema Data Baru (Final)

```prisma
enum ClassAttendanceStatus {
  hadir
  tidak_ada_di_kelas
  izin              // BARU
}

model ClassAttendanceMark {
  // ...field existing (id, sessionId, studentId, status, markedById, markedAt, dst)...
  keterangan String? @db.Text @map("keterangan") // BARU — alasan Alfa (wajib via popup) ATAU catatan izin
  permitId   Int?    @map("permit_id")            // BARU — terisi kalau status=izin dari Permit resmi (auto-sync §7)
  permit     Permit? @relation(fields: [permitId], references: [id])
}

enum ClassPermitRequestStatus {
  menunggu
  diizinkan
  ditolak
}

// Izin SAAT JAM PELAJARAN AKTIF — diajukan guru kelas, diputuskan piket.
// TERPISAH dari Permit (yang tetap dipakai utuh untuk alur piket-langsung §7,
// TIDAK ada state "menunggu" di Permit existing).
model ClassPermitRequest {
  id                   Int                       @id @default(autoincrement())
  sessionId            Int                       @map("session_id")
  studentId            Int                       @map("student_id")
  requestedById        Int                       @map("requested_by") // User.id guru kelas
  alasan               String?                   @db.Text
  jamKeluar            DateTime                  @map("jam_keluar") // auto-isi saat form dibuka, editable guru
  jamKembaliDiharapkan DateTime?                 @map("jam_kembali_diharapkan")
  status               ClassPermitRequestStatus  @default(menunggu)
  decidedById          Int?                      @map("decided_by") // User.id piket
  decidedAt            DateTime?                 @map("decided_at")
  alasanTolak          String?                   @map("alasan_tolak") @db.Text
  permitId             Int?                      @map("permit_id") // terisi kalau diizinkan — REUSE Permit existing (cetak surat, kode verifikasi)
  createdAt            DateTime                  @default(now()) @map("created_at")

  session     TeachingSession @relation(fields: [sessionId], references: [id])
  student     Student         @relation(fields: [studentId], references: [id])
  requestedBy User            @relation("ClassPermitRequestedBy", fields: [requestedById], references: [id])
  decidedBy   User?           @relation("ClassPermitDecidedBy", fields: [decidedById], references: [id])
  permit      Permit?         @relation(fields: [permitId], references: [id])

  @@index([status])
  @@index([studentId, sessionId])
  @@map("class_permit_requests")
}
```

**Alur Permit unified**: begitu `Permit` row dibuat — LEWAT JALUR MANAPUN (disetujui `ClassPermitRequest` di §6, ATAU diinput piket langsung untuk kasus luar-jam-istirahat §7) — sistem cari `TeachingSession` aktif siswa itu pada rentang waktu izin, auto-set `ClassAttendanceMark.status = izin` + `permitId` terisi. Guru kelas TIDAK checklist manual untuk kasus ini (konsisten §2.4).

## 13. Breakdown Task (8 task, urutan dependency)

1. **task-CORE-010** — Migrasi skema DB (fondasi, semua task lain depends on ini).
2. **task-CORE-011** — Backend: endpoint bulk checklist+simpan dengan validasi "belum terabsen".
3. **task-WEB-013** — Frontend: UI checklist (ganti instant-toggle) + popup alasan Alfa + tombol massal.
4. **task-CORE-012** — Backend: perluas `autoCloseDueSessions()` untuk auto-lock kartu.
5. **task-CORE-013 / WEB-014** — Edit presensi setelah closed (window 1 minggu) + cascade auto-unlock + activity log.
6. **task-CORE-014 / WEB-015** — Alur `ClassPermitRequest`: form guru kelas + tabel piket + approve/tolak + notifikasi.
7. **task-CORE-015** — Auto-sync `Permit` → `ClassAttendanceMark` (status izin otomatis).
8. **task-WEB-016** — Re-enable + scope ulang Sub-alur B (Konfirmasi Izin Pulang) dengan filter+auto-cleanup.

Detail lengkap tiap task ada di `06-Features/tasks/task-CORE-010-*.md` dst.

## Status
**DISETUJUI 2026-09-02** — siap dipecah jadi task individual (lihat §13).

## 🔗 Lihat Juga
- `06-Features/tasks/T264-hide-konfirmasi-izin-pulang-perbaikan-riwayat-izin.md` — konteks kenapa Sub-alur B di-hide sebelumnya
- `06-Features/dashboard-guru-jurnal.md` — spec Jurnal Guru
- `STATUS.md` — daftar task aktif
