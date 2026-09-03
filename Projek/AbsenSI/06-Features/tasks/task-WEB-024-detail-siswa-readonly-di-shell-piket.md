# Task-WEB-024: Halaman Detail Siswa Read-Only di Dalam Shell Piket (Direktori Murid Tidak Keluar dari Shell)

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah audit kejanggalan Dashboard Piket + diskusi kritis dengan user (2026-09-03). Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.

**Task Terbuat:** 2026-09-03
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Bukan fitur baru dari nol (data & tampilan detail siswa SUDAH ADA di `(admin)/siswa/[id]/`), tapi butuh ketelitian memisahkan route baru di dalam `(piket)/` yang read-only total tanpa duplikasi logic besar-besaran — perlu keputusan struktur file yang rapi (REUSE komponen presentasi, bukan REUSE route admin).

## 2. Konteks & Tujuan Utama

Audit Dashboard Piket (2026-09-03) menemukan: klik baris murid di **Direktori Siswa** (`apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx` baris ~207, `router.push(`/siswa/${student.id}`)`) membawa guru_piket ke halaman `/siswa/[id]` yang dirender oleh **`(admin)/layout.tsx`** — BUKAN `(piket)/layout.tsx`. Di sana `PiketSidebar` memang sudah dipasang manual (peninggalan T076), TAPI `PiketNotificationsProvider` dan `PiketDutyProvider` **TIDAK ikut ter-mount** — akibatnya begitu piket masuk ke halaman detail siswa: ikon lonceng notifikasi hilang, badge count hilang, dan proteksi "tidak bertugas hari ini" (`usePiketOnDuty()`) tidak berlaku sama sekali di halaman itu (kalau ada aksi tulis apa pun yang seharusnya di-gate, gate itu hilang).

**Keputusan yang sudah disepakati user**: Opsi (a) — buat route detail siswa BARU khusus di dalam `(piket)/` yang read-only total, BUKAN mencoba membuat `(admin)/siswa/[id]/layout.tsx` mendukung mount context piket (itu menambah coupling antar route group yang tidak diinginkan).

**Depends on:** Tidak ada — tapi WAJIB baca dulu struktur `(admin)/siswa/[id]/` existing untuk tahu apa saja yang perlu direplikasi/di-reuse.

## 3. Langkah Eksekusi Detail

1. **Riset dulu** — baca struktur `apps/web/src/app/(admin)/siswa/[id]/` (page.tsx + komponen detail: foto, biodata, riwayat terlambat/izin/sakit/alfa — sesuai deskripsi di `dashboard-piket.md` §6b: *"Klik siswa → buka halaman detail siswa yang SAMA dipakai admin... read-only untuk piket, tombol edit/upload/hapus disembunyikan untuk role ini"*). Identifikasi komponen presentasi murni (render biodata/riwayat) yang BISA di-reuse tanpa membawa logic edit/hapus — kemungkinan sudah ada pemisahan komponen tampilan vs komponen form edit, VERIFIKASI SAAT IMPLEMENTASI seberapa mudah dipisah.

2. **Buat route baru** `apps/web/src/app/(piket)/piket/siswa/[id]/page.tsx` (di dalam shell piket, `[id]` dynamic segment) — fetch data siswa yang sama (endpoint detail siswa existing, cek apakah endpoint sudah cukup generic untuk dipakai role `guru_piket` atau perlu penyesuaian scope/permission di backend, VERIFIKASI SAAT IMPLEMENTASI, KEMUNGKINAN endpoint sudah dipakai admin dan bisa langsung di-reuse kalau guard role-nya sudah mengizinkan `guru_piket` baca).

3. **REUSE komponen presentasi** (foto, biodata, riwayat terlambat/izin/sakit/alfa) dari `(admin)/siswa/[id]/` — import langsung komponennya (bukan copy-paste isi), TAPI pastikan komponen itu SUDAH menerima prop untuk menyembunyikan tombol edit/upload/hapus (kalau belum ada prop seperti itu, tambahkan prop opsional `readOnly`/`hideActions` ke komponen SHARED tersebut — VERIFIKASI SAAT IMPLEMENTASI apakah komponen ini dipakai HANYA oleh admin atau sudah dipakai di beberapa tempat, supaya perubahan prop tidak breaking untuk konsumen lain).

4. **Ubah `direktori-siswa-view.tsx`** baris ~207 — `router.push(`/siswa/${student.id}`)` → `router.push(`/piket/siswa/${student.id}`)` (arahkan ke route baru di dalam shell piket).

