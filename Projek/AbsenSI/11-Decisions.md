---
tags: [absensi, adr, decisions]
updated: 2026-06-25
---

# 11 — Decisions (ADR)

← [[Projek/AbsenSI/00-INDEX|Index]]

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
- **Risiko diakui secara sadar:** 2 dari 3 developer belum familiar paradigma backend Node/Nest. Mitigasi: pembagian modul disesuaikan kekuatan masing-masing (lihat [[Projek/AbsenSI/_claudian/team|team.md]]), modul Core/paling kritis dipegang developer paling fullstack.
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
**Konsekuensi:** Modul **Schedule** harus sudah punya struktur jadwal mengajar guru dari fase 1 (untuk hitung "jam mengajar pertama"), meskipun reader kelas belum ada. Desain skema DB harus mendukung kedua fase tanpa rebuild — lihat [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]].

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
**Konsekuensi:** Semua developer perlu nyaman kerja di satu repo besar — branch & PR harus disiplin scoped ke modul masing-masing (lihat [[Projek/AbsenSI/09-Conventions|09-Conventions]]).

---

## ADR-010: Skema `students` + `teachers` Terpisah, Pola Dual-FK untuk Relasi Lintas-Tipe Orang
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Skema awal di [[Projek/AbsenSI/04-Database-Schema|04-Database-Schema]] mengusulkan 1 tabel `persons` gabungan untuk siswa & guru dengan kolom generik. Field yang relevan untuk siswa (kelas, jurusan) dan guru (jadwal mengajar, mapel) cukup berbeda. Tabel `cards` (dan tabel lain yang relasi ke "orang") butuh cara menghubungkan ke salah satu dari keduanya.
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
**Konsekuensi:** Bagian ADR-002 yang menyebut PostgreSQL sebagai alasan teknis dianggap **superseded** oleh ADR ini — keputusan NestJS, Prisma, dan arsitektur modular monolith dari ADR-002 **tetap berlaku**, hanya database engine yang berubah. [[Projek/AbsenSI/02-Tech-Stack|02-Tech-Stack.md]] perlu diupdate untuk merefleksikan MySQL, bukan PostgreSQL.

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
**Konteks:** Selain tap RFID otomatis di gerbang, ada jalur pencatatan kehadiran manual: guru piket menerima laporan lisan siswa (izin tidak masuk dengan alasan sakit/izin, atau izin keluar sekolah saat jam belajar) dan harus mencatatnya ke sistem. Ini jalur kedua yang menulis ke data kehadiran, terpisah dari tap kartu, dan berpotensi tumpang tindih/ambigu dengan status yang sudah ada dari tap (terutama kasus "lupa tap pulang" yang sudah jadi Open Question di [[Projek/AbsenSI/06-Features/absensi-gerbang|absensi-gerbang.md]]).
**Keputusan:** Tambah 1 tabel `permits` untuk menampung 2 jenis alur (`jenis`: `tidak_masuk` atau `keluar`), dengan UI berbeda untuk masing-masing (tombol cepat [Izin]/[Sakit] di daftar siswa untuk `tidak_masuk`; menu form terpisah untuk `keluar`). Setiap entri `permits` otomatis memperbarui `attendance_records` hari itu (status `izin`/`sakit`, atau `waktu_pulang` + `pulang_via: izin_piket` untuk kasus keluar tanpa kembali) — siswa **tidak perlu tap** saat menerima izin maupun saat keluar fisik.
**Alasan:** Memisahkan jalur manual (lewat `permits`) dari jalur otomatis (lewat tap) tapi tetap menyatukan dampaknya ke `attendance_records` yang sama, mencegah laporan salah klasifikasi (izin sah dibaca sebagai bolos/lupa tap). 1 tabel untuk 2 jenis (bukan 2 tabel terpisah) karena strukturnya cukup mirip dan keduanya sama-sama "izin yang disetujui piket," cuma beda kelengkapan field.
**Konsekuensi:** Modul Attendance harus punya logic baru untuk menerima override status dari `permits`, di atas logic tap yang sudah ada — perlu didesain agar prioritas/urutan precedence jelas (status dari `permits` tidak boleh ditimpa balik oleh tap yang terjadi setelahnya di hari yang sama, atau sebaliknya — aturan tegas ini menyusul saat breakdown task modul Attendance).

