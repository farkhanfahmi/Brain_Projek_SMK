# Task-CORE-006 / WEB-006: Deteksi Proaktif + Pencegahan Opsi Jadwal Aktif Tanpa Induk Aktif

> Modul prefix: CORE (apps/api) / WEB (apps/web) / KIOSK (apps/kiosk).
> Ditulis oleh Hermes (sesi Planning) setelah diskusi kritis dengan user. Dieksekusi oleh Claude Code — user yang memicu jalannya, BUKAN Hermes.
> Task ini gabungkan backend (deteksi + validasi) dan frontend (notifikasi) — dipertahankan 1 file karena keduanya saling terkait erat.

**Task Terbuat:** 2026-09-02
**Task Tereksekusi:** —

---

## 1. Info Eksekusi

**Rekomendasi Model:** Sonnet
**Tingkat Effort:** medium
**Alasan pemilihan:** Bagian A (deteksi/notifikasi) murni query baru + tampilan, low-medium. Bagian B (validasi saat nonaktifkan semester/tahun ajaran) menyentuh flow AcademicYear/Semester yang sudah established — perlu ketelitian supaya tidak mengganggu operasi normal (nonaktifkan semester adalah operasi rutin, bukan destruktif, jangan bikin terlalu ribet).

## 2. Konteks & Tujuan Utama

Audit menu Jadwal Pelajaran (sesi diskusi 2026-09-02): warning banner "Opsi Jadwal aktif tapi induk (Semester/Tahun Ajaran) tidak aktif" (`jadwal-pelajaran-view.tsx` baris 256-270) sudah ADA dan bekerja dengan baik — TAPI hanya muncul saat admin membuka halaman Jadwal Pelajaran untuk **semester yang sedang dipilih di dropdown**. Kalau kondisi "aktif tapi tidak nyata berlaku" ini terjadi di semester LAIN yang sedang tidak dilihat admin, tidak ada yang memberi tahu — guru bisa bingung kenapa jadwalnya kosong tanpa admin sadar penyebabnya.

**Keputusan user: implementasikan KEDUA lapis (A + B):**
- **A (Deteksi proaktif)** — tampilkan kondisi ini di tempat yang admin PASTI lihat tanpa perlu buka semester spesifik (Dashboard Admin), lintas semua semester.
- **B (Pencegahan di akar)** — saat admin menonaktifkan Semester/AcademicYear yang MASIH punya Opsi Jadwal aktif di bawahnya, tampilkan konfirmasi eksplisit dulu, bukan diam-diam membiarkan kondisi anomali tercipta.

**Depends on:** Tidak ada.

## 3. Langkah Eksekusi Detail

### Bagian A — Deteksi Proaktif (Backend + Dashboard)

1. **Cari/buat endpoint deteksi lintas-semester** — kemungkinan tambah method baru di `OpsiJadwalService` atau service dashboard yang sudah ada (`apps/api/src/dashboard/` — cek dulu apakah ada modul dashboard sentral untuk admin, atau ini perlu endpoint baru di `opsi-jadwal.controller.ts`):
   ```ts
   async findAnomaliIndukTidakAktif() {
     return this.prisma.opsiJadwal.findMany({
       where: {
         isActive: true,
         OR: [
           { semester: { isActive: false } },
           { semester: { academicYear: { isActive: false } } },
         ],
       },
       include: { semester: { include: { academicYear: true } } },
     });
   }
   ```
   Return data secukupnya untuk pesan actionable (nama Opsi Jadwal, nama semester, nama tahun ajaran, penyebab spesifik — SAMA formatnya dengan warning existing di `jadwal-pelajaran-view.tsx` yang SUDAH actionable, REUSE logic penyebab yang sama, jangan duplikasi logic beda).

2. **Endpoint** `GET /opsi-jadwal/anomali-induk-tidak-aktif` (atau lokasi lain yang lebih konsisten dengan struktur dashboard existing — cek dulu) — role `super_admin` + `admin_jurnal` (konsisten akses domain Jurnal Mengajar).

