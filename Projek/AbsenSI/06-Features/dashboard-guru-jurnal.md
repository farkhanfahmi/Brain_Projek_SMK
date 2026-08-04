---
tags: [absensi, feature, guru, jurnal, kelas, fase-2, planning]
status: planning-interview
updated: 2026-07-21
---

# Feature — Dashboard Guru: Jurnal Mengajar & Absensi Mapel (Fase 2 — Prioritas #1)

← Index (00-INDEX AbsenSI.md)

> **Status: masih interview/diskusi, BELUM final.** Dokumen ini menggantikan pendekatan lama di absensi-kelas-mapel.md (06-Features/absensi-kelas-mapel.md) (tap RFID di tiap kelas). Jangan mulai coding dari dokumen ini sampai status berubah jadi `final`.

---

## 🔄 Kenapa Ganti Pendekatan dari Rencana Lama

Rencana lama (`absensi-kelas-mapel.md`) mengasumsikan reader RFID fisik di tiap ruang kelas, dengan siswa wajib tap gerbang dulu sebagai **hard block** sebelum bisa tap kelas. Blocker yang tidak pernah terpecahkan: gerbang sekolah tidak punya penghalang fisik (bukan turnstile), jadi siswa yang fisik hadir tapi lupa tap gerbang akan salah ditolak.

**Keputusan baru:** tidak ada tap RFID di kelas sama sekali. Absensi per mapel diturunkan dari data tap gerbang yang sudah ada (asumsi: siswa yang tap gerbang = hadir di sekolah), dikoreksi manual oleh guru (siapa yang secara fisik tidak ada di kelas itu) dan guru piket (izin/sakit/alfa resmi). Ini menghilangkan seluruh masalah "hard block gerbang" karena tidak ada lagi gerbang yang jadi syarat keras.

