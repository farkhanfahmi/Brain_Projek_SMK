---
tags: [absensi, adr, decisions]
updated: 2026-07-21
---

# 11 — Decisions (ADR)

← Index (00-INDEX AbsenSI.md)

> Setiap keputusan arsitektur permanen dicatat di sini. Format: Konteks → Keputusan → Alasan → Konsekuensi. **Jangan ubah keputusan di sini tanpa diskusi ulang dan catat ADR baru yang men-supersede.**

---

## ADR-001: Tidak Import Data Spreadsheet Lama
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Sekolah saat ini pakai Google Form + spreadsheet untuk absensi. Ada histori data di sana.
**Keputusan:** AbsenSI mulai dari data baru (clean slate). Tidak ada migrasi data spreadsheet di fase 1.
**Alasan:** Mapping NISN/NIP ke UID kartu RFID belum ada di data lama (data lama tidak berbasis kartu). Effort import tidak proporsional dengan manfaat di fase awal.
**Konsekuensi:** Kalau nanti dibutuhkan historical data, harus dikerjakan sebagai task terpisah di fase development selanjutnya — bukan blocker fase 1.

---

## ADR-002: Stack TypeScript Full (NestJS + Next.js), Bukan Laravel
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Tim biasa pakai Laravel/MySQL di proyek lain (DasiPelajar, SIMA-Sarpras). AbsenSI direncanakan jadi proyek perintis ekosistem aplikasi sekolah yang lebih besar. Tim ingin belajar ekosistem JS, dibantu Claude Code untuk eksekusi.
**Keputusan:** Backend NestJS + Prisma + PostgreSQL, frontend Next.js, dalam satu monorepo Turborepo.
**Alasan:**
- Shared TypeScript types lintas layer (backend ↔ frontend ↔ client) — manfaat nyata untuk visi ekosistem banyak aplikasi terhubung
- NestJS dipilih (bukan Express polos) karena strukturnya paling dekat dengan konsep Laravel (module, DI, decorator) — transisi developer Laravel paling mulus
- PostgreSQL dipilih atas MySQL karena kebutuhan query rekap analitik fleksibel (filter multi-dimensi) di masa depan
**Konsekuensi:**
- **Risiko diakui secara sadar:** 2 dari 3 developer belum familiar paradigma backend Node/Nest. Mitigasi: pembagian modul disesuaikan kekuatan masing-masing (lihat team.md (_claudian/team.md)), modul Core/paling kritis dipegang developer paling fullstack.
- Tim akan memelihara 2 stack berbeda (Laravel di proyek lama, TS di AbsenSI dst). Ini diterima sebagai trade-off sadar demi belajar.
- Ini bukan keputusan "JS lebih baik untuk realtime/API" — itu klaim yang ditolak (realtime/API bisa dicapai stack apa pun). Alasan sebenarnya adalah shared-types dan kesiapan tim belajar ekosistem baru untuk proyek-proyek berikutnya.

---

## ADR-003: Modular Monolith, Bukan Microservices
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Visi jangka panjang adalah ekosistem banyak aplikasi sekolah dengan data terpusat. Ada godaan untuk langsung desain microservices.
**Keputusan:** AbsenSI dibangun sebagai **modular monolith** — satu aplikasi NestJS dengan batas modul tegas (Core, Attendance, Card, Schedule, Notification-stub), bukan service terpisah-terpisah.
**Alasan:** Skala 1 sekolah (2.500 siswa, 100+ guru) tidak butuh kompleksitas distributed system. Microservices di skala ini menambah overhead operasional (service discovery, network latency, deployment complexity) tanpa manfaat nyata. Modul Core dirancang punya batas jelas agar **bisa** dipecah jadi service terpisah nanti — saat aplikasi sekolah ke-2/ke-3 benar-benar butuh akses data siswa/guru terpusat.
**Konsekuensi:** Tim harus disiplin menjaga batas modul (Attendance tidak boleh query langsung tabel Core di luar service layer yang disediakan) agar pemecahan ke microservice nanti tidak butuh rewrite besar.

---

## ADR-004: Reader RFID = USB-HID Keyboard Emulation
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Ada dua tipe USB RFID reader: HID keyboard-emulation vs SDK/serial proprietary. Tipe ini menentukan seluruh desain client mini-PC.
**Keputusan:** Hardware yang dipakai dikonfirmasi tipe **HID keyboard emulation** (tap kartu = device "mengetik" UID + Enter).
**Alasan:** Sudah dicek langsung oleh tim di hardware fisik.
**Konsekuensi:** Client mini-PC **tidak perlu** library serial/driver native (`serialport`, `node-hid`, SDK vendor). Cukup halaman web kiosk (`apps/kiosk`) dengan input field auto-focus menangkap keystroke. Ini menyederhanakan scope `apps/kiosk` secara signifikan — jadi web app biasa, bukan software hardware-adjacent.

---

## ADR-005: Fase 1 Hanya Gerbang — Terlambat Dihitung dari Jam Masuk Sekolah, Bukan Per Sesi Mengajar
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Definisi final "guru terlambat" adalah tap kelas melebihi jadwal mengajar (fase 2). Tapi fase 1 hanya punya reader di gerbang, belum ada reader kelas.
**Keputusan:** Fase 1: status terlambat (siswa & guru) dihitung dari **jam tap gerbang vs jam masuk sekolah** (atau jam mengajar pertama guru hari itu, untuk kasus guru). Logika "terlambat per sesi mengajar" baru aktif di fase 2 saat reader kelas terpasang.
**Alasan:** Data yang tersedia di fase 1 cuma tap gerbang — tidak cukup untuk menyimpulkan kehadiran per sesi mengajar spesifik.
**Konsekuensi:** Modul **Schedule** harus sudah punya struktur jadwal mengajar guru dari fase 1 (untuk hitung "jam mengajar pertama"), meskipun reader kelas belum ada. Desain skema DB harus mendukung kedua fase tanpa rebuild — lihat 04-Database-Schema (04-Database-Schema.md).

---

## ADR-006: Event-Driven untuk Notifikasi Masa Depan
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Notifikasi WA ke orang tua direncanakan fase 3, tapi jalur arsitekturnya harus disiapkan dari fase 1 agar tidak perlu refactor core logic nanti.
**Keputusan:** Setiap tap kartu yang berhasil di-record memicu event (`attendance.recorded`) yang di-dispatch ke BullMQ queue. Modul Notification (fase 1: stub/cuma log) subscribe ke event ini.
**Alasan:** Decoupling — controller attendance tidak perlu tahu/peduli ada listener WA atau tidak. Menambah fitur notifikasi nanti = tambah listener baru, bukan ubah modul Attendance.
**Konsekuensi:** Butuh Redis + BullMQ terpasang dari fase 1 meskipun belum ada consumer nyata selain logging.

