# Task-CORE-014 / WEB-015: Alur Izin Saat Jam Pelajaran (Guru Kelas → Antri Piket) + Notifikasi

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) — lihat `06-Features/desain-redesign-presensi-izin-keluar.md` §6. Dieksekusi oleh Claude Code.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium-high
**Alasan pemilihan:** Model data baru (`ClassPermitRequest`) + 2 UI terpisah (form guru kelas, tabel review piket) + integrasi cetak surat existing + mekanisme notifikasi real-time. Task terbesar dari batch redesign ini.

## 2. Konteks & Tujuan Utama

Baca `06-Features/desain-redesign-presensi-izin-keluar.md` §6 dan §12. Alur BARU: siswa minta izin ke guru kelas SAAT jam pelajaran aktif → guru kelas input (BUKAN langsung ke Permit, tapi ke `ClassPermitRequest` status `menunggu`) → masuk tabel "Permintaan Izin" di dashboard piket → piket **Izinkan** (buat `Permit` resmi, cetak surat SAMA alur existing) atau **Tolak** (notifikasi ke guru + presensi siswa kembali kosong).

**Depends on:** task-CORE-010 (skema `ClassPermitRequest` harus ada), task-WEB-013 (UI checklist dengan status Izin harus ada).

## 3. Langkah Eksekusi Detail

### Backend

1. Buat service baru `ClassPermitRequestService` (`apps/api/src/class-permit-requests/`, atau di dalam `journal` module — cek konvensi struktur folder proyek, ikuti pola module NestJS existing) dengan method:
   - `create(sessionId, studentId, requestedById, alasan, jamKembaliDiharapkan)` — `jamKeluar` di-set `now()` otomatis di server (BUKAN dari FE, supaya konsisten "waktu form dibuka" — TAPI FE bisa kirim override kalau guru edit manual, lihat langkah 6 desain: "guru tetap bisa mengeditnya" — jadi terima `jamKeluar` dari FE, default value-nya yang di-generate FE saat form dibuka).
   - `listMenunggu()` — untuk dashboard piket, scope kampus petugas (REPLIKASI pola scoping kampus di `IzinKeluarView`/`PiketBoardRow` existing).
   - `izinkan(id, decidedById)` — buat `Permit` baru (`jenis: "keluar"`, `alasanKategori` dari `ClassPermitRequest.alasan` — MAPPING perlu ditentukan, cek `PermitAlasanKategori` existing yang cuma `sakit`/`izin_pribadi`/`tugas_dinas` DAN `PermitJenis` yang cuma `tidak_masuk`/`keluar` — VERIFIKASI dulu field mana yang cocok, jangan asumsi), link `permitId` balik ke `ClassPermitRequest`, update status jadi `diizinkan`.
   - `tolak(id, decidedById, alasanTolak)` — update status jadi `ditolak`, TIDAK buat `Permit`.

2. **Endpoint guru kelas**: `POST /teaching-sessions/:sessionId/permit-request` (role guru) — body `{ studentId, alasan, jamKeluar, jamKembaliDiharapkan }`, validasi `studentId` adalah siswa di kelas sesi ini (REPLIKASI pola validasi existing di `updateAttendance`).

3. **Endpoint piket**: `GET /class-permit-requests?status=menunggu` (role piket/admin — cek role yang tepat, REPLIKASI scoping kampus), `POST /class-permit-requests/:id/izinkan`, `POST /class-permit-requests/:id/tolak`.

4. **Saat `izinkan()` sukses** — TRIGGER auto-sync ke `ClassAttendanceMark` (task-CORE-015 akan implementasi detailnya, tapi method `izinkan()` di sini WAJIB memanggil service auto-sync itu — koordinasikan interface antara 2 task ini, atau gabung jadi 1 kalau lebih masuk akal saat implementasi).

5. **Saat `tolak()` sukses** — TRIGGER 2 hal: (a) reset `ClassAttendanceMark` siswa itu untuk sesi terkait jadi KOSONG lagi (hapus baris kalau ada, konsisten filosofi "default hadir" — TAPI dalam kasus ini justru harus JELAS statusnya "belum di-checklist ulang", bukan otomatis jadi hadir — VERIFIKASI ulang bagaimana FE menampilkan state ini, kemungkinan perlu field/flag terpisah "perlu di-checklist ulang" daripada sekadar hapus baris), (b) kirim notifikasi ke guru kelas (langkah 8).

### Frontend — Guru Kelas

6. Tambah tombol "Ajukan Izin" di UI checklist presensi (`attendance-table.tsx`, dari task-WEB-013) — saat siswa dipilih status "Izin" DAN ini SAAT JAM PELAJARAN AKTIF (sesi masih berlangsung), buka form (Dialog) isi alasan + jam kembali diharapkan — `jamKeluar` auto-terisi waktu form dibuka (`new Date()`), TAMPILKAN sebagai field editable (input time, sama pola `jam-keluar`/`jam-kembali` di `IzinKeluarForm` existing sebagai referensi).

