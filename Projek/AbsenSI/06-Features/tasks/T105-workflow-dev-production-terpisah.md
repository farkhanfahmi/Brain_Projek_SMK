# T105 — Infra: Pisahkan Environment Development dari Production (1 Mesin, Menuju Migrasi ke Server Sekolah)

## Depends on
Tidak ada — murni infrastruktur (docker-compose, folder, git branch), TIDAK mengubah kode aplikasi.

## Objective
Menutup celah yang menyebabkan [[06-Features/tasks/T104-akun-section-berbasis-role|insiden database wipe 2026-07-30]]: testing/development TIDAK LAGI berbagi database DAN checkout kode yang sama dengan yang dipakai live sekolah. Disiapkan supaya nanti mudah "naik pangkat" saat server sekolah sungguhan siap — production tinggal `git clone` branch `main` + restore backup, development di komputer ini lanjut sebagai tempat eksperimen permanen.

## Context
- **Diskusi 2026-07-31**, lanjutan pasca-insiden [[feedback_insiden_database_wipe_2026-07-30]] dan pemasangan [[project_backup_absensi_terpasang|backup otomatis]].
- **Kondisi topologi saat ini** (dikonfirmasi user): komputer ini SEMENTARA berperan sebagai production (diakses sekolah via `10.10.10.198`, sampai server sekolah sungguhan siap). Rencana jangka panjang: pindah ke server sekolah sebagai production permanen, komputer ini kembali murni jadi mesin development.
- **Temuan penting soal Git** (WAJIB ditindaklanjuti, bagian dari task ini): repo HANYA punya 1 branch (`main`), commit TERAKHIR tanggal **2026-07-23** (`48c6668`), sementara **136 file berubah/baru** belum pernah di-commit — artinya SELURUH pekerjaan T072-T104 (migrasi CSV siswa, sinkronisasi kelas/kampus, fitur ekstrakurikuler lengkap, dashboard piket batch 3, dst) tidak ada jejaknya di GitHub. Kalau mesin ini rusak, kerugian akan sama fatalnya dengan insiden database wipe — kali ini kode, bukan data.
- **Temuan infra existing**: `docker-compose.yml` di root cuma 2 service (`mysql:8` port 3307→3306, `redis:7` port 6379) — sederhana, mudah diduplikasi untuk dev. Container database production bernama `absensi-mysql-1` (lihat [[project_backup_absensi_terpasang]]).

## Keputusan Final (dikonfirmasi user 2026-07-31)

1. **Tetap 1 mesin untuk sekarang** (bukan 2 server terpisah) — dipisahkan lewat kombinasi database berbeda + folder checkout berbeda + branch git berbeda, semuanya berjalan BERSAMAAN di mesin yang sama tanpa saling ganggu.
2. **2 database terpisah**:
   - `absensi-mysql-1` (existing, port 3307) → **TETAP PRODUCTION**, JANGAN PERNAH di-reset/diutak-atik untuk testing.
   - Container BARU (misal `absensi-mysql-dev`, port `3308`) → **DEVELOPMENT**, boleh di-`migrate reset`/seed/rusak sebebasnya.
3. **2 folder checkout terpisah** (BUKAN 1 folder gonta-ganti branch) — supaya production tidak pernah ke-checkout ke branch lain secara tidak sengaja, dan supaya keduanya bisa jalan bersamaan:
   - `/home/anunnaki/Documents/APP SMK/AbsenSI` (existing) → **PRODUCTION**, branch `main`, port app SAMA seperti sekarang (web 3000, api 3001, kiosk 3002), `.env` menunjuk `absensi-mysql-1`.
   - `/home/anunnaki/Documents/APP SMK/AbsenSI-dev` (BARU, `git clone` dari repo yang sama) → **DEVELOPMENT**, branch `dev`, port BERBEDA (usulan: web 3100, api 3101, kiosk 3102 — cek dulu tidak bentrok port lain yang sudah dipakai container Docker lain di mesin ini, lihat `docker ps` untuk daftar lengkap), `.env` menunjuk `absensi-mysql-dev`.