---

## ADR-008: Role Generik (`super_admin`, `card_admin`, `guru`), Bukan Diikat ke Identitas Developer
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Saat diskusi role, "Admin Pusat" awalnya didefinisikan sebagai "3 orang developer" — ini mengikat role sistem ke identitas spesifik orang, bukan ke fungsi.
**Keputusan:** Role disimpan generik di database: `super_admin` (full akses semua fitur), `card_admin` (CRUD kartu saja), `guru` (read-only riwayat sendiri). Saat ini 3 akun `super_admin` kebetulan dipegang oleh 3 developer, tapi ini bukan aturan permanen yang mengikat role ke orang tersebut.
**Alasan:** Kalau role diikat ke identitas, sistem rapuh terhadap perubahan personel (developer resign/pindah) dan tidak fleksibel kalau nanti staff non-developer (Kepala TU, dst.) butuh akses setingkat itu.
**Konsekuensi:** Skema tabel `users`/`accounts` butuh kolom `role` (enum atau FK ke tabel roles), permission dicek di level API (backend), **bukan** cuma disembunyikan di UI frontend — supaya Admin Pengelola Kartu tidak bisa akses endpoint di luar modul kartu meski tahu URL-nya.

---

## ADR-009: Import Data Master Naik ke Scope Fase 1, Desain Hybrid CSV + Tap-to-Assign
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** Rollout awal butuh daftarkan 2.500 siswa + 100 guru + kartu RFID-nya. Input manual satu-satu tidak praktis. UID kartu untuk rollout awal sudah diketahui (dari vendor), tapi ada siswa baru yang kartunya belum terekam.
**Keputusan:** Fitur import dinaikkan dari backlog fase 3 ke scope fase 1, dengan urutan wajib: (1) Kelas & Jurusan diinput manual dulu sebagai master data, (2) import CSV data siswa & guru, (3) pemetaan kartu RFID dengan 2 mode — Mode A bulk CSV (UID sudah diketahui) dan Mode B tap-to-assign (untuk kartu yang belum terekam).
**Alasan:** CSV murni tidak cukup karena tidak semua UID diketahui di muka. Kelas/Jurusan tidak di-auto-create dari CSV untuk mencegah duplikat akibat inkonsistensi penulisan di file sumber.
**Konsekuensi:** Modul Card & modul import butuh UI tambahan untuk flow tap-to-assign (real-time capture UID), bukan cuma form upload file. Estimasi effort fase 1 bertambah, tapi krusial untuk rollout awal yang realistis.

---

## ADR-007: Monorepo Turborepo, Bukan Polyrepo
**Tanggal:** 2026-06-25
**Status:** Accepted
**Konteks:** 3 developer perlu kelola backend, dashboard, dan kiosk app.
**Keputusan:** Satu repo GitHub, dikelola dengan Turborepo (`apps/api`, `apps/web`, `apps/kiosk`, `packages/types`, `packages/config`).
**Alasan:** Tim kecil (3 orang), perlu shared types package yang mudah diakses semua app tanpa publish package terpisah. Monorepo minim overhead koordinasi untuk skala ini.
**Konsekuensi:** Semua developer perlu nyaman kerja di satu repo besar — branch & PR harus disiplin scoped ke modul masing-masing (lihat 09-Conventions (09-Conventions.md)).

---

## ADR-010: Skema `students` + `teachers` Terpisah, Pola Dual-FK untuk Relasi Lintas-Tipe Orang
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Skema awal di 04-Database-Schema (04-Database-Schema.md) mengusulkan 1 tabel `persons` gabungan untuk siswa & guru dengan kolom generik. Field yang relevan untuk siswa (kelas, jurusan) dan guru (jadwal mengajar, mapel) cukup berbeda. Tabel `cards` (dan tabel lain yang relasi ke "orang") butuh cara menghubungkan ke salah satu dari keduanya.
**Keputusan:** `persons` dipecah jadi 2 tabel terpisah: `students` dan `teachers`. Relasi dari tabel lain ke salah satu dari keduanya menggunakan **2 kolom foreign key nullable** (misal `student_id`, `teacher_id` — tepat 1 yang terisi, sisanya null), **bukan** pola polymorphic generik (`owner_type` + `owner_id`).
**Alasan:** Field siswa dan guru cukup berbeda untuk dipisah jadi tabel sendiri-sendiri (query lebih sederhana, tidak ada kolom yang nullable untuk separuh baris). Pola dual-FK nullable dipilih di atas polymorphic association karena memungkinkan foreign key constraint **asli di level database** — pola polymorphic generik tidak bisa diberi constraint FK yang valid di MySQL, sehingga integritas data 100% bergantung pada logic aplikasi (rawan data yatim/orphan kalau ada bug).
**Konsekuensi:** Semua tabel yang butuh relasi ke "siswa ATAU guru" (`cards`, `schedules`, `attendance_records`, dst) memakai pola dual-FK nullable yang konsisten. Query yang butuh data gabungan siswa+guru (misal 1 laporan kehadiran semua orang) butuh `UNION` atau view gabungan di level query — trade-off yang diterima demi integritas data yang lebih kuat.

---

## ADR-011: Mesin Database MySQL, Bukan PostgreSQL — Supersede Sebagian ADR-002
**Tanggal:** 2026-06-26
**Status:** Accepted (men-supersede sebagian ADR-002)
**Konteks:** ADR-002 awalnya memilih PostgreSQL dengan alasan kebutuhan analitik kompleks (window function, partial index) untuk query rekap multi-dimensi. Setelah dikaji ulang, volume data riil AbsenSI (±500rb baris/tahun untuk `attendance_records`) jauh di bawah skala yang benar-benar membutuhkan keunggulan analitik PostgreSQL — MySQL 8 juga sudah mendukung window function, dan index komposit yang tepat sudah cukup untuk kebutuhan filter rekap di skala ini. Tim juga sudah lama familiar dengan MySQL dari proyek-proyek sebelumnya (DasiPelajar, SIMA-Sarpras), sementara di saat yang sama tim sedang belajar stack TypeScript/NestJS yang baru.
**Keputusan:** Database engine diganti dari PostgreSQL ke **MySQL** di AbsenSI, dan dijadikan standar untuk semua aplikasi ekosistem sekolah berikutnya. ORM tetap **Prisma** (mendukung MySQL dengan baik, tidak perlu ganti ORM).
**Alasan:** Mengurangi satu sumber kurva belajar baru di tengah tim yang sudah belajar paradigma backend Node/NestJS — keunggulan analitik PostgreSQL tidak signifikan di skala data AbsenSI. Konsistensi 1 jenis engine database di semua aplikasi ekosistem masa depan juga menyederhanakan operasional (tim cuma perlu kuasai dan rawat 1 jenis database, bukan dua).
**Konsekuensi:** Bagian ADR-002 yang menyebut PostgreSQL sebagai alasan teknis dianggap **superseded** oleh ADR ini — keputusan NestJS, Prisma, dan arsitektur modular monolith dari ADR-002 **tetap berlaku**, hanya database engine yang berubah. 02-Tech-Stack.md (02-Tech-Stack.md) perlu diupdate untuk merefleksikan MySQL, bukan PostgreSQL.