3. **Tampilkan di Dashboard admin** — cari halaman dashboard utama admin (`apps/web/src/app/(admin)/page.tsx` atau serupa, ATAU sidebar notifikasi kalau proyek ini punya pola notifikasi terpusat — cek dulu apakah ada komponen notifikasi/badge count di shell admin) — tampilkan sebagai card/banner peringatan kalau `findAnomaliIndukTidakAktif()` return non-kosong, dengan link langsung ke halaman Jadwal Pelajaran semester terkait (`/jadwal-pelajaran` dengan semester itu ter-pre-select, atau minimal sebutkan nama semesternya supaya admin tahu ke mana harus pergi).

### Bagian B — Pencegahan di Akar (Konfirmasi saat Nonaktifkan)

4. **Cari alur nonaktifkan Semester** — cek `SemestersService` (sudah dibaca Hermes: `activate()` ada, tapi TIDAK ADA method `deactivate()` eksplisit — semester "nonaktif" terjadi SECARA IMPLISIT saat semester LAIN di-`activate()` — lihat `activate()` baris 111-114: `updateMany({ isActive: false })` lalu `update` yang baru jadi true). **Ini mengubah pendekatan** — tidak ada 1 titik "nonaktifkan semester X" yang eksplisit, yang ada adalah "aktifkan semester Y (otomatis menonaktifkan semua yang lain)".

5. **Sesuaikan validasi di `SemestersService.activate()`** — SEBELUM melakukan `updateMany({ isActive: false })` yang menonaktifkan semua semester lain, cek dulu apakah ADA semester lain (yang AKAN dinonaktifkan oleh aksi ini) yang MASIH punya Opsi Jadwal `isActive: true` di bawahnya. Kalau ada:
   - **JANGAN otomatis block** — operasi "ganti semester aktif" adalah operasi RUTIN (2x setahun), terlalu mengganggu kalau di-hard-block tiap kali.
   - **Kembalikan info tersebut ke response** (bukan exception) — mis. `{ semester: {...}, opsiJadwalTerdampak: [...] }` — biarkan FRONTEND yang menampilkan dialog konfirmasi ("Semester Ganjil masih punya 2 Opsi Jadwal aktif, tetap lanjut aktifkan Semester Genap?") SEBELUM benar-benar submit. Ini pola **konfirmasi 2-langkah** (preview dulu, baru commit), bukan hard validation di backend.
   - Tambahkan endpoint GET terpisah untuk preview dampak (`GET /semesters/:id/preview-aktivasi` atau serupa) yang dipanggil FE SEBELUM tombol "Aktifkan" ditekan sungguhan, ATAU sertakan info ini di response `activate()` yang BARU DIJALANKAN SETELAH konfirmasi user (2 kali panggil: 1x dry-run untuk cek, 1x commit sungguhan) — **tentukan pendekatan mana yang lebih simpel diimplementasikan tanpa mengubah terlalu banyak kontrak existing**, laporkan pilihan desainnya di ringkasan hasil task.

6. **Frontend** (`apps/web/src/app/(admin)/semester/` atau lokasi tombol aktivasi semester — cari dulu lokasinya) — sebelum submit `activate()`, panggil dulu endpoint preview, kalau ada Opsi Jadwal terdampak, tampilkan `Dialog` konfirmasi menyebutkan SECARA SPESIFIK Opsi Jadwal mana yang akan jadi "aktif tapi tidak nyata berlaku" — biarkan admin PILIH: lanjut aktifkan (terima kondisinya, akan tetap muncul di warning banner + dashboard anomali dari Bagian A) ATAU batal dulu (pergi nonaktifkan Opsi Jadwal itu manual duluan).

## 4. Batasan & Penanganan Kasus Khusus

**Files:**
- **Modifikasi:** `apps/api/src/opsi-jadwal/opsi-jadwal.service.ts` — tambah `findAnomaliIndukTidakAktif()`
- **Modifikasi:** `apps/api/src/opsi-jadwal/opsi-jadwal.controller.ts` — endpoint baru
- **Modifikasi:** `apps/api/src/semesters/semesters.service.ts` — `activate()` disesuaikan (tambah info dampak, TIDAK mengubah behavior inti "1 semester aktif lintas semua")
- **Modifikasi:** halaman dashboard admin (lokasi tepatnya dicari saat eksekusi) — tampilkan card anomali
- **Modifikasi:** halaman/komponen aktivasi semester — tambah dialog konfirmasi dampak
- **Jangan sentuh:** logic inti `activate()` yang menegakkan "1 semester aktif lintas semua semester" (`updateMany({isActive: false})` lalu `update`) — itu SUDAH BENAR (dikonfirmasi audit sebelumnya), task ini HANYA menambah lapisan info/konfirmasi, BUKAN mengubah constraint itu.