4. **Alur kerja Git**: `dev` (atau feature branch dari `dev`) untuk semua eksperimen → setelah teruji, merge `dev` → `main` → push ke GitHub → **folder production** `git pull origin main` + `prisma migrate deploy` (BUKAN `migrate reset`) + restart service.
5. **Migrasi ke server sekolah nanti**: server sekolah `git clone` branch `main` dari GitHub, restore dump terbaru dari `/media/anunnaki/DataNvme/backups/absensi/` (lihat [[project_backup_absensi_terpasang]]), set `.env` production baru. Komputer ini otomatis jadi development permanen (folder production di sini bisa dihentikan/dihapus setelah migrasi sukses, TIDAK sebelum ada konfirmasi server baru berjalan stabil).

## Spec Detail — Langkah Implementasi

### 1. Selamatkan histori kode SEKARANG (paling mendesak, kerjakan PERTAMA sebelum langkah lain)
- `git add` semua perubahan yang relevan di folder production existing (review dulu `git status`, JANGAN `git add -A` membabi buta — ada kemungkinan file sensitif/besar yang tidak seharusnya di-commit, cek `.gitignore` sudah benar untuk `node_modules`, `.next`, `storage/photos`, dll).
- Commit dengan pesan jelas per kelompok fitur (atau 1 commit besar dengan pesan yang menjelaskan rentang T072-T104 kalau user memilih itu — **tanya user preferensinya saat eksekusi**, jangan asumsikan granularitas commit).
- `git push origin main` — pastikan SETELAH ini GitHub benar-benar punya salinan kerja terkini, verifikasi dengan `git log origin/main` cocok dengan HEAD lokal.
- **PENTING**: baru setelah histori kode aman di GitHub, lanjut ke langkah berikutnya — jangan bikin folder dev dulu sebelum kode existing benar-benar tersimpan.

### 2. Buat branch `dev`
- `git checkout -b dev` di folder production (existing), lalu `git push -u origin dev` — sekarang GitHub punya `main` (stabil) dan `dev` (aktif dikembangkan).
- Folder production KEMBALI ke `main` (`git checkout main`) — folder ini seterusnya HANYA boleh di `main`.

### 3. Buat database development terpisah
- Duplikasi service MySQL di `docker-compose.yml` (atau file compose terpisah `docker-compose.dev.yml`) — container baru nama BEDA (`absensi-mysql-dev`), port host BEDA (`3308:3306`), volume Docker BEDA (`mysql_dev_data`, JANGAN share volume dengan `mysql_data` production — itu akan membuat 2 container menunjuk data yang sama, bukan terpisah).
- `docker compose -f docker-compose.dev.yml up -d` untuk start database dev.

### 4. Clone folder development
- `git clone git@github.com:farkhanfahmi/AbsenSI.git "AbsenSI-dev"` (di `/home/anunnaki/Documents/APP SMK/`), `git checkout dev` di folder baru ini.
- `.env` folder dev: copy dari production lalu ubah `DATABASE_URL` ke port `3308`, ubah port app (`PORT` di api, `--port` di package.json script web/kiosk kalau perlu, atau pakai env var kalau Next.js mendukung) — **cek satu-satu file `.env`-relevant, JANGAN asumsikan semua variabel port bisa di-override tanpa ubah kode** (lihat `apps/web/package.json` script `dev`/`start` yang hardcode `--port 3000`/`--port 3002` — perlu diubah jadi port dev di package.json folder INI SAJA, tidak memengaruhi production karena folder terpisah).
- `pnpm install`, `prisma migrate dev` (folder dev BOLEH pakai `migrate dev`/`reset` sebebasnya karena ini database development).

### 5. Dokumentasi workflow
- Tulis `WORKFLOW.md` atau tambahkan section baru di `CLAUDE.md`/`CONTEXT.md` project — jelaskan 2 folder ini, kapan pakai yang mana, alur `dev → main → deploy`, dan PERINGATAN EKSPLISIT "JANGAN PERNAH jalankan migrate reset/seed di folder production — kalau ragu folder mana yang sedang aktif, cek `pwd` dan `git branch` dulu sebelum operasi destruktif apapun".