---

## ADR-012: Topologi Server — 1 Server Fisik dengan Separasi Logis per Aplikasi, Bukan Split-VM
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Visi jangka panjang "ekosistem aplikasi sekolah" memunculkan usulan awal: setiap aplikasi dijalankan di virtual server (VM) terpisah-pisah, dengan 1 VM "utama" sebagai penampung data gabungan yang disinkronkan tahunan. Usulan ini didiskusikan dan ditemukan berisiko: beban operasional (patching, monitoring, backup, keamanan) naik linear setiap ada aplikasi baru, padahal kapasitas maintenance tim terbatas (3 guru aktif part-time; 1 orang jadi penanggung jawab utama infrastruktur server fisik, 2 lainnya sebagai backup). Estimasi volume data (±500rb baris/tahun per aplikasi) juga jauh di bawah skala yang benar-benar membutuhkan pemisahan server fisik/virtual untuk alasan performa.
**Keputusan:** Semua aplikasi (AbsenSI dan aplikasi ekosistem berikutnya) berjalan di **1 server fisik**, dengan **separasi logis** — setiap aplikasi punya database/schema MySQL sendiri di server yang sama, **bukan** VM/server virtual terpisah per aplikasi.
**Alasan:** Separasi logis (schema/database berbeda per aplikasi) memberi isolasi yang memadai (1 aplikasi tidak bisa sembarangan akses database aplikasi lain, lewat pembatasan kredensial database) tanpa biaya operasional N-server (N sistem operasi yang harus dipatch, dimonitor, dan diamankan terpisah-pisah). Resource fisik server (RAM/CPU) juga dipakai lebih efisien karena dibagi secara dinamis antar aplikasi, bukan dikunci statis per VM yang masing-masing punya overhead OS sendiri. Keputusan ini juga konsisten dengan ADR-003 (modular monolith, hindari kompleksitas distributed system) — split-VM-per-aplikasi pada dasarnya memindahkan masalah yang sama yang sudah ditolak ADR-003, hanya dipindah dari level aplikasi ke level infrastruktur.
**Konsekuensi:** Kalau di masa depan ada aplikasi yang **terbukti lewat pengukuran nyata** (bukan spekulasi) butuh resource terisolasi, pemisahan server untuk aplikasi spesifik itu bisa dipertimbangkan ulang saat itu — bukan kebijakan default dari awal. Tim wajib disiplin memberi kredensial database yang dibatasi per aplikasi (1 aplikasi cuma punya akses ke schema/database miliknya sendiri), supaya separasi logis ini benar-benar menegakkan isolasi, bukan cuma di atas kertas.

---

## ADR-013: Data Warehouse Berkala + Arsip Dingin Tahunan + Backup Operasional Terpisah
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Kebutuhan melihat data gabungan lintas-aplikasi (misal laporan tahunan lintas sistem ekosistem sekolah) berhadapan dengan ADR-012 (database terpisah secara logis per aplikasi). Usulan awal (push data ke server utama setahun sekali) ditemukan berisiko: data jadi basi sampai 12 bulan, dan proses tahunan adalah kode yang jarang dijalankan sehingga bug di dalamnya baru ketahuan setahun kemudian — saat itu kemungkinan data sudah berubah/tidak bisa di-rollback. Kebutuhan disaster recovery operasional (kalau server utama tiba-tiba rusak) juga berbeda tujuannya dari kebutuhan arsip historis jangka panjang, dan keduanya sempat tercampur jadi 1 rencana.
**Keputusan:** Tiga mekanisme terpisah, masing-masing dengan tujuan sendiri:
1. **Database global (data warehouse)** — diisi lewat ETL/replikasi **satu arah**, berkala **bulanan atau per semester** (bukan tahunan), dari tiap database aplikasi. Database global bersifat **read-only** untuk keperluan laporan lintas-aplikasi — tidak pernah ditulis langsung oleh aplikasi mana pun.
2. **Arsip dingin tahunan ("server 1")** — backup historis murni dari database global, di-push manual setiap tahun ajaran baru, tidak digunakan untuk kebutuhan operasional aplikasi apa pun ("tempat penyimpanan mati").
3. **Backup operasional rutin** — backup harian/mingguan terjadwal dari tiap database aplikasi (terpisah dari arsip tahunan), untuk disaster recovery jangka pendek. Peningkatan opsional: MySQL native replication (master-replica) untuk recovery point objective yang lebih baik, jika kapasitas tim memungkinkan setup dan pemeliharaannya.
**Alasan:** Memisahkan 3 tujuan (laporan lintas-aplikasi, arsip historis jangka panjang, disaster recovery jangka pendek) mencegah 1 mekanisme dipaksa melayani semua tujuan sekaligus dengan trade-off buruk di semua sisi. Sinkronisasi satu arah & read-only di sisi global mencegah konsistensi data rusak akibat banyak sumber tulis ke 1 tujuan yang sama.
**Konsekuensi:** Tim perlu menyiapkan job ETL terjadwal (bulanan/semester) menggunakan tooling teruji (built-in MySQL replication, atau script ETL terjadwal via cron) — bukan logic push custom yang dibangun sendiri per aplikasi. Backup harian/mingguan untuk disaster recovery **wajib ada sebelum go-live Fase 1** — tidak boleh hanya mengandalkan arsip tahunan sebagai satu-satunya jaring pengaman (prinsip 3-2-1: minimal 3 copy data, di 2 jenis media berbeda, 1 di antaranya di luar sistem utama/offline).

---