**Dilarang dilakukan:**
- Jangan bikin aktivasi semester jadi hard-blocked oleh Opsi Jadwal aktif lain — itu operasi rutin, terlalu mengganggu kalau diblokir total. Solusinya adalah INFORMASI + KONFIRMASI, bukan larangan.
- Jangan duplikasi logic "kenapa Opsi Jadwal ini anomali" — REUSE logic yang sama persis dengan warning banner existing di `jadwal-pelajaran-view.tsx` (kalau logic-nya di FE, pertimbangkan pindah ke backend supaya bisa dipakai both by dashboard DAN halaman existing tanpa duplikasi).

**Skenario kegagalan yang WAJIB ditangani:**
- Kondisi: admin mengaktifkan semester baru TANPA sempat lihat dialog konfirmasi (mis. race condition 2 tab) → kondisi anomali TETAP akan terdeteksi oleh Bagian A (dashboard) meski Bagian B (konfirmasi) terlewat — 2 lapis ini saling menutupi, bukan saling bergantung.
- Kondisi: TIDAK ADA Opsi Jadwal anomali sama sekali (kasus normal, mayoritas waktu) → dashboard TIDAK menampilkan card/banner apa pun (jangan tampilkan card kosong "tidak ada masalah", cukup tidak render sama sekali).
- Kondisi: admin_jurnal (bukan super_admin) mengakses dashboard anomali → pastikan RBAC endpoint ini konsisten dengan domain "Jurnal Mengajar" yang memang wewenang admin_jurnal (BEDA dari temuan task-WEB-001 soal Jam Masuk Sekolah yang BUKAN domain admin_jurnal — Opsi Jadwal/Jadwal Pelajaran MEMANG domain admin_jurnal, jadi akses ini benar diberikan).

**Edge case:**
- Semester yang mau diaktifkan justru semester YANG SAMA dengan yang sudah aktif (no-op, `activate()` sudah handle ini di baris `if (before.isActive) return before;`) — tidak perlu tampilkan dialog konfirmasi untuk kasus ini (tidak ada perubahan state sama sekali).

## 5. Kriteria Selesai

**Acceptance Criteria:**
- [ ] Endpoint backend mengembalikan daftar Opsi Jadwal aktif dengan induk (Semester/TahunAjaran) tidak aktif, lintas SEMUA semester (bukan cuma yang sedang dipilih)
- [ ] Dashboard admin menampilkan card/banner anomali ini kalau ada, dengan link ke halaman terkait
- [ ] Aktivasi semester baru menampilkan dialog konfirmasi kalau akan menciptakan kondisi anomali (Opsi Jadwal semester lama jadi "aktif tapi tidak berlaku")
- [ ] Admin tetap BISA melanjutkan aktivasi meski ada dampak (bukan hard-block) — hanya perlu konfirmasi eksplisit
- [ ] Constraint inti "1 semester aktif lintas semua semester" TIDAK berubah

**Validasi sebelum dianggap selesai:**
- [ ] Tidak ada ambiguitas dalam spec ini (dicek ulang oleh Hermes sebelum handoff) — KECUALI pilihan desain preview-dampak (dry-run vs GET terpisah) yang eksplisit didiskusikan saat eksekusi
- [ ] Semua skenario kegagalan di bagian 4 sudah tercakup implementasinya
- [ ] Scope tidak terlalu besar (estimasi < 300 baris perubahan; kalau Bagian A dan B ternyata besar, pertimbangkan pecah jadi 2 task terpisah saat eksekusi — task-CORE-006a untuk deteksi, task-CORE-006b untuk konfirmasi aktivasi)
- [ ] Tidak ada konflik dengan keputusan arsitektur yang sudah ada (constraint semester aktif tunggal dipertahankan)
- [ ] Dependency (jika ada) sudah selesai sebelum task ini di-assign — tidak ada dependency