5. **Verifikasi shell tetap utuh** — pastikan halaman baru ini otomatis mewarisi `PiketNotificationsProvider`+`PiketDutyProvider`+`PiketSidebar` dari `(piket)/layout.tsx` (karena berada di dalam route group yang sama, seharusnya otomatis — VERIFIKASI dengan cek manual/dev server bahwa lonceng notifikasi & sidebar tetap muncul di halaman detail siswa baru ini).

6. **Cek breadcrumb/tombol kembali** — pastikan ada cara jelas kembali ke Direktori Siswa dari halaman detail (konsisten UX, REPLIKASI pola tombol back yang mungkin sudah ada di halaman admin).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Buat:** `apps/web/src/app/(piket)/piket/siswa/[id]/page.tsx` (dan komponen view pendamping kalau perlu, mis. `detail-siswa-piket-view.tsx`)
- **Modifikasi:** `apps/web/src/app/(piket)/piket/siswa/direktori-siswa-view.tsx` — ubah target navigasi
- **Modifikasi (kemungkinan, VERIFIKASI SAAT IMPLEMENTASI):** komponen presentasi shared di `(admin)/siswa/[id]/` — tambah prop `readOnly`/`hideActions` KALAU belum ada, supaya bisa dipakai ulang dari route piket baru tanpa menampilkan tombol edit/hapus.
- **Jangan sentuh:** `(admin)/siswa/[id]/page.tsx` sendiri (route admin TETAP seperti semula, tidak diubah perilakunya untuk admin), logic edit/upload/hapus siswa (tidak disentuh sama sekali, hanya disembunyikan lewat prop untuk konteks piket).

**Dilarang dilakukan:**
- Jangan copy-paste isi komponen presentasi jadi duplikat baru — WAJIB reuse import langsung, supaya kalau nanti field biodata berubah, tidak perlu update 2 tempat (pelajaran proyek: drift antar file yang mestinya sama).
- Jangan biarkan tombol edit/upload/hapus tetap ADA di DOM tapi cuma `disabled` — WAJIB benar-benar disembunyikan (`hidden`/tidak dirender sama sekali), sesuai prinsip existing Direktori Siswa yang SUDAH benar melakukan ini (konsisten, bukan celah baru).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: guru_piket akses langsung URL `/piket/siswa/[id]` untuk siswa DI LUAR kampusnya (lewat Direktori Siswa yang MEMANG sengaja lintas-kampus, lihat `dashboard-piket.md` §6b) → Perilaku benar: TETAP BISA diakses (ini pengecualian sadar existing untuk Direktori Siswa, BUKAN bug baru) — TAPI pastikan endpoint detail siswa yang dipanggil tidak ikut membocorkan kemampuan EDIT lintas kampus (read-only tetap berlaku, hanya baca yang lintas kampus).
- Kondisi: siswa dengan `id` tidak valid/tidak ditemukan → pesan error jelas ("Siswa tidak ditemukan"), bukan crash/500 generik.
- Kondisi: komponen shared yang di-reuse ternyata sudah dipakai di tempat lain SELAIN `(admin)/siswa/[id]/` (mis. Wali Kelas) → pastikan penambahan prop `readOnly` TIDAK mengubah behavior default konsumen existing (default value prop = `false`/tampilkan semua tombol seperti sebelumnya).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Klik siswa di Direktori Siswa membawa piket ke `/piket/siswa/[id]` (di dalam shell piket), BUKAN `/siswa/[id]` (admin).
- [ ] Lonceng notifikasi, badge count, dan sidebar piket TETAP tampil normal di halaman detail siswa baru ini.
- [ ] Tombol edit/upload/hapus TIDAK dirender sama sekali (bukan sekadar disabled) di halaman ini.
- [ ] Data biodata + riwayat terlambat/izin/sakit/alfa tampil identik dengan versi admin (reuse komponen, bukan reimplementasi).
- [ ] `(admin)/siswa/[id]/` tetap berfungsi normal untuk role admin (tidak ada regresi).
- [ ] Build + typecheck bersih.

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau komponen shared ternyata sulit dipisah tanpa refactor besar, LAPORKAN dan diskusikan ulang sebelum lanjut, jangan paksa reuse yang berisiko merusak halaman admin)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (konsisten prinsip read-only Direktori Siswa existing)
- [ ] Dependency: tidak ada