## ADR-015: Struktur Kampus dan Role `guru_piket` dengan Scoping Per Kampus
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Sekolah punya 2 kampus fisik (Kampus 1, Kampus 2), masing-masing punya guru piket sendiri yang hanya boleh mengelola data siswa di kampusnya. Role yang sudah ada (`super_admin`, `card_admin`, `guru`, `kepsek`) semuanya berskala global atau cuma diri sendiri — tidak ada role yang dibatasi per lokasi/kampus.
**Keputusan:** Tambah tabel `kampus`. `kelas` diberi `kampus_id` (FK) — setiap kelas terikat ke 1 kampus, siswa mewarisi kampus lewat relasi ke kelasnya (tidak ada `kampus_id` duplikat di tabel `students`). Tambah role baru `guru_piket` di `users`, dengan kolom `kampus_id` (scope akses) dan `teacher_id` (profil guru terkait). Tap RFID di gerbang tetap diizinkan lintas-kampus (siswa Kampus 1 boleh tap di kiosk Kampus 2, karena lokasi ruang praktek tidak selalu sama dengan kampus asal) — `kiosk_id` yang sudah ada cukup untuk mencatat lokasi tap, tidak perlu validasi pembatas baru.
**Alasan:** Menaruh kampus di `kelas` (bukan di `students` langsung) mencegah risiko data tidak sinkron (siswa pindah kelas tapi kolom kampus lupa diupdate) — 1 sumber kebenaran. Dashboard Piket harus scoping berdasarkan kampus asal siswa (lewat kelas), bukan lokasi tap, supaya tap lintas-kampus tidak salah muncul di dashboard kampus yang salah.
**Konsekuensi:** Semua query Dashboard Piket wajib filter berdasarkan `kampus_id` akun yang login, ditegakkan di level API (bukan cuma UI), konsisten dengan ADR-008. Asumsi "1 kelas selalu di 1 kampus yang sama" perlu dipegang konsisten — kalau ternyata ada kelas yang berpindah kampus di kemudian hari, itu jadi perubahan data manual, bukan kasus yang didesain otomatis.

---

## ADR-016: Tabel `permits` untuk Status Kehadiran Manual (Izin/Sakit/Keluar) oleh Guru Piket
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Selain tap RFID otomatis di gerbang, ada jalur pencatatan kehadiran manual: guru piket menerima laporan lisan siswa (izin tidak masuk dengan alasan sakit/izin, atau izin keluar sekolah saat jam belajar) dan harus mencatatnya ke sistem. Ini jalur kedua yang menulis ke data kehadiran, terpisah dari tap kartu, dan berpotensi tumpang tindih/ambigu dengan status yang sudah ada dari tap (terutama kasus "lupa tap pulang" yang sudah jadi Open Question di absensi-gerbang.md (06-Features/absensi-gerbang.md)).
**Keputusan:** Tambah 1 tabel `permits` untuk menampung 2 jenis alur (`jenis`: `tidak_masuk` atau `keluar`), dengan UI berbeda untuk masing-masing (tombol cepat [Izin]/[Sakit] di daftar siswa untuk `tidak_masuk`; menu form terpisah untuk `keluar`). Setiap entri `permits` otomatis memperbarui `attendance_records` hari itu (status `izin`/`sakit`, atau `waktu_pulang` + `pulang_via: izin_piket` untuk kasus keluar tanpa kembali) — siswa **tidak perlu tap** saat menerima izin maupun saat keluar fisik.
**Alasan:** Memisahkan jalur manual (lewat `permits`) dari jalur otomatis (lewat tap) tapi tetap menyatukan dampaknya ke `attendance_records` yang sama, mencegah laporan salah klasifikasi (izin sah dibaca sebagai bolos/lupa tap). 1 tabel untuk 2 jenis (bukan 2 tabel terpisah) karena strukturnya cukup mirip dan keduanya sama-sama "izin yang disetujui piket," cuma beda kelengkapan field.
**Konsekuensi:** Modul Attendance harus punya logic baru untuk menerima override status dari `permits`, di atas logic tap yang sudah ada — perlu didesain agar prioritas/urutan precedence jelas (status dari `permits` tidak boleh ditimpa balik oleh tap yang terjadi setelahnya di hari yang sama, atau sebaliknya — aturan tegas ini menyusul saat breakdown task modul Attendance).

---

## ADR-017: Mekanisme Lock/Unlock Siswa yang Tidak Kembali dari Izin Keluar
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Siswa yang diberi izin keluar dengan rencana kembali, tapi tidak kembali/tidak ada konfirmasi sampai jam yang dijanjikan, perlu ditindaklanjuti sebelum dianggap "kembali normal" — risiko keselamatan & akuntabilitas (siswa di bawah umur meninggalkan pengawasan sekolah tanpa penyelesaian jelas).
**Keputusan:** Sistem **menandai** (tidak otomatis mengunci) siswa yang lewat jam kembali tanpa konfirmasi sebagai "perlu ditinjau" di Dashboard Piket. Guru piket **secara manual** memutuskan untuk mengunci (`students.locked_at` dst terisi) setelah peninjauan. Siswa terkunci ditolak tap di gerbang dengan pesan jelas ("Hubungi Guru Piket") dan tetap dicatat sebagai log percobaan (pola sama dengan tap kartu inactive di manajemen-kartu.md (06-Features/manajemen-kartu.md)). Proses BK (bimbingan konseling) tetap berjalan offline/manual seperti biasa — sistem hanya menyimpan catatan ringkas hasil proses tersebut. Guru piket yang membuka kunci setelah proses BK selesai.
**Alasan:** Mengunci otomatis tanpa peninjauan manusia berisiko mengunci siswa yang sebenarnya sudah kembali tapi belum ditandai (human error piket, bukan masalah siswa) — tindakan disipliner dengan konsekuensi nyata (siswa tidak bisa absen) tidak boleh sepenuhnya otomatis. Membangun workflow BK lengkap di dalam sistem dianggap berlebihan untuk kebutuhan yang sebenarnya sederhana — proses BK punya alur sendiri di luar sistem ini.
**Konsekuensi:** Modul Attendance & Kiosk perlu logic tambahan untuk cek status lock sebelum proses tap, dan UI kiosk perlu pesan error khusus untuk kasus ini (beda dari pesan UID tidak terdaftar). Dashboard Piket butuh 1 halaman/section baru: daftar siswa "perlu ditinjau" dan daftar siswa "terkunci".

---

## ADR-018: Reuse `print.php` yang Sudah Ada untuk Cetak Surat Izin Keluar
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Sekolah sudah punya mekanisme cetak surat izin yang berjalan baik di sistem lama (AppSheet) — printer thermal Blueprint ECO 58D terinstal sebagai printer Windows biasa, dipanggil lewat script `print.php` yang berjalan di server lokal yang selalu hidup (`10.10.10.100:8800`), menerima parameter via URL, render preview HTML, petugas klik print manual dari browser.
**Keputusan:** Dashboard Piket yang baru **memakai ulang** `print.php` yang sudah ada, bukan membangun mekanisme print baru. Sistem baru mengonstruksi URL dengan pola parameter yang sama (`petugas`, `tgl`, `nama`, `kls`, `alasan`, `ket`, `jamkembali`), ditambah 1 parameter baru `kode` untuk kode verifikasi unik per surat (ditambahkan sebagai antisipasi pemalsuan, meski belum pernah ada kasus nyata — keputusan preventif berbiaya rendah).
**Alasan:** `print.php` sudah terbukti bekerja dengan hardware fisik yang sama selama ini — membangun ulang integrasi printer dari nol untuk hardware yang sama adalah kerja duplikat tanpa manfaat. Reuse ini juga selaras dengan prinsip menghindari over-engineering yang sudah disepakati berkali-kali di proyek ini.
**Konsekuensi:** `print.php` perlu sedikit modifikasi untuk menerima & menampilkan parameter `kode` baru (akses ke source code sudah dikonfirmasi tersedia). Server yang menjalankan `print.php` (`10.10.10.100`) tetap berdiri independen dari server utama AbsenSI (ADR-012) untuk saat ini — konsolidasi infrastruktur ini dicatat sebagai technical debt opsional, bukan kebutuhan mendesak.