Referensi pembanding: aplikasi [Jurnale PRO](https://pro.jurnale.id/panduan/?p=pendahuluan) yang pernah dipakai sekolah. Kekurangan utamanya: jadwal blok tidak mendukung rotasi minggu ganjil/genap (Minggu A/B) yang dipakai sekolah — sistemnya cuma "jam ke mulai–selesai" dalam 1 hari, tidak ada konsep pola mingguan bergantian.

---

## 🎯 Konsep Inti

1. Guru login dashboard (password biasa, **tidak** digate oleh tap gerbang — tap gerbang cuma syarat untuk aksi "Mulai Mengajar", bukan syarat login)
2. Dashboard tampilkan jadwal mengajar **hari ini** otomatis, resolve dari: hari + mode jadwal aktif (Blok A/B atau Normal, lihat bawah) + slot yang cocok
3. Sesi sedang berlangsung / akan berlangsung di-highlight; guru klik untuk mulai
4. Tiap slot jadwal punya **dua tombol**: **"Mulai Mengajar"** dan **"Izin"** (lihat bagian [[#🙋 Izin Guru Tidak Mengajar]] untuk detail tombol kedua)
5. Slot hanya bisa diklik "Mulai Mengajar" kalau **ketiga syarat** terpenuhi:
   - Guru sudah tap gerbang hari itu
   - Waktu sekarang berada dalam jam jadwal slot tsb (dengan toleransi keterlambatan configurable)
   - GPS aktif & berada dalam radius geofence kampus tempat kelas itu berada
6. Klik "Mulai Mengajar" → tercatat sebagai kehadiran guru di kelas itu (dengan koordinat lokasi), sekaligus membuka sesi jurnal
7. Daftar siswa kelas muncul otomatis, default status = hadir (mengikuti data tap gerbang hari itu). Guru koreksi manual siapa yang tidak ada secara fisik di kelasnya (status terpisah dari izin/sakit/alfa resmi milik piket)
8. Guru isi jurnal: materi, tujuan pembelajaran, tugas/penilaian, catatan/kendala
9. Sesi auto-close begitu jam pelajaran terjadwal berakhir sesuai jadwal

---

## 📅 Jadwal: Mode Blok (Minggu A/B) vs Mode Normal

> **Revisi final (2026-07-22) — MENGGANTIKAN TOTAL mekanisme "titik acuan + hitung selisih minggu" yang dijelaskan sebelumnya.** Jangan bingung dengan draf awal: tidak ada lagi `minggu_acuan_tanggal`/`minggu_acuan_nilai` dihitung otomatis. Admin sekarang input **rentang tanggal eksplisit** untuk setiap minggu A dan setiap minggu B, per semester. Ini keputusan sadar untuk menghindari kesalahan sistem dari perhitungan otomatis (misal geser akibat libur panjang yang tidak terantisipasi formula).

> Minggu A/B tetap berlaku **global untuk seluruh sekolah** (satu jadwal blok per sekolah, bukan per kelas) — hanya ISI jadwalnya (mapel, jam, guru) yang tetap per kelas seperti biasa.

### Mekanisme Baru: `block_week_ranges` (Rentang Eksplisit per Semester)

- Sistem punya **toggle mode jadwal** per semester: **Mode Blok (A/B)** atau **Mode Normal** — field ini sekarang di level `semesters` (bukan konfigurasi sekolah global tunggal seperti draf awal), karena semester ganjil bisa mode blok sementara semester genap (yang sedang disiapkan) belum tentu ikut mode yang sama
- **Mode Normal**: 1 slot jadwal berlaku setiap minggu dalam semester itu, tidak ada konsep A/B sama sekali
- **Mode Blok**: admin input **banyak pasangan rentang tanggal** dalam 1 semester — tiap rentang ditandai eksplisit `A` atau `B`. Contoh nyata: "21–27 Jul 2026 = A", "28 Jul–3 Ags 2026 = B", "4–10 Ags 2026 = A", dst, berselang-seling sepanjang semester (bukan cuma 2 rentang besar — bisa belasan pasangan per semester)
- Sistem resolve jadwal aktif hari ini = hari + cari tanggal hari ini masuk rentang mana (`A`/`B`) dari `block_week_ranges` semester yang sedang aktif

### Validasi Wajib (Mencegah Kesalahan Sistem)

1. **Tanpa celah (no gap), tanpa tumpang tindih (no overlap) DALAM 1 semester** — seluruh rentang tanggal `A`+`B` yang diinput admin untuk 1 semester **harus 100% menutupi** rentang `semesters.tanggal_mulai`–`tanggal_selesai` semester itu, tanpa ada 1 hari pun yang terlewat atau diklaim 2 rentang sekaligus. Validasi ini dicek **setiap kali admin submit 1 pasangan rentang baru** (real-time, bukan ditunda ke akhir) — tolak submit kalau overlap dengan rentang lain di semester YANG SAMA
2. **Tidak boleh overlap ANTAR semester** — kalau admin sedang menyiapkan Semester Genap sementara Semester Ganjil masih berjalan (diizinkan, lihat bagian "Loopback" di bawah), rentang tanggal yang diinput untuk Genap **tidak boleh** beririsan dengan rentang tanggal manapun yang sudah ada di semester lain manapun (Ganjil atau semester sebelumnya) — validasi ini juga **real-time saat submit**, bukan ditunda ke saat admin "aktifkan" semester
3. **Celah tanggal yang belum lengkap = hard block** — kalau ada tanggal dalam rentang semester yang BELUM ada di `block_week_ranges` manapun (admin belum selesai input semua rentang), maka untuk tanggal itu: **semua sesi "Mulai Mengajar" terkunci total** (dianggap tidak ada jadwal blok yang valid) sampai admin melengkapi rentang untuk tanggal tsb. Ini bukan silent fallback ke suatu default — harus tampil sebagai pesan jelas ke guru ("Jadwal untuk tanggal ini belum lengkap, hubungi Admin Jurnal") dan sebagai warning mencolok di dashboard admin_jurnal

### Semester Genap Bisa Disiapkan Sebelum Ganjil Selesai
- Admin **boleh input** rentang `block_week_ranges` untuk Semester Genap kapan saja, termasuk saat Semester Ganjil masih `is_active: true` dan sedang berjalan — tidak perlu menunggu Ganjil selesai secara waktu nyata
- **Tapi** tanggalnya tetap tidak boleh tumpang tindih secara kalender dengan semester lain (validasi #2 di atas) — "boleh disiapkan lebih awal" berarti keleluasaan OPERASIONAL kerja admin, bukan izin membuat 2 semester mengklaim tanggal yang sama
- Semester Genap yang sudah lengkap rentangnya TETAP tidak aktif (`is_active: false`) dan tidak memengaruhi resolve jadwal harian sampai admin eksplisit `PATCH /semesters/:id/activate`

### Implikasi Skema
- Field `mode` (blok/normal) pindah dari konfigurasi sekolah global (draf awal) ke **kolom di `semesters`** — tiap semester punya mode sendiri
- Tabel baru **`block_week_ranges`** — `id`, `semester_id` (FK), `tanggal_mulai`, `tanggal_selesai`, `minggu` (`A`/`B`, **tidak ada** `setiap_minggu` di level ini — kalau ada jam yang berlaku di kedua minggu, itu diatur di `schedules.minggu = setiap_minggu` seperti sebelumnya, bukan di rentang tanggal)
- `schedules.minggu` (`A`/`B`/`setiap_minggu`) **tidak berubah** — tetap dipakai persis seperti sebelumnya, hanya SUMBER "minggu aktif hari ini" yang berubah dari hitungan otomatis jadi lookup ke `block_week_ranges`

### ⚠️ Trade-off yang Disadari (Bukan Bug)
- **Hard block berlaku per-SEMESTER, bukan per-kelas.** Kalau ada 1 hari yang lubang jadwal blok-nya (belum ditandai A/B), SEMUA kelas terkunci untuk hari itu — termasuk kelas yang jadwalnya sebenarnya `setiap_minggu` (tidak bergantung minggu A/B sama sekali). Ini konsekuensi dari desain "1 resolver global per semester" yang dipilih supaya logic tetap sederhana (1 sumber kebenaran, bukan per-kelas) — diterima sebagai trade-off, bukan diperbaiki jadi granular per-kelas kecuali diminta eksplisit nanti.
- **Race condition pada submit rentang blok ditutup dengan DB transaction+lock** (lihat T054) — untuk 1 admin_jurnal yang bekerja sendirian risikonya kecil, tapi mekanismenya tetap wajib ada sebagai jaring pengaman.
- **Aktivasi semester `mode: blok` dengan lubang jadwal di-hard-block** (T054) — admin harus melengkapi seluruh rentang dulu sebelum semester itu bisa diaktifkan. Ini bisa terasa kaku kalau admin ingin aktivasi lebih awal sambil menyelesaikan sisa minggu di masa depan — tapi dipilih demi mencegah insiden "semua guru terkunci mendadak".
- **`block_week_ranges` tidak bisa diubah untuk tanggal hari ini/masa lalu** (T054) — mencegah sesi yang sudah tergenerate berubah di tengah jalan, tapi berarti kesalahan input untuk HARI INI tidak bisa langsung dikoreksi (harus tunggu besok/berlaku ke depan saja).

### ❓ Belum diputuskan — mode jadwal
- [ ] Kalau admin perlu mengubah 1 rentang yang sudah ada (bukan tambah baru, dan untuk tanggal MASA DEPAN yang masih boleh diubah) — apakah edit in-place (dengan validasi ulang no-gap/no-overlap terhadap rentang lain), atau harus hapus dulu baru buat ulang (keputusan sementara T054: hapus+buat ulang)?
- [ ] Apakah histori perubahan `block_week_ranges` perlu disimpan (audit log), atau cukup state final saat ini?

---

## 📔 Jurnal Mengajar — Field

Per sesi (1 sesi = 1 slot jadwal guru+kelas+mapel+jam pada tanggal tertentu):
- **Materi/topik** — deskripsi bebas apa yang diajarkan
- **Tujuan pembelajaran/capaian** — teks bebas (belum diputuskan apakah terhubung ke daftar CP/TP formal atau bebas total)
- **Tugas/penilaian** — catatan tugas/penilaian yang diberikan sesi itu
- **Catatan/kendala** — field bebas tambahan

Referensi Jurnale: data izin/sakit/alfa otomatis muncul di jurnal dari hasil presensi. Pola serupa dipakai di sini — tapi sumber presensinya dari tap gerbang, bukan tap kelas.

### ❓ Belum diputuskan — jurnal
- [ ] Apakah field "Tujuan Pembelajaran" perlu link ke daftar CP/TP kurikulum resmi (butuh modul kurikulum terpisah), atau teks bebas saja untuk versi awal?
- [ ] Bisakah jurnal diedit setelah sesi auto-close? Sampai kapan batas edit (akhir hari? tidak bisa sama sekali?)

---

## 🔐 Gating & Validasi "Mulai Mengajar"

### 1. Tap Gerbang
- Guru harus sudah tap gerbang hari itu (`attendance_records` guru ada untuk tanggal berjalan) baru tombol "Mulai Mengajar" aktif
- **Tidak** menggate login — guru tetap bisa masuk dashboard, lihat jadwal, riwayat, dll tanpa tap gerbang. Hanya aksi mulai sesi yang terkunci
- Guru yang gagal tap (kartu rusak, dll) → tidak bisa mulai sesi apapun hari itu sampai ditangani petugas kartu/piket (alur existing manajemen kartu, tidak ada jalur baru)

### 2. Jendela Waktu Jadwal
- Slot hanya aktif (bisa diklik) begitu jam mulai jadwal tercapai
- **Toleransi keterlambatan configurable** (bukan langsung dianggap telat di menit ke-0) — konsisten dengan prinsip "fully configurable" di 01-Overview (01-Overview.md)
- Keterlambatan dihitung dari selisih jam mulai jadwal (+ toleransi) ke waktu klik "Mulai Mengajar"
- Sesi **auto-close** otomatis saat jam selesai jadwal tercapai — tidak perlu aksi tutup manual dari guru

### 3. Geofencing Lokasi
- Setiap **kampus** (bukan tiap kelas) punya 1 titik koordinat + radius geofence sendiri — sekolah ini multi-kampus
- Klik "Mulai Mengajar" → browser capture GPS → validasi dalam radius kampus tempat kelas itu berada
- **Hard block, tanpa jalur override**: gagal GPS/di luar radius = tidak bisa mulai sesi, titik. Tidak ada override manual oleh piket/admin dalam desain ini (risiko UX diterima secara sadar — HP dengan akurasi GPS buruk di gedung beton berpotensi memblokir guru yang sebenarnya hadir)

### ❓ Belum diputuskan — gating
- [ ] Radius geofence per kampus — angka pastinya (rekomendasi awal ~100-150m, perlu disesuaikan denah kampus nyata)
- [ ] Guru pengganti/piket yang masuk kelas orang lain di luar jadwal tetapnya — **belum ada jalur** karena sesi "ketat ke jadwal terdaftar" (lihat bagian di bawah)
- [ ] Nilai default toleransi keterlambatan (menit) — dan apakah configurable global atau per-guru/per-mapel

---

## 🙋 Izin Guru Tidak Mengajar

> Pola nyata di lapangan: guru izin **tidak pernah** digantikan guru pengganti fisik. Guru yang izin biasanya tetap menitipkan tugas ke kelas yang ditinggalkan (lewat WA/lisan ke piket/rekan). Alur di bawah mendigitalkan pola ini, bukan menciptakan pola baru (tidak ada mekanisme "assign guru pengganti").

### Kategori Izin (Final — 2026-07-22)
Field `kategori` di `teacher_permits`, murni informatif/pelaporan — **tidak mengubah alur approval atau efek ke `teaching_sessions`**, semua kategori diperlakukan sama secara teknis:
- `sakit`
- `izin_pribadi`
- `tugas_dinas` — nota dinas dari sekolah
- `pelatihan` — surat undangan/penugasan pelatihan

### Alur (2 tahap, admin duluan baru guru)

1. **Guru lapor izin di luar sistem** (WA/lisan ke admin jurnal) — sesuai kebiasaan sekolah, tidak ada form pengajuan digital dari sisi guru di tahap ini
2. **Admin Jurnal input status izin** untuk guru tsb pada tanggal (dan sesi/slot spesifik, atau seharian penuh) tertentu, pilih `kategori`, dan **WAJIB upload file bukti pendukung** (surat izin/nota dinas/surat pelatihan/surat dokter — apapun yang relevan dengan kategori) — begitu diinput lengkap, status guru itu di hari itu jadi **"Diizinkan"**
   - **File bukti WAJIB untuk semua kategori**, tanpa pengecualian. Untuk kasus dadakan (guru sakit mendadak, admin approve cepat tanpa surat dokter di tangan) — admin **boleh upload bukti sementara dulu** (misal screenshot chat WA guru yang lapor), lalu **update/ganti file** begitu dokumen resmi (surat dokter, dst) sudah didapat. Sistem harus mendukung **replace file** kapan saja setelah izin dibuat, bukan cuma sekali upload immutable
   - **Tidak ada status "izin tanpa dokumen"** — kalau admin belum punya file apapun (bahkan bukti WA), izin belum bisa diinput sampai ada sesuatu untuk diupload. Ini keputusan sadar: kelengkapan dokumentasi didahulukan, admin yang mengatur ritmenya sendiri (upload cepat dengan bukti sementara, sempurnakan nanti)
3. **Baru setelah admin approve**, tombol **"Izin"** di slot jadwal guru itu (dashboard guru) menjadi aktif/terbuka — sebelum admin approve, tombol ini terkunci/tidak ada efek. Ini mencegah guru sembarangan klik izin tanpa sepengetahuan admin
4. Guru klik "Izin" → form terbuka: **upload file tugas** + **keterangan pengganti jurnal** (field bebas, mengisi peran "materi/tugas" hari itu meski guru tidak hadir fisik) — **ini upload TERPISAH dari file bukti izin di langkah 2** (bukti izin = alasan guru tidak hadir; tugas titipan = materi pengganti untuk siswa)
5. Kalau guru **tidak sempat** mengisi form tugas (izin dadakan, lupa, dll) — sesi tetap tercatat **"Guru Izin"**, tapi ditandai flag **"Perlu Ditindaklanjuti"** supaya guru piket tahu ada kelas kosong tanpa arahan tugas dan bisa turun tangan (misal titip tugas dari mapel lain, atau sekadar mengawasi ketertiban)

### Implikasi ke sesi & jurnal

- Sesi berstatus izin **tidak dihitung** sebagai "jurnal belum diisi"/bolos mengajar di rekap kedisiplinan guru — beda kategori dari sesi yang memang kosong tanpa keterangan
- Tombol "Mulai Mengajar" dan tombol "Izin" saling eksklusif per slot per hari — begitu satu dipakai, yang lain terkunci untuk slot itu
- Siswa di kelas tsb tetap bisa melihat/mengakses tugas titipan (lewat kanal yang sama dengan jurnal biasa — detail UI menyusul saat breakdown task)

### ❓ Belum diputuskan — izin guru
- [ ] Izin per-slot (guru izin cuma 1 jam pelajaran tertentu) vs izin seharian penuh (semua slot guru itu hari ini) — apakah admin pilih granularitasnya saat input, atau selalu salah satu?
- [ ] Format upload (bukti izin maupun tugas titipan) — jenis file apa saja yang diterima, ada batas ukuran? (asumsi kerja: sama seperti T046 — PDF/JPG/PNG/DOCX, maks 10MB, kecuali diputuskan lain)
- [ ] Siapa yang menyelesaikan/clear flag "Perlu Ditindaklanjuti" — guru piket saja, atau admin jurnal juga bisa?
- [ ] Riwayat izin guru — apakah masuk ke rekap terpisah (mirip rekap perizinan siswa oleh piket), dipakai untuk evaluasi kepegawaian?

---

## 🚫 Batasan yang Disengaja (Scope Ketat)

- **Sesi ketat ke jadwal resmi** — guru hanya bisa mulai sesi untuk slot yang terdaftar sebagai jadwal tetapnya. **Tidak ada entri jurnal bebas/insidental** untuk kasus guru pengganti dadakan mengisi kelas kosong — dan berdasarkan diskusi izin di atas, memang **tidak pernah ada guru pengganti fisik** di sekolah ini, jadi gap ini kemungkinan besar tidak relevan lagi (izin + tugas titipan sudah menutup kasus nyatanya)
- **Status "bolos mapel"** ditandai manual oleh guru per siswa (bukan otomatis dari sistem), terpisah dari status izin/sakit/alfa resmi yang tetap wewenang guru piket

---

## 👤 Role Baru: `admin_jurnal`

> Mengikuti pola existing `card_admin` (ADR-008) — role generik terkunci ke satu domain modul, ditegakkan di level API guard bukan cuma UI, konsisten dengan aturan tegas di 03-User-Roles (03-User-Roles.md) ("kalau cuma disembunyikan di frontend, endpoint API tetap bisa diakses langsung").

**Wewenang `admin_jurnal`** (terkunci murni ke domain jurnal — **tidak** bisa akses `users`, `cards`, `academic_years`/`school_holidays`, atau rekap kehadiran siswa fase 1):
- Kelola jadwal mengajar: assign guru–kelas–mapel–jam, termasuk konfigurasi Mode Blok A/B vs Normal dan titik acuan minggu aktif
- Kelola izin guru: input status "Diizinkan" per guru/tanggal/slot (alur di atas)
- Monitor **dan koreksi** semua jurnal & status sesi guru (lebih dari read-only `kepsek`/`super_admin` — bisa perbaiki kalau ada kesalahan input guru)
- Kelola master data mapel/kurikulum (daftar mata pelajaran; CP/TP formal kalau nanti dibuat — lihat open question jurnal di atas)

**Yang TETAP jadi wewenang `super_admin`** (tidak berpindah): kelola akun `users` (termasuk membuat akun `admin_jurnal` itu sendiri), kartu RFID, kalender pendidikan, rekap kehadiran siswa/gerbang.

### ❓ Belum diputuskan — admin_jurnal
- [ ] Assign jadwal butuh data guru yang valid — apakah `admin_jurnal` minimal bisa *lihat/pilih* dari daftar guru existing (read-only ke `teachers`), atau benar-benar nol akses ke modul guru/users?
- [ ] Siapa yang membuat akun `admin_jurnal` pertama kali — tetap `super_admin` seperti pola `card_admin`?

---

## 👀 Hak Akses Lihat Jurnal

| Role | Akses |
|---|---|
| Guru bersangkutan | Full akses jurnal & presensi miliknya sendiri |
| `admin_jurnal` | Full akses + koreksi seluruh jurnal, jadwal, dan izin guru (lihat di atas) |
| `super_admin` | Read-only semua jurnal (sesuai pola akses admin existing) |
| `kepsek` | Read-only semua jurnal semua guru |
| `guru_piket` | Lihat **status sesi hari itu** (siapa sudah/belum mulai mengajar, siapa terlambat, siapa izin+flag "Perlu Ditindaklanjuti") — untuk pantauan real-time, bukan isi jurnal lengkap |
| Wali Kelas (`guru` + `kelas_id_wali` terisi, lihat bagian di bawah) | Read-only, scope kelas yang diampu — **v1: rekap kehadiran + catatan/kendala saja** (bukan materi/tujuan/tugas), lihat detail di bawah |

---

## 📚 Semester & Loopback Tahun Ajaran (Final — 2026-07-22)

> Detail skema semester ada di kalender-pendidikan.md (06-Features/kalender-pendidikan.md) bagian "1b. Semester" — baca itu dulu untuk struktur tabel `semesters`. Bagian ini fokus ke IMPLIKASI ke Jadwal Mengajar & Rekap yang sudah dispec sebelumnya di dokumen ini.

### Jadwal Mengajar sekarang di-scope per semester
- `schedules` (`type: jam_mengajar`) **wajib** punya `semester_id` — ini **mengubah** T047/T050 yang sebelumnya hanya scope per tahun ajaran implisit. Jadwal semester ganjil dan genap adalah **set data terpisah sepenuhnya**, tidak otomatis lanjut dari semester sebelumnya
- **Fitur "Salin Jadwal dari Semester Sebelumnya"**: saat admin_jurnal mulai kelola jadwal semester baru (belum ada 1 pun `schedules` untuk `semester_id` itu), tampilkan tombol/opsi ini di halaman Jadwal Mengajar. Klik → duplikasi SEMUA `schedules` (`type: jam_mengajar`) dari semester aktif sebelumnya ke `semester_id` baru (termasuk `mapel_id`, `minggu`, hari, jam) → admin tinggal edit/hapus/tambah yang berubah
- **Semester aktif**: 1 semester aktif per waktu (`semesters.is_active`), di-switch manual oleh admin_jurnal (bukan otomatis dari tanggal) — job T040 (generate `teaching_sessions` harian) HARUS resolve dari semester aktif ini untuk tahu `schedules` mana yang berlaku hari ini, bukan lagi query semua `schedules` guru tanpa filter semester

### Loopback ke Tahun Ajaran/Semester Sebelumnya
- **Rekap Kehadiran** (rekap-kehadiran.md (06-Features/rekap-kehadiran.md)): tambah filter dropdown "Tahun Ajaran" (dan opsional "Semester") — data historis SUDAH tidak terhapus by design (lihat `kalender-pendidikan.md` poin 1), yang kurang cuma UI untuk memilihnya secara eksplisit. Default filter: tahun ajaran + semester aktif saat ini
- **Jadwal Mengajar** (Dashboard Admin Jurnal, T050): tambah dropdown "Semester" di halaman jadwal — admin_jurnal bisa lihat/edit jadwal semester manapun (termasuk semester yang belum aktif, untuk persiapan), tapi **generate `teaching_sessions` harian (T040) HANYA pernah baca dari semester yang `is_active: true`** — melihat/menyiapkan jadwal semester lain tidak memengaruhi operasional harian sampai semester itu benar-benar diaktifkan
- **Ini juga berlaku untuk `block_week_ranges`** (lihat [[#📅 Jadwal: Mode Blok (Minggu A/B) vs Mode Normal]]) — admin boleh input rentang minggu A/B untuk semester genap kapan saja meski ganjil masih berjalan, TAPI validasi no-overlap-antar-semester tetap berlaku real-time. "Boleh disiapkan lebih awal" = keleluasaan kerja, bukan izin tanggal tumpang tindih

### ❓ Belum diputuskan — semester
- [ ] Histori perubahan semester aktif — perlu audit log terpisah atau cukup `activity_log` existing?
- [ ] Kalau admin_jurnal salin jadwal lalu semester lama masih `is_active`, apakah ada validasi/peringatan supaya tidak salah aktifkan semester baru sebelum jadwalnya siap?

---

## 🏫 Wali Kelas (Final — Fase 2, siap eksekusi)

> **Keputusan (2026-07-21):** Wali Kelas **bukan role baru** — extend pola `guru_piket` (akun `guru` existing + kolom scope tambahan), bukan entitas terpisah. Read-only murni, tidak ada wewenang tulis apapun (konsisten "Aturan Tegas" di 03-User-Roles (03-User-Roles.md): guru read-only mutlak).

### Assignment (menu baru di Dashboard Admin Jurnal)
- **1 kelas = 1 wali kelas.** Bukan banyak-ke-banyak — kalau kebutuhan co-wali muncul nanti, itu perubahan skema terpisah di luar scope ini.
- `admin_jurnal` assign lewat menu baru "Wali Kelas" (mirip pola assign jadwal T050): pilih kelas dari dropdown → pilih guru dari daftar existing (read-only lookup ke `teachers`, sama seperti assign jadwal mengajar) → simpan
- **Implikasi skema:** tambah kolom `kelas_id_wali` (FK ke `kelas`, nullable) di `users` — pola identik `guru_piket.kampus_id`. Nullable karena tidak semua guru jadi wali kelas. Constraint unique di `kelas_id_wali` (1 kelas cuma bisa dipegang 1 user wali kelas aktif sekaligus) ditegakkan di service layer (MySQL/Prisma tidak native support unique constraint pada nullable FK dengan mudah — cek pendekatan saat implementasi, boleh partial unique index kalau MySQL versi yang dipakai mendukung, atau validasi service).
- Begitu `kelas_id_wali` terisi pada akun guru, sidebar dashboard guru (`/guru`) otomatis menampilkan menu baru **"Wali Kelas"** — tidak perlu login terpisah, akun yang sama (guru bisa punya jadwal mengajar SEKALIGUS jadi wali kelas 1 kelas).

### Isi Menu "Wali Kelas" (v1 — final, siap eksekusi)

**1. Ringkasan Kehadiran Kelas** (halaman utama menu)
- Angka ringkas hari ini: jumlah hadir / izin / sakit / alfa untuk kelas yang diampu
- Tabel per siswa dengan filter rentang tanggal (default bulan berjalan): Nama | Hadir | Terlambat | Izin | Sakit | Alfa | % Kehadiran
- **Reuse logic yang sama persis dengan Rekap Admin** (rekap-kehadiran.md (06-Features/rekap-kehadiran.md)), di-scope 1 `kelas_id` dari `req.user.kelas_id_wali` (bukan dari query param, sama seperti pola scope `guru_piket.kampus_id`) — **tidak** membangun ulang logic hitung alfa, panggil service Rekap yang sudah ada dengan filter kelas terkunci

**2. Rekap Per Mapel (dari sesi kelas)**
- Breakdown per siswa: berapa kali ditandai `tidak_ada_di_kelas` (dari `class_attendance_marks`) per mapel — beda dari alfa gerbang, ini sinyal "hadir sekolah tapi bolos mapel tertentu"
- Daftar sesi kelas berstatus "Guru Izin" untuk kelasnya (dari `teacher_permits`) — transparansi kapan & mapel apa yang kosong karena guru izin, termasuk status tugas titipan (sudah/belum diisi)

**3. Catatan/Kendala dari Guru Mapel**
- List kronologis per tanggal: mapel, guru, isi field **`catatan`** SAJA dari `journal_entries` (bukan `materi`/`tujuan_pembelajaran`/`tugas_penilaian`) — field ini yang paling relevan untuk wali kelas (kejadian/kendala di kelas), field akademik murni sengaja belum diekspos di v1

### 📌 Dicatat untuk masa depan (TIDAK masuk scope v1, JANGAN dikerjakan sekarang)
- **Full akses ke seluruh jurnal semua mapel** (materi, tujuan pembelajaran, tugas/penilaian) — sudah diminta user sebagai antisipasi kebutuhan mendatang, tapi *scope v1 sengaja dibatasi ke kehadiran + catatan saja*. Kalau nanti dibutuhkan, buka akses read ke `journal_entries` penuh untuk `kelas_id_wali` yang cocok — **tidak perlu desain ulang**, cukup extend query di halaman "Catatan/Kendala" (poin 3 di atas) untuk include `materi`/`tujuan_pembelajaran`/`tugas_penilaian`, dan ubah tampilan dari "list catatan" jadi "list jurnal lengkap per sesi". Endpoint backend sebaiknya SUDAH dirancang mengembalikan field lengkap dari awal (query `journal_entries` tanpa exclude kolom), filtering field mana yang ditampilkan cukup dilakukan di response DTO/frontend — supaya "buka penuh" nanti tinggal ubah 1 baris filter, bukan bongkar query.

---

## 🗄️ Implikasi Skema (Awal, Belum Final)

Berdasarkan `schedules` yang sudah generic di 04-Database-Schema (04-Database-Schema.md) (ADR-005 mengantisipasi ini dari fase 1):

- `schedules` tambah kolom `minggu` (enum: `A` / `B` / `setiap_minggu`, nullable — dipakai hanya kalau mode blok aktif)
- `schedules` (`type: jam_mengajar`) tambah kolom **`semester_id`** (FK ke `semesters`, wajib) — lihat [[#📚 Semester & Loopback Tahun Ajaran (Final — 2026-07-22)]] dan kalender-pendidikan.md (06-Features/kalender-pendidikan.md) untuk skema tabel `semesters`
- Entitas baru **`semesters`** — didefinisikan lengkap di kalender-pendidikan.md (06-Features/kalender-pendidikan.md) (bukan diulang di sini, satu sumber kebenaran)
- Entitas baru untuk **konfigurasi mode jadwal sekolah**: mode aktif (`blok`/`normal`), titik acuan minggu A/B (tanggal + minggu apa), kemungkinan histori override manual per minggu
- Entitas baru **`teaching_sessions`** (atau reuse `attendance_sessions` yang sudah generic `location_type: kelas`) — perlu field tambahan: `lokasi_lat`, `lokasi_lng` (capture saat mulai), `started_at`, `closed_at` (auto), `status` (`open`/`closed`)
- Entitas baru **`journal_entries`** — `session_id` (FK), `teacher_id`, `materi`, `tujuan_pembelajaran`, `tugas_penilaian`, `catatan`, `created_at`, `updated_at`
- Entitas baru **`class_attendance_marks`** (atau nama lain) — per siswa per sesi kelas, default hadir (dari tap gerbang), guru override manual jadi "tidak ada di kelas" — `session_id`, `student_id`, `status` (`hadir`/`tidak_ada_di_kelas`), `marked_by`, `marked_at`
- **`kampus`** tambah kolom `lokasi_lat`, `lokasi_lng`, `radius_geofence_meter`
- **`users`** tambah kolom **`kelas_id_wali`** (FK ke `kelas`, nullable) — pola Wali Kelas final, lihat bagian [[#🏫 Wali Kelas (Final — Fase 2, siap eksekusi)]]
- `users.role` tambah nilai enum **`admin_jurnal`** (pola sama seperti `card_admin`)
- Entitas baru **`teacher_permits`** (nama sementara, hindari tabrakan dengan `permits` siswa milik piket) — `id`, `teacher_id` (FK), `tanggal`, `session_id` (FK ke sesi/slot jadwal, nullable — null berarti izin seharian penuh, lihat open question granularitas), `kategori` (enum: `sakit`/`izin_pribadi`/`tugas_dinas`/`pelatihan` — murni informatif, lihat bagian "Kategori Izin"), `status` (`diizinkan` — set oleh admin_jurnal), `bukti_file_path` (**wajib diisi**, path relatif file bukti izin — surat/nota dinas/dokumentasi lain, bisa di-replace kapan saja oleh admin_jurnal), `bukti_updated_at` (timestamp terakhir file bukti diganti), `approved_by` (FK ke `users`, role `admin_jurnal`), `approved_at`, `tugas_file_path` (nullable, diisi guru — **berbeda dari `bukti_file_path`**, ini untuk tugas titipan siswa bukan bukti izin), `tugas_keterangan` (nullable, teks, diisi guru), `submitted_at` (nullable, kapan guru isi form), `follow_up_needed` (boolean, true kalau lewat waktu sesi tanpa tugas terisi — dasar flag "Perlu Ditindaklanjuti" untuk piket)

> Semua di atas **draft kasar**, bukan skema final — perlu direview ulang saat masuk breakdown task.

---

## ❓ Open Questions — Ringkasan Semua yang Belum Diputuskan

- [ ] Titik acuan awal Minggu A/B (tanggal, dan siapa/bagaimana override-nya)
- [ ] Radius geofence per kampus (angka pasti)
- [ ] Toleransi keterlambatan default (menit), scope configurable-nya
- [ ] Field "Tujuan Pembelajaran" bebas teks vs terhubung ke CP/TP kurikulum formal
- [ ] Batas waktu edit jurnal setelah sesi auto-close
- [ ] Histori/audit perubahan mode jadwal (blok↔normal)
- [ ] Izin per-slot vs izin seharian penuh — granularitas input admin_jurnal
- [ ] Format & batas ukuran upload file (bukti izin maupun tugas titipan) — asumsi kerja PDF/JPG/PNG/DOCX maks 10MB sampai diputuskan lain
- [ ] Siapa clear flag "Perlu Ditindaklanjuti" — piket saja atau admin_jurnal juga
- [ ] Riwayat izin guru masuk rekap terpisah untuk evaluasi kepegawaian atau tidak
- [ ] `admin_jurnal` perlu akses read-only ke daftar guru (`teachers`) untuk keperluan assign jadwal, atau benar-benar nol akses modul users
- [ ] Siapa yang membuat akun `admin_jurnal` pertama — asumsi `super_admin` seperti pola `card_admin`, perlu dikonfirmasi
- [ ] Histori perubahan semester aktif — audit terpisah atau cukup `activity_log`?
- [ ] Validasi/peringatan saat aktifkan semester baru yang jadwalnya belum lengkap disalin/diedit

---

## 🔗 Lihat Juga
- absensi-kelas-mapel.md (06-Features/absensi-kelas-mapel.md) — rencana LAMA yang digantikan dokumen ini (tap RFID kelas, dipertahankan sebagai arsip alasan)
- absensi-gerbang.md (06-Features/absensi-gerbang.md) — sumber data tap gerbang yang jadi basis presensi mapel
- dashboard-piket.md (06-Features/dashboard-piket.md) — pola akses scoped by `kampus_id`, referensi untuk desain scope Wali Kelas
- 03-User-Roles (03-User-Roles.md) — role Wali Kelas masih catatan terbuka
- 04-Database-Schema (04-Database-Schema.md) — `schedules`, `attendance_sessions` generic (ADR-005)
- 13-Backlog (13-Backlog.md)
- Referensi pembanding: [Jurnale PRO — Panduan](https://pro.jurnale.id/panduan/?p=pendahuluan)