---

## ADR-017: Mekanisme Lock/Unlock Siswa yang Tidak Kembali dari Izin Keluar
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Siswa yang diberi izin keluar dengan rencana kembali, tapi tidak kembali/tidak ada konfirmasi sampai jam yang dijanjikan, perlu ditindaklanjuti sebelum dianggap "kembali normal" — risiko keselamatan & akuntabilitas (siswa di bawah umur meninggalkan pengawasan sekolah tanpa penyelesaian jelas).
**Keputusan:** Sistem **menandai** (tidak otomatis mengunci) siswa yang lewat jam kembali tanpa konfirmasi sebagai "perlu ditinjau" di Dashboard Piket. Guru piket **secara manual** memutuskan untuk mengunci (`students.locked_at` dst terisi) setelah peninjauan. Siswa terkunci ditolak tap di gerbang dengan pesan jelas ("Hubungi Guru Piket") dan tetap dicatat sebagai log percobaan (pola sama dengan tap kartu inactive di [[Projek/AbsenSI/06-Features/manajemen-kartu|manajemen-kartu.md]]). Proses BK (bimbingan konseling) tetap berjalan offline/manual seperti biasa — sistem hanya menyimpan catatan ringkas hasil proses tersebut. Guru piket yang membuka kunci setelah proses BK selesai.
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

## ADR-014: Master Data (Core) Tetap di Dalam AbsenSI, Ekstraksi Servis Terpisah Ditunda
**Tanggal:** 2026-06-26
**Status:** Accepted
**Konteks:** Semua aplikasi ekosistem sekolah masa depan akan butuh data master yang sama (siswa, guru, kelas, jurusan, jadwal). Muncul usulan untuk memisahkan data ini jadi "Master Data Service" tersendiri (database + API independen) yang dikonsumsi semua aplikasi dari awal. Namun aplikasi ke-2 dalam ekosistem ini **belum konkret ada** — belum punya spesifikasi, belum jelas data apa persis yang dibutuhkan, seberapa sering diakses, dalam bentuk/kontrak apa.
**Keputusan:** Master data (siswa, guru, jadwal, kelas, jurusan — modul **Core**) **tetap berada di dalam AbsenSI** sebagai modul internal, sesuai ADR-003 (modular monolith). **Tidak diekstrak** jadi servis atau database terpisah sekarang. Batas modul Core dijaga bersih — semua akses ke data Core, baik dari modul lain di AbsenSI maupun (nanti) dari aplikasi lain, **wajib lewat service layer/API yang disediakan Core**, tidak pernah lewat query langsung ke tabel Core.
**Alasan:** Mendesain Master Data Service independen sekarang berarti merancang kontrak API untuk konsumen yang masih imajiner (aplikasi ke-2 belum ada) — risiko nyata desainnya salah bentuk dan harus dirombak ulang begitu aplikasi ke-2 benar-benar mulai dibangun dengan kebutuhan riil. Menjaga batas modul yang bersih di dalam AbsenSI sekarang (biaya rendah, sudah jadi konvensi tim di ADR-003 & [[Projek/AbsenSI/09-Conventions|09-Conventions]]) membuat ekstraksi nanti — saat benar-benar dibutuhkan — jauh lebih murah dilakukan, dibanding membangun servis terpisah sekarang berdasarkan spekulasi.
**Konsekuensi:** Saat aplikasi ke-2 mulai konkret dirancang, keputusan ini **harus dibuka kembali**: apakah aplikasi ke-2 cukup memanggil API yang AbsenSI sediakan untuk data Core, atau memang sudah waktunya Core diekstrak jadi servis independen dengan database sendiri. Sampai saat itu tiba, tidak ada pekerjaan tambahan yang perlu dilakukan di luar disiplin menjaga service layer Core tetap bersih dan tidak dilanggar modul lain.