---

## ADR-019: Status Kehadiran Adalah Kewenangan Eksklusif Guru Piket
**Tanggal:** 2026-07-03
**Status:** Accepted
**Konteks:** Dua aktor berpotensi menyentuh data kehadiran: `super_admin` (punya akses koreksi data teknis) dan `guru_piket` (yang menerima laporan langsung dari siswa/orang tua dan mencatat status izin/sakit/alfa). Sebelumnya sempat diasumsikan bahwa admin pusat punya kewenangan penuh termasuk mengubah status kehadiran — ini berpotensi menghasilkan konflik bila admin dan piket berbeda pendapat soal status yang sama.
**Keputusan:** Status kehadiran (`izin`, `sakit`, `alfa`) adalah **kewenangan eksklusif `guru_piket`**. `super_admin` **tidak bisa mengubah** status kehadiran siswa. Domain kewenangan dipisah bersih: admin mengelola data teknis (kartu, akun user, jadwal, kelas, jurusan, konfigurasi sistem), piket mengelola status kehadiran harian.
**Alasan:** Piket adalah petugas yang menerima laporan langsung dari siswa dan orang tua — mereka punya informasi ground truth yang admin tidak punya saat koreksi dari jauh. Memisahkan domain sepenuhnya menghilangkan konflik kewenangan secara struktural (bukan dengan precedence rule yang rumit). Kalau ada sengketa status kehadiran, jalurnya adalah komunikasi dengan piket kampus yang bersangkutan — bukan override admin.
**Konsekuensi:** Endpoint API untuk `POST/PATCH` data `permits` dan update status `attendance_records` hanya dapat dipanggil oleh role `guru_piket`, ditegakkan di level guard NestJS (bukan sekadar disembunyikan di UI). `super_admin` tidak punya akses ke endpoint ini meskipun mereka tahu URL-nya — konsisten dengan ADR-008. Bila ada kebutuhan "koreksi darurat" status kehadiran di masa depan, mekanismenya adalah piket yang melakukan koreksi (bukan admin langsung ubah), agar trail aksi tetap jelas di `activity_log` (lihat ADR-020).

---

## ADR-020: Dua Layer Logging — Raw Tap Events + Activity Log Pengguna
**Tanggal:** 2026-07-03
**Status:** Accepted
**Konteks:** Sistem butuh dua jenis bukti yang berbeda tujuannya: (1) bukti forensik bahwa tap RFID fisik benar-benar terjadi — dibutuhkan untuk kasus siswa klaim sudah tap tapi tidak tercatat, atau investigasi tap yang ditolak; (2) bukti audit siapa pengguna yang melakukan aksi apa di sistem — dibutuhkan untuk akuntabilitas piket dan admin serta investigasi data yang tidak sesuai.
**Keputusan:** Dua tabel immutable terpisah:
- **`tap_events`** — log mentah setiap tap RFID, insert-only, tidak bisa diedit/dihapus dari aplikasi. Mencatat: UID kartu, `card_id` (nullable — null kalau UID tidak dikenal), `kiosk_id` (mesin scanner), `scanned_at` (timestamp server, bukan client), `result` (enum: `accepted` / `rejected_inactive` / `rejected_locked` / `rejected_unknown` / `rejected_duplicate`), dan `attendance_record_id` (FK nullable — terisi kalau tap berhasil).
- **`activity_log`** — log setiap aksi pengguna yang login, insert-only. Mencatat: `actor_id` (FK ke `users`), `action` (string: `permit.create`, `permit.confirm_kembali`, `student.lock`, `student.unlock`, `attendance.manual_pulang`, dsb), `target_type` + `target_id` (record yang diubah), `snapshot_before` + `snapshot_after` (JSON state sebelum/sesudah), `ip_address`, `created_at`.
**Alasan:** Satu log gabungan tidak bisa melayani dua tujuan sekaligus — tap system events tidak punya `actor_id` (tap tidak punya sesi user), sementara activity log tidak perlu menyimpan raw UID dan kiosk_id yang hanya relevan untuk forensik hardware. Memisahkan keduanya membuat query forensik dan query audit masing-masing sederhana tanpa kolom nullable yang membingungkan.
**Konsekuensi:** `tap_events.scanned_at` selalu menggunakan **timestamp server** (bukan timestamp dari kiosk client) — clock kiosk bisa drift dan tidak bisa dijadikan sumber kebenaran waktu. `tap_events` diperkirakan tumbuh ±5.000 baris/hari (2.500 siswa × 2 tap) = ±1,8 juta baris/tahun — masih manageable di MySQL dengan index pada `(kiosk_id, scanned_at)`, dan termasuk dalam scope ETL ke data warehouse (ADR-013) agar tidak terus tumbuh tanpa batas di database operasional. Kedua tabel ini tidak boleh punya endpoint DELETE atau UPDATE yang bisa diakses dari aplikasi — hanya INSERT yang diizinkan.

---

## ADR-025: Lock Otomatis Setelah 2x Terlambat — Amandemen ADR-017