7. Setelah submit — status siswa itu di checklist TAMPIL sebagai **"Menunggu Persetujuan Piket"** (state visual berbeda dari Hadir/Alfa/Izin biasa) — BUKAN langsung dianggap Izin final (karena masih `status: menunggu` di backend).

### Frontend — Piket

8. Buat halaman/section baru "Permintaan Izin" di dashboard piket (lokasi: cek struktur `(piket)` existing, kemungkinan tambah tab di `/piket/izin-keluar` atau halaman terpisah — TENTUKAN saat implementasi mengikuti konvensi navigasi piket yang sudah ada) — tabel list `ClassPermitRequest` status `menunggu`, kolom: Nama Siswa, Kelas, Guru Pengaju, Alasan, Jam Keluar, Jam Kembali Diharapkan, tombol **Izinkan** / **Tolak** per baris.
   - Klik **Izinkan** → panggil endpoint, otomatis buka print surat izin SAMA POLA `buildPrintUrl()`+`window.open()` di `IzinKeluarForm` existing (REUSE fungsi itu, jangan duplikasi).
   - Klik **Tolak** → Dialog kecil minta alasan tolak (opsional atau wajib, tentukan saat implementasi), submit.

### Notifikasi ke Guru

9. **Mekanisme notifikasi** — REUSE STRUKTUR `WaliKelasNotificationsProvider` (Socket.IO, room-based) sebagai referensi POLA, TAPI buat provider/context BARU scope ke **guru pengaju** (bukan wali kelas per-kampus) — room baru mis. `permit-request:teacher:${teacherId}`, emit event saat piket tolak. Badge/toast di `TopBar` guru (REUSE komponen bell yang sudah ada sebagai referensi visual, tapi data source beda).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **File baru:** `apps/api/src/class-permit-requests/*` (service, controller, dto)
- **Modifikasi:** `apps/web/src/app/(guru)/guru/sesi/[sessionId]/components/attendance-table.tsx` — tombol Ajukan Izin
- **File baru:** halaman/komponen "Permintaan Izin" piket
- **File baru:** provider notifikasi guru (REUSE pola `WaliKelasNotificationsProvider`)

**Dilarang dilakukan:**
- Jangan reuse `Permit` langsung untuk status "menunggu" — SUDAH DIPUTUSKAN model terpisah (`ClassPermitRequest`), `Permit` tetap murni "sudah final" seperti desainnya semula.
- Jangan duplikasi logic cetak surat — REUSE `buildPrintUrl()` pattern existing.

**Skenario kegagalan yang WAJIB ditangani:**
- Guru ajukan izin untuk siswa yang SUDAH punya `ClassPermitRequest` menunggu untuk sesi yang sama → tolak duplikasi (400) atau update yang existing, tentukan mana yang lebih masuk akal saat implementasi (REKOMENDASI: tolak duplikasi, minta guru batalkan dulu kalau mau ubah).
- Piket tolak izin, TAPI sesi sudah keburu closed (guru tidak sempat re-checklist) → status presensi siswa itu perlu penanganan KHUSUS (masuk kategori "belum terabsen" retroaktif) — dokumentasikan behavior yang dipilih, koordinasikan dengan task-CORE-013 (edit presensi) sebagai jalur guru memperbaiki setelah closed.
- Guru pengaju BUKAN guru yang sedang login sesi itu (race condition role) → validasi backend `requestedById` HARUS dari JWT sesi yang sedang aktif, bukan dari body request (SAMA prinsip keamanan seperti `teacherId` di seluruh proyek).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Guru kelas bisa ajukan izin siswa dari UI checklist presensi, status "Menunggu Persetujuan Piket"
- [ ] Piket punya halaman review dengan tombol Izinkan/Tolak
- [ ] Izinkan → buat `Permit` resmi, cetak surat (REUSE alur existing), auto-sync ke presensi (koordinasi task-CORE-015)
- [ ] Tolak → notifikasi ke guru kelas + presensi siswa kembali kosong/perlu-checklist-ulang
- [ ] Validasi keamanan: guru cuma bisa ajukan untuk sesi miliknya sendiri

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (KECUALI beberapa titik yang eksplisit ditandai "tentukan saat implementasi" — itu keputusan detail teknis yang wajar diputuskan Claude Code selama konsisten prinsip desain)
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar — **PERTIMBANGKAN PECAH jadi 2 task terpisah saat eksekusi** kalau BE+FE+notifikasi ternyata > 400 baris gabungan (mis. task-A backend+form guru, task-B dashboard piket+notifikasi)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada
- [ ] Dependency: task-CORE-010, task-WEB-013 WAJIB selesai dulu