## Business Rules
- Folder production **TIDAK PERNAH** di-checkout ke branch selain `main`.
- Folder dev **TIDAK PERNAH** dipakai untuk akses langsung dari jaringan sekolah (`10.10.10.198`) — kalau perlu demo ke user/staff sekolah, pakai folder production.
- Container `absensi-mysql-1` (production) TIDAK PERNAH menerima perintah `migrate reset`/`db push --force-reset`/`TRUNCATE` sembarangan — HANYA `migrate deploy` (apply migration yang sudah final, tidak destruktif terhadap data existing kalau migration ditulis benar).
- Backup otomatis ([[project_backup_absensi_terpasang]]) TETAP berjalan HANYA untuk `absensi-mysql-1` (production) — database dev TIDAK perlu di-backup (isinya memang boleh hilang kapan saja).

## Edge Cases
- Kedua folder jalan bersamaan di mesin yang sama — pastikan TIDAK ADA bentrok port (app maupun database) sebelum start keduanya sekaligus. Cek `docker ps`/`ss -tlnp` sebelum menetapkan port dev final.
- BullMQ/Redis — `docker-compose.yml` production pakai Redis port 6379. Kalau folder dev juga butuh Redis sendiri (BullMQ job scheduler dsb), perlu instance Redis TERPISAH juga (port beda), atau pakai Redis DB index berbeda (`REDIS_URL` dengan `/1` alih-alih `/0`) kalau ingin share 1 instance Redis dengan namespace terpisah — **putuskan pendekatan mana saat implementasi**, keduanya valid.
- Migrasi ke server sekolah nanti — pastikan proses ini didokumentasikan sebagai task/checklist TERPISAH saat waktunya tiba (jangan asumsikan langkah-langkah di atas otomatis mencakup migrasi fisik ke mesin baru sepenuhnya, ini task infra yang beda scope).

## Files
- **Buat:** `docker-compose.dev.yml` (atau service tambahan di compose existing), folder `AbsenSI-dev/` (git clone terpisah, di luar scope "file" karena bukan bagian repo yang sama — clone kedua dari remote yang sama).
- **Modifikasi:** `.gitignore` (review dulu, pastikan `node_modules`, `.next`, `apps/api/storage/photos`, `.env` sudah masuk sebelum commit besar pertama), `CLAUDE.md`/`CONTEXT.md` (dokumentasi workflow baru).
- **Jangan sentuh:** kode aplikasi (`apps/*/src`) — task ini murni infrastruktur/proses, tidak ada perubahan logic.

## Acceptance Criteria
- [ ] Seluruh kerja T072-T104 (136 file yang saat ini uncommitted) sudah ter-commit dan ter-push ke GitHub branch `main`.
- [ ] Branch `dev` dibuat dan ter-push.
- [ ] Database dev (`absensi-mysql-dev`, port 3308) berjalan terpisah dari production, migration bisa dijalankan bebas tanpa memengaruhi production.
- [ ] Folder `AbsenSI-dev/` berjalan dengan port berbeda dari production, tidak bentrok.
- [ ] Dokumentasi workflow tertulis jelas, termasuk peringatan eksplisit soal operasi destruktif.
- [ ] Folder production tetap bisa diakses normal oleh jaringan sekolah selama proses ini (tidak ada downtime yang tidak direncanakan).

## Validasi Claudian
- [ ] LANGKAH 1 (selamatkan kode ke GitHub) WAJIB dikerjakan PALING AWAL dan TERPISAH dari langkah lain — ini yang paling mendesak, jangan tunda demi menyiapkan folder dev dulu.
- [ ] Konfirmasi ke user granularitas commit (1 commit besar vs dipecah per fitur) sebelum commit — jangan asumsikan.
- [ ] Cek ulang `.gitignore` SEBELUM commit besar pertama — pastikan tidak ada file besar/sensitif (foto siswa di `storage/photos`, `.env` dengan kredensial) ikut ter-commit tanpa sengaja.
- [ ] Setelah setup selesai, JELASKAN ke user cara memverifikasi folder mana yang sedang aktif dikerjakan (`pwd`, `git branch`, cek `.env` DATABASE_URL) — supaya insiden "salah kira folder/database" tidak terulang dalam bentuk lain.