**Tanggal:** 2026-07-20
**Status:** Accepted
**Supersedes (sebagian):** ADR-017 — menambahkan SATU pengecualian baru pada prinsip "lock selalu manual", bukan mencabut ADR-017 secara keseluruhan (lock untuk kasus "tidak kembali dari izin keluar" tetap manual sepenuhnya seperti sebelumnya).
**Konteks:** User (kepala sekolah/pemilik produk) secara eksplisit meminta efek jera otomatis untuk siswa yang terlambat berulang: begitu terdeteksi terlambat untuk kedua kalinya, kartu langsung terkunci saat itu juga tanpa menunggu tindakan piket — beda dari kasus "tidak kembali dari izin" (ADR-017) yang memang sengaja dibuat manual karena risikonya (siswa terkunci keliru akibat piket lupa update) lebih besar dari manfaat otomatisasi.
**Keputusan:**
1. Sistem menghitung jumlah tap dengan `status = terlambat` milik seorang siswa sejak **counter terakhir direset** (lihat poin 3). Tidak perlu kolom counter terpisah — dihitung dari `COUNT(attendance_records WHERE studentId = X AND status = 'terlambat' AND tanggal > lastResetAt)` saat tap terjadi, konsisten dengan prinsip "jangan simpan yang bisa dihitung" yang sudah dipakai untuk alfa.
2. Begitu hasil hitung mencapai **2** pada tap yang baru saja diproses (yaitu tap ini sendiri adalah keterlambatan kedua), sistem **langsung mengunci** siswa tersebut sebagai bagian dari alur `tap()` yang sama — `lockedAt` diisi `now()`, `lockedReason` diisi otomatis (misal `"Terlambat 2 kali — hubungi orang tua"`), `lockedById` **null** (bukan piket manapun, sistem yang mengunci). Layar kiosk saat itu juga menampilkan pesan **"Sudah terlambat 2 kali, silahkan hadirkan orang tua"** — bukan pesan tap-diterima biasa, meski tap tetap tercatat sebagai `accepted`/`terlambat` di `attendance_records` (siswa tetap tercatat hadir-terlambat, cuma kartunya langsung nonaktif untuk tap berikutnya).
3. **Reset counter:** hanya terjadi saat guru piket melakukan **unlock** siswa ini (bukan periodik per semester/tahun ajaran). Field baru `student.lateStrikeResetAt` (nullable, default null) diisi `now()` setiap kali `unlock()` dipanggil untuk siswa dengan alasan lock ini — perhitungan poin 1 di atas memakai `tanggal > lateStrikeResetAt` (kalau null, hitung dari awal data).
4. Alur unlock yang sudah ada (ADR-017, dialog piket) dipakai ulang tanpa perlu UI terpisah — piket unlock seperti biasa setelah orang tua hadir; hanya bedanya sistem juga menulis `lateStrikeResetAt = now()` di baliknya untuk lock jenis ini.
**Alasan:** Reset-by-unlock (bukan reset periodik) dipilih karena mencerminkan maksud aslinya: 2 kesempatan itu berlaku sampai insiden itu "diselesaikan" (orang tua dihadirkan), bukan berlaku untuk jendela waktu kalender yang arbitrer. Lock otomatis (bukan cuma badge/saran ke piket) dipilih karena user eksplisit menginginkan efek jera instan tanpa jeda menunggu piket bertindak — ini beda konteks dari ADR-017 (izin tidak kembali) yang risikonya salah-kunci jauh lebih tinggi (piket sering belum tahu situasi lapangan), sedangkan "2x terlambat" adalah fakta objektif dari data tap itu sendiri, tidak butuh penilaian manusia untuk memastikan kebenarannya.
**Konsekuensi:**
- `Student` +1 kolom: `lateStrikeResetAt DateTime?`.
- `AttendanceService.tap()` — setelah `determineStatus()` mengembalikan `terlambat` dan record berhasil dibuat, hitung ulang jumlah keterlambatan sejak `lateStrikeResetAt`; kalau mencapai 2, panggil logic lock (bisa reuse sebagian `StudentsService.lock()` tapi dengan `lockedById: null` — perlu cek apakah kolom itu nullable atau perlu diubah jadi nullable, karena assignment sebelumnya asumsi selalu ada piket yang mengunci).
- `TapResultPayload`/`TapResponse` (`@absensi/types`) perlu varian pesan baru untuk kasus ini — kemungkinan `result` tetap `accepted` tapi flag tambahan `locked: true` + `message` khusus, ATAU `result` baru `accepted_then_locked` — **desain persis diputuskan saat implementasi**, jangan asumsikan dari sini.
- `apps/kiosk` — layar feedback perlu varian tampilan ke-3 (bukan cuma accepted/rejected): "diterima tapi langsung terkunci", dengan pesan spesifik yang diminta user persis.
- Dashboard Piket: siswa yang terkunci lewat mekanisme ini tetap muncul di section "Siswa Terkunci" yang sudah ada (ADR-017) — `lockedReason` yang beda (otomatis vs manual) sudah cukup untuk membedakan tanpa perlu section terpisah.
- ADR-017 **tidak dicabut** — tetap berlaku penuh untuk kasus "izin keluar tidak kembali". Kalau nanti ditemukan kebutuhan lock-otomatis jenis ketiga, pertimbangkan apakah perlu digeneralisasi jadi satu mekanisme "lock reasons" yang eksplisit, bukan ditambah kondisi khusus lagi satu-satu.

---

## ADR-024: Jadwal Hari Piket per Guru — Login Dibatasi Hari Aktif

**Tanggal:** 2026-07-20
**Status:** Accepted
**Konteks:** Saat ini akun `guru_piket` bisa login kapan saja tanpa pembatasan hari — tidak ada konsep "hari piket" di skema sama sekali (dikonfirmasi tidak ada field terkait di model `User`). User meminta agar tiap guru piket cuma bertugas (dan bisa login efektif) pada hari-hari tertentu yang di-assign admin, dengan UI mirip Google Calendar (grid Senin–Sabtu, klik hari untuk assign guru piket ke hari itu).
**Keputusan:**
1. Tabel baru `piket_schedules`: `id`, `hari` (Int, 1=Senin..6=Sabtu — **beda dari konvensi `Schedule.hari` yang sudah ada** yang pakai basis MySQL DAYOFWEEK 1=Minggu; perlu keputusan eksplisit yang mana dipakai di sini, dicatat sebagai open point di task, **jangan asumsikan sama dengan basis lama begitu saja**), `userId` (FK ke `users`, harus role `guru_piket`), `createdById`, `createdAt`. Satu hari bisa punya lebih dari satu guru piket (banyak-ke-banyak secara implisit lewat baris terpisah per hari+user).
2. Guru piket yang login pada hari **di luar** jadwalnya: login tetap **berhasil** (tidak ditolak di endpoint `/auth/login`), tapi begitu masuk Dashboard Piket, semua **aksi tulis dinonaktifkan** (lock/unlock, buat izin, konfirmasi kembali, dsb) — halaman jadi read-only. Pesan jelas ditampilkan (misal banner "Anda tidak bertugas piket hari ini — mode lihat saja").
3. UI admin baru: halaman kalender mingguan (grid 6 kolom Senin–Sabtu, TANPA Minggu — pola visual mengambil referensi dari `apps/web/src/app/(admin)/kalender/kalender-view.tsx` yang sudah ada, tapi ini bukan kalender bulanan/tahun-ajaran, jadi komponennya baru, cuma mengambil gaya visual). Klik satu hari → popup pilih guru piket (dari daftar user role `guru_piket`) untuk di-assign/dilepas dari hari itu. Nama guru piket yang sudah ter-assign tampil langsung di dalam kotak hari tersebut (gaya Google Calendar).
**Alasan:** Read-only (bukan block-login-total) dipilih karena guru piket kadang perlu memantau situasi kampusnya di hari libur piketnya sendiri (misal cek apakah ada masalah) tanpa perlu mengganggu alur tugas piket hari itu yang sedang aktif dengan aksi yang tidak semestinya dari orang yang bukan bertugas — ini keputusan yang lebih aman daripada menolak akses sepenuhnya, sekaligus tetap mencegah aksi tulis yang salah pihak.
**Konsekuensi:**
- Endpoint baru dibutuhkan: `GET/POST/DELETE /piket-schedules` (admin kelola), `GET /piket-schedules/me/today` atau serupa (dipanggil `(piket)/layout.tsx` untuk tahu apakah user ini bertugas hari ini).
- `(piket)/layout.tsx` dan semua Server Action/Route Handler penulisan (lock, unlock, buat izin, dst.) perlu cek status "bertugas hari ini" sebelum eksekusi — bukan cuma UI yang disembunyikan, validasi juga harus di backend (guard baru atau cek eksplisit di service), supaya guru piket tidak bisa bypass read-only lewat panggilan API langsung.
- Definisi "hari ini" harus konsisten timezone — proyek ini belum eksplisit menyatakan timezone standar di ADR manapun, perlu diperjelas saat implementasi (kemungkinan `Asia/Jakarta` mengikuti lokasi sekolah, tapi cek dulu bagaimana `startOfDay()`/tanggal lain di `attendance.service.ts` menangani ini sebelum menambah konvensi baru yang beda).

---

## ADR-023: Foto Profil — Disk Lokal + Upload Bulk Auto-Match by Filename (NISN/NIY)

**Tanggal:** 2026-07-17
**Status:** Accepted
**Konteks:** T028 menambahkan foto profil siswa & guru (ditampilkan di layar kiosk saat scan). Perlu diputuskan cara penyimpanan file dan alur upload untuk ratusan foto sekaligus (bukan satu-satu manual per siswa).
**Keputusan:**
1. **Storage:** foto disimpan sebagai file di disk lokal server (folder di dalam `apps/api`, di luar `dist`/`src` — misal `apps/api/storage/photos/`), path relatif disimpan di kolom `foto` (`students`/`teachers`). **Bukan** object storage cloud (S3, dsb).
2. **Alur upload:** admin pilih banyak file sekaligus (bulk, mirip `ImportService` CSV yang sudah ada). Server cocokkan **nama file (tanpa ekstensi) = NISN (siswa) atau NIY (guru)** secara otomatis. File yang cocok langsung ter-assign ke siswa/guru terkait; file yang tidak match dilaporkan ke admin untuk di-assign manual lewat dropdown pencarian siswa/guru di UI.
**Alasan:** Disk lokal dipilih karena skala project ini on-premise single-server (konsisten dengan MySQL & Redis lokal, tidak ada infra cloud storage yang sudah ada) — menambah dependency S3-compatible storage untuk ribuan foto ukuran kecil adalah over-engineering di tahap ini. Auto-match by filename dipilih karena operator sekolah realistis akan punya folder foto hasil scan/kamera yang sudah dinamai sesuai NISN/NIY siswa (pola umum di sistem sejenis), sehingga upload 500+ foto sekaligus tidak perlu 500 kali klik manual — fallback manual-assign tetap disediakan untuk file yang gagal auto-match (nama file typo, siswa belum ada di sistem, dsb).
**Konsekuensi:**
- Endpoint upload baru: terima `multipart/form-data` banyak file sekaligus, proses satu-satu (mirip pola `ImportService.importStudents()`), return laporan `{ matched: [...], unmatched: [...] }`.
- Validasi tipe file (JPEG/PNG saja) dan batas ukuran **1MB per file** (diputuskan final 2026-07-17) — cukup untuk foto potret kualitas layar kiosk, jaga total volume storage tetap kecil.
- Volume disk bertambah seiring jumlah siswa/guru (skala ±2.500 siswa × rata-rata foto beberapa ratus KB = order tunggal digit GB, tidak signifikan untuk server sekolah).
- Backup database **tidak** mencakup foto (foto ada di filesystem, bukan di kolom BLOB) — folder `storage/photos/` harus ikut masuk strategi backup terpisah (dicatat sebagai catatan operasional, bukan blocker implementasi).
- Kalau nanti sekolah pindah ke infra multi-server/container ephemeral, keputusan disk lokal ini harus dibuka ulang (foto akan hilang kalau container di-recreate tanpa persistent volume) — untuk saat ini diasumsikan deployment tetap single-server dengan disk persisten.

---

## ADR-022: Kiosk Bertipe (Siswa/Guru) — Fisik Terpisah, Kartu Salah Tipe Ditolak

**Tanggal:** 2026-07-17
**Status:** Accepted
**Konteks:** Fase data profil (T028) menambahkan tampilan scan berbeda untuk siswa vs guru/karyawan di layar kiosk (siswa: foto+nama+jam; guru: foto+nama+jam + 2 tabel "5 terbaru datang/pulang"). Perlu diputuskan bagaimana kiosk tahu tampilan mana yang harus ditampilkan, dan apa yang terjadi kalau kartu siswa di-tap di kiosk yang salah (atau sebaliknya).
**Keputusan:**
1. Setiap kiosk (tabel `kiosks`, ADR-021) punya kolom `tipe` (`siswa` | `guru`), diisi admin saat registrasi kiosk — **bukan** dideteksi otomatis dari kartu yang di-tap. Kiosk secara fisik didedikasikan untuk satu gerbang/tipe (mis. gerbang siswa vs gerbang guru), konsisten dengan model ADR-021 (kiosk = device fisik dengan IP tetap).
2. `KioskGuard` (atau `AttendanceService.tap()`) memvalidasi tipe pemilik kartu (siswa/guru) terhadap `kiosk.tipe`. Kartu siswa di-tap di kiosk `tipe=guru` (atau sebaliknya) → **ditolak**, direkam di `tap_events` dengan `result=rejected_wrong_kiosk_type` (insert-only, forensik — bukan diam-diam diabaikan), **tidak** membuat/mengubah `attendance_record`. Layar kiosk tampilkan pesan "Kartu ini bukan untuk gerbang ini".
**Alasan:** Pemisahan fisik kiosk per tipe mencerminkan kenyataan operasional sekolah — gerbang siswa dan ruang guru adalah lokasi fisik berbeda, device kiosknya pun berbeda. Deteksi otomatis dari kartu tidak dipilih karena akan membuat satu device melayani dua UI berbeda tergantung siapa yang tap, yang lebih rumit untuk kiosk mode kios (fullscreen, tanpa navigasi) dan tidak sesuai kebutuhan mengunci akses fisik per gerbang (siswa tidak boleh tap di gerbang guru meski secara teknis bisa).
**Konsekuensi:**
- `kiosks.tipe` wajib diisi saat admin membuat kiosk baru — form Tambah Kiosk (T027 UI, belum dikerjakan) perlu field ini.
- `TapResult` enum tambah varian `rejected_wrong_kiosk_type`.
- `apps/kiosk` tidak perlu logic deteksi tipe dari kartu — device kiosk sendiri sudah tahu tipenya (dari konfigurasi/URL registrasi kiosk saat setup, sama seperti token ADR-021), dipakai untuk pilih varian tampilan feedback screen.
- Kalau sekolah nanti butuh 1 device melayani siswa+guru sekaligus (misal gerbang tunggal kecil), keputusan ini harus dibuka ulang — saat ini diasumsikan gerbang siswa dan akses guru/karyawan selalu punya titik masuk fisik terpisah.

---

## ADR-021: Kiosk Auth — URL Token + IP Whitelist, Token Disimpan di Database
**Tanggal:** 2026-07-15
**Status:** Accepted
**Supersedes:** Bagian kiosk auth di ADR-008 (static env token `KIOSK_DEVICE_TOKEN`)
**Konteks:** Desain awal (ADR-008) menggunakan satu static device token yang disimpan di `.env` kiosk. Masalahnya: setiap kiosk baru butuh akses fisik ke device untuk edit `.env` dan restart app — ini ribet secara operasional, terutama saat ganti device atau rotasi token. Selain itu, satu token tanpa lapisan lain berarti siapapun yang tahu token URL bisa kirim request dari device lain di jaringan yang sama.
**Keputusan:** Auth kiosk menggunakan dua lapis:
1. **Token per-kiosk dari URL** — admin buat kiosk di dashboard, server generate token unik, dashboard tampilkan URL lengkap + QR code (misal `http://server/kiosk?device=TOKEN`). IT/operator set URL ini sebagai homepage browser kiosk — sekali saja. Token di-extract dari URL dan disimpan ke `localStorage` kiosk; semua request ke API menyertakan token ini di header `Authorization: Bearer`. Token disimpan di tabel `kiosks` di database (bukan `.env` server), sehingga manajemen token (buat, revoke, rotasi) dilakukan dari admin dashboard tanpa menyentuh file konfigurasi server.
2. **IP whitelist per-kiosk** — setiap device kiosk diberi IP static di jaringan sekolah. IP ini diinput admin saat registrasi kiosk. `KioskGuard` memvalidasi `request.ip` terhadap `kiosks.allowed_ip` di database. Kombinasi: token yang bocor tidak bisa dipakai dari device lain (IP tidak cocok); akses dari IP kiosk yang benar tanpa token tetap ditolak.
**Alasan:** IP-only tidak cukup (siapapun di LAN sekolah bisa kirim request dari IP yang sama subnet). Token-only tanpa IP membuat token yang bocor bisa dipakai dari device apapun di jaringan. Kombinasi keduanya membutuhkan kompromi dua lapis sekaligus, yang di lingkungan fisik sekolah (akses ke ruang kiosk = akses fisik yang sudah terlindungi) secara praktis sangat sulit. Setup operasional jauh lebih mudah dari desain lama: setup kiosk baru = set homepage browser + set IP static di OS, tidak perlu SSH atau edit file `.env`.
**Konsekuensi:**
- Tabel baru `kiosks` ditambahkan ke schema: `id`, `nama`, `kampus_id` (FK), `device_token` (random 256-bit), `allowed_ip`, `last_seen_at`, `last_seen_ip`, `is_active`, `created_by`, `created_at`.
- `KioskGuard` direfaktor: tidak lagi baca `KIOSK_DEVICE_TOKEN` dari env — ganti ke query tabel `kiosks` berdasarkan token dari header, lalu validasi `allowed_ip`.
- `KIOSK_DEVICE_TOKEN` dihapus dari `.env` dan dari `10-Environment.md`.
- `kiosk_id` di `tap_events` terisi dari `kiosks.id` yang ditemukan saat validasi guard — tidak lagi self-reported dari request body kiosk.
- Rotasi token kiosk cukup dari admin dashboard (generate token baru, salin URL baru ke homepage browser) — tanpa deploy ulang atau SSH.
- `last_seen_at` diupdate setiap tap sukses — dashboard admin bisa tampilkan status online/offline (threshold: `last_seen_at > 5 menit = offline`).

---

## ADR-014: Master Data (Core) Tetap di Dalam AbsenSI, Ekstraksi Servis Terpisah Ditunda
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Semua aplikasi ekosistem sekolah masa depan akan butuh data master yang sama (siswa, guru, kelas, jurusan, jadwal). Muncul usulan untuk memisahkan data ini jadi "Master Data Service" tersendiri (database + API independen) yang dikonsumsi semua aplikasi dari awal. Namun aplikasi ke-2 dalam ekosistem ini **belum konkret ada** — belum punya spesifikasi, belum jelas data apa persis yang dibutuhkan, seberapa sering diakses, dalam bentuk/kontrak apa.
**Keputusan:** Master data (siswa, guru, jadwal, kelas, jurusan — modul **Core**) **tetap berada di dalam AbsenSI** sebagai modul internal, sesuai ADR-003 (modular monolith). **Tidak diekstrak** jadi servis atau database terpisah sekarang. Batas modul Core dijaga bersih — semua akses ke data Core, baik dari modul lain di AbsenSI maupun (nanti) dari aplikasi lain, **wajib lewat service layer/API yang disediakan Core**, tidak pernah lewat query langsung ke tabel Core.
**Alasan:** Mendesain Master Data Service independen sekarang berarti merancang kontrak API untuk konsumen yang masih imajiner (aplikasi ke-2 belum ada) — risiko nyata desainnya salah bentuk dan harus dirombak ulang begitu aplikasi ke-2 benar-benar mulai dibangun dengan kebutuhan riil. Menjaga batas modul yang bersih di dalam AbsenSI sekarang (biaya rendah, sudah jadi konvensi tim di ADR-003 & 09-Conventions (09-Conventions.md)) membuat ekstraksi nanti — saat benar-benar dibutuhkan — jauh lebih murah dilakukan, dibanding membangun servis terpisah sekarang berdasarkan spekulasi.
**Konsekuensi:** Saat aplikasi ke-2 mulai konkret dirancang, keputusan ini **harus dibuka kembali**: apakah aplikasi ke-2 cukup memanggil API yang AbsenSI sediakan untuk data Core, atau memang sudah waktunya Core diekstrak jadi servis independen dengan database sendiri. Sampai saat itu tiba, tidak ada pekerjaan tambahan yang perlu dilakukan di luar disiplin menjaga service layer Core tetap bersih dan tidak dilanggar modul lain.

